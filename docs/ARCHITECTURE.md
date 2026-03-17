# 🏗️ Architecture — CyberGuard AI

> Version 2.1.0

---

## System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                          USER BROWSER                               │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    FRONTEND (React + Vite)                   │   │
│  │                     container: cyberguard-ui                 │   │
│  │                         PORT: 3000                           │   │
│  │                                                              │   │
│  │  ┌──────────────────┐    ┌────────────────────────────────┐ │   │
│  │  │  HRAttrition.jsx │    │    AdvancedDashboard.jsx        │ │   │
│  │  │  - Employee list │    │  - CounterfactualPanel          │ │   │
│  │  │  - Risk scores   │    │  - BiasAuditPanel               │ │   │
│  │  │  - Predictions   │    │  - AnomalyPanel                 │ │   │
│  │  └──────────────────┘    │  - SimulationPanel              │ │   │
│  │                          │  - TrendsPanel                  │ │   │
│  │                          └────────────────────────────────┘ │   │
│  └─────────────────────┬───────────────────────────────────────┘   │
│                         │  HTTP REST (JSON) via VITE_API_URL=/api   │
└─────────────────────────┼───────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   BACKEND (FastAPI + Uvicorn)                        │
│                   container: cyberguard-api                          │
│                       PORT: 8000                                     │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                      main.py                                  │  │
│  │  ┌──────────────────┐  ┌──────────────────────────────────┐  │  │
│  │  │ CORS Middleware  │  │  SecurityHeadersMiddleware        │  │  │
│  │  │ (allow frontend) │  │  (HSTS, X-Frame, XSS, nosniff)   │  │  │
│  │  └──────────────────┘  └──────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                        ROUTERS                                │ │
│  │                                                               │ │
│  │  /api/hr/*          /api/advanced/*    /api/alerts/*          │ │
│  │  ─────────────────  ────────────────  ─────────────          │ │
│  │  hr.py              advanced.py        alerts.py             │ │
│  │  - stats            - counterfactual   - security            │ │
│  │  - employees        - bias-audit         alerts              │ │
│  │  - predict/{id}     - anomaly/{id}                           │ │
│  │  - simulate         - simulate         /api/audit/*          │ │
│  │  - explainability   - trends           ─────────────         │ │
│  │  - correlation      ────────────────   audit.py              │ │
│  │  - audit-rgpd                          - audit trail         │ │
│  │  - ia-act/{id}      /api/predict/*                           │ │
│  │  ─────────────────  ────────────────                         │ │
│  │                     predict.py                               │ │
│  └────────────────────────────┬──────────────────────────────── ┘ │
│                                │                                    │
│  ┌─────────────────────────────▼──────────────────────────────────┐ │
│  │                    ML PIPELINE                                  │ │
│  │                                                                 │ │
│  │  HRDataset.csv ──► Feature Engineering ──► RandomForest        │ │
│  │                    (one-hot encoding,      (n=300, depth=14    │ │
│  │                     median imputation)      balanced)           │ │
│  │                                                  │              │ │
│  │  ┌─────────────────┐   ┌────────────────┐   ┌───▼──────────┐  │ │
│  │  │ Feature         │   │ Counterfactual │   │ Predictions  │  │ │
│  │  │ Importance      │   │ Generator      │   │ + Risk Level │  │ │
│  │  │ (Gini)          │   │ (binary search)│   │              │  │ │
│  │  └─────────────────┘   └────────────────┘   └──────────────┘  │ │
│  │                                                                 │ │
│  │  ┌─────────────────┐   ┌────────────────┐                      │ │
│  │  │ Bias Audit      │   │ Anomaly        │                      │ │
│  │  │ (4/5 rule)      │   │ Detection      │                      │ │
│  │  │                 │   │ (IsolationFrst)│                      │ │
│  │  └─────────────────┘   └────────────────┘                      │ │
│  └─────────────────────────────────────────────────────────────── ┘ │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                       STORAGE (Docker Volumes)                │  │
│  │  model-artifacts/        processed-data/     HRDataset.csv   │  │
│  │  (serialized models)     (cached features)   (training data) │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Data Flow

### Prediction Flow

```
User selects employee in UI
        │
        ▼
GET /api/hr/predict/{emp_id}
        │
        ▼
_sanitize_input(emp_id)         ← Security: bounds check
        │
        ▼
_load_data()                    ← Lazy-load HRDataset.csv
        │
        ▼
_get_model()                    ← Lazy-train RandomForest (once)
        │
        ▼
_prepare_model_frame(row)       ← Encode + impute features
        │
        ▼
model.predict_proba(X)          ← Get P(attrition)
        │
        ▼
_get_top_feature_impacts(row)   ← Extract top 8 risk factors
        │
        ▼
_create_audit_log(emp_id, ...)  ← Write immutable audit entry
        │
        ▼
JSON Response → Frontend        ← Risk score + explanation
```

### Simulation Flow

```
HR Manager inputs changes in SimulationPanel
        │
        ▼
POST /api/advanced/simulate
{ "emp_id": 42, "interventions": [{"type": "salary", "amount": 10}] }
        │
        ▼
Apply intervention to employee features
        │
        ▼
Re-run model.predict_proba(X_modified)
        │
        ▼
Return delta: original_risk vs new_risk + feature breakdown
```

---

## Component Interaction Map

```
┌──────────────┐     REST API      ┌──────────────────────┐
│   Frontend   │ ◄───────────────► │   Backend (FastAPI)   │
│  React/Vite  │                   │                       │
│   :3000      │                   │  Routers:             │
└──────────────┘                   │  - hr.py              │
                                   │  - advanced.py        │
                                   │  - alerts.py          │
                                   │  - audit.py           │
                                   │  - predict.py         │
                                   │        │              │
                                   │        ▼              │
                                   │  ML Pipeline          │
                                   │  RandomForest         │
                                   │  + SHAP               │
                                   │  + IsolationForest    │
                                   │        │              │
                                   │        ▼              │
                                   │  HRDataset.csv        │
                                   │  (1470 employees)     │
                                   └──────────────────────┘
```

---

## Deployment Architecture

```
Host Machine
│
├── docker-compose.yml
│
├── cyberguard-api (backend container)
│   ├── Image: python:3.11-slim
│   ├── Port: 8000
│   ├── Volume mount: ./backend → /app
│   ├── Volume mount: ./HRDataset.csv → /app/HRDataset.csv
│   ├── Volume: model-artifacts → /app/model/artifacts
│   └── Health check: GET /health every 10s
│
├── cyberguard-ui (frontend container)
│   ├── Image: node:18-alpine
│   ├── Port: 3000
│   ├── Volume mount: ./frontend → /app
│   └── depends_on: backend
│
└── Docker Volumes
    ├── model-artifacts  (persisted ML artifacts)
    └── processed-data   (cached preprocessed features)
```

---

## Security Architecture

```
Incoming Request
      │
      ▼
[ CORS Middleware ]         ← Whitelist allowed origins
      │
      ▼
[ SecurityHeadersMiddleware ] ← Add HSTS, X-Frame, XSS headers
      │
      ▼
[ Router Handler ]
      │
      ├── _sanitize_input()  ← Validate all user-supplied IDs
      │
      ├── Processing...
      │
      └── _create_audit_log() ← Log every prediction to stdout/SIEM
```
