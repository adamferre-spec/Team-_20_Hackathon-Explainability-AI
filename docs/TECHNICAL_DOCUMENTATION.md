# 📘 Technical Documentation — CyberGuard AI

> Version 2.1.0 | Last updated: 2026-03-17

---

## 1. Overview

CyberGuard AI is a full-stack web application for **HR attrition prediction** powered by a Random Forest classifier with integrated Explainable AI (XAI) features. The system is built as a **microservices architecture** containerized with Docker, exposing a REST API (FastAPI) consumed by a React frontend.

---

## 2. Tech Stack

| Layer | Technology | Version |
|---|---|---|
| Backend Framework | FastAPI | 0.111.0 |
| ASGI Server | Uvicorn | 0.29.0 |
| Data Validation | Pydantic | 2.7.0 |
| Machine Learning | scikit-learn | 1.4.2 |
| Explainability | SHAP | 0.45.0 |
| Data Processing | pandas / numpy | 2.2.2 / 1.26.4 |
| Imbalanced Data | imbalanced-learn | 0.12.3 |
| Frontend | React + Vite | — |
| Containerization | Docker + Compose | — |
| Language | Python 3.11+ / JavaScript | — |

---

## 3. Backend Architecture

### 3.1 Entry Point — `main.py`

The FastAPI application initializes with:
- **CORS Middleware** — allows calls from `localhost:3000` and `localhost:5173`
- **SecurityHeadersMiddleware** — injects HTTP security headers on every response:
  - `X-Content-Type-Options: nosniff`
  - `X-Frame-Options: DENY`
  - `X-XSS-Protection: 1; mode=block`
  - `Strict-Transport-Security: max-age=31536000`

### 3.2 Routers

#### `routers/hr.py` — Core HR Prediction

Handles all core ML prediction logic. Key responsibilities:

- **Data Loading** (`_load_data`): Reads `HRDataset.csv` with UTF-8-BOM encoding, strips column whitespace
- **Feature Engineering**:
  - 23 numeric features (Age, DailyRate, MonthlyIncome, etc.)
  - 7 categorical features (Department, JobRole, OverTime, etc.) → one-hot encoded
- **Model Training** (`_get_model`): Lazy-loaded `RandomForestClassifier` (n_estimators=300, max_depth=14, class_weight='balanced')
- **Feature Importance Aggregation**: Sums encoded dummy importances back to original feature names
- **Audit Logging** (`_create_audit_log`): Creates immutable JSON audit entry per prediction
- **Input Sanitization** (`_sanitize_input`): Validates integer bounds (0–100,000)

**Endpoints:**

| Method | Path | Description |
|---|---|---|
| GET | `/api/hr/stats` | Global attrition statistics by department |
| GET | `/api/hr/employees` | Employee list with risk scores (anonymized) |
| GET | `/api/hr/predict/{emp_id}` | Full prediction + explainability for one employee |
| GET | `/api/hr/model-explainability` | Global model explainability report |
| POST | `/api/hr/simulate` | Predict for a hypothetical employee profile |
| GET | `/api/hr/correlation-matrix` | Feature-attrition correlation matrix |
| GET | `/api/hr/termination-reasons` | Departure reason analysis |
| GET | `/api/hr/audit-rgpd/{emp_id}` | GDPR compliance check |
| GET | `/api/hr/ia-act/{emp_id}` | EU AI Act full compliance audit |

#### `routers/advanced.py` — Advanced Analytics

Advanced XAI features:

| Endpoint | Feature |
|---|---|
| `/api/advanced/counterfactual/{emp_id}` | Counterfactual explanations (what changes flip the prediction) |
| `/api/advanced/bias-audit` | Fairness audit using the 4/5 disparate impact rule |
| `/api/advanced/anomaly/{emp_id}` | Isolation Forest + Z-score anomaly detection |
| `/api/advanced/simulate` | Chained what-if intervention simulation |
| `/api/advanced/trends` | Department attrition trends |

#### `routers/alerts.py` — Cybersecurity Alerts
Manages security alerts and breach notifications for the HR system.

#### `routers/audit.py` — Audit Trail
Exposes audit log endpoints for compliance review.

#### `routers/predict.py` — Generic Prediction Utility
General-purpose prediction endpoint (model-agnostic layer).

---

## 4. Machine Learning Pipeline

### 4.1 Data Preprocessing

```
Raw CSV (HRDataset.csv)
    ↓
Load with pandas (utf-8-sig encoding)
    ↓
Select 30 features (23 numeric + 7 categorical)
    ↓
Numeric: fillna(median), coerce to float
Categorical: fillna('Unknown'), one-hot encode
    ↓
Reindex to fixed encoded column list
    ↓
Output: Feature matrix X
```

### 4.2 Target Variable

`Attrition` (binary): `'Yes'` → 1, `'No'` → 0

### 4.3 Model

```python
RandomForestClassifier(
    n_estimators=300,
    max_depth=14,
    class_weight='balanced',  # handles class imbalance (~16% attrition)
    random_state=42
)
```

### 4.4 Explainability Methods

| Method | Usage |
|---|---|
| **Gini Feature Importance** | Global model explanation (all employees) |
| **Per-employee top factors** | Local explanation (individual prediction) |
| **SHAP** (library available) | Deep local explanation (available via advanced router) |
| **Counterfactuals** | Binary search on feature space to find minimal changes |
| **Isolation Forest** | Anomaly/outlier detection in employee profiles |

### 4.5 Risk Thresholds

| Level | Probability Range | Action |
|---|---|---|
| 🔴 Critical | ≥ 0.70 | Immediate HR intervention |
| 🟠 High | 0.50 – 0.69 | Targeted retention actions |
| 🟡 Moderate | 0.30 – 0.49 | Regular monitoring |
| 🟢 Low | < 0.30 | No immediate action |

---

## 5. Frontend Architecture

Built with **React + Vite**, structured as a SPA:

```
frontend/src/
├── pages/
│   ├── HRAttrition.jsx          ← Main dashboard (employee list, risk scores)
│   └── AdvancedDashboard.jsx    ← Advanced analytics page
└── components/
    ├── CounterfactualPanel.jsx  ← Shows actionable changes
    ├── BiasAuditPanel.jsx       ← Fairness metrics visualization
    ├── AnomalyPanel.jsx         ← Outlier profiles & early warnings
    ├── SimulationPanel.jsx      ← What-if testing UI
    └── TrendsPanel.jsx          ← Department trends charts
```

API calls are proxied via Vite config (`VITE_API_URL=/api`) to the backend at port 8000.

---

## 6. Infrastructure & Deployment

### 6.1 Docker Compose Services

| Service | Container | Port | Description |
|---|---|---|---|
| `backend` | `cyberguard-api` | 8000 | FastAPI + Uvicorn |
| `frontend` | `cyberguard-ui` | 3000 | React + Vite dev server |

### 6.2 Volumes

| Volume | Purpose |
|---|---|
| `model-artifacts` | Persists trained model files |
| `processed-data` | Caches preprocessed data |

### 6.3 Health Check

```bash
GET /health
→ { "status": "ok", "version": "2.0.0", "security": "Cybersecurity & RGPD Compliance Active" }
```

Docker health check polls this endpoint every 10s with a 5s timeout, 3 retries.

---

## 7. Security Measures

| Measure | Implementation |
|---|---|
| HTTP Security Headers | Custom `SecurityHeadersMiddleware` in `main.py` |
| Input Validation | `_sanitize_input()` — bounds check on all integer inputs |
| CORS | Restricted to known frontend origins |
| Data Anonymization | Employee names replaced by `Emp_{ID}` |
| Audit Logging | Every prediction logged with timestamp, employee_id, risk_score |
| Container Isolation | Docker network isolation between services |

---

## 8. API Reference

Full interactive documentation available at:
- **Swagger UI**: http://localhost:8000/docs
- **ReDoc**: http://localhost:8000/redoc

### Example: Prediction Response

```json
{
  "emp_id": 42,
  "name": "Emp_42",
  "risk_score": 0.731,
  "risk_level": "Critique",
  "probable_reason": "Insatisfaction",
  "recommendation": "Action immédiate: Retenir l'employé - Intervention RH urgente",
  "risk_factors": [
    { "feature": "OverTime", "importance": 0.142, "value": "Yes" },
    { "feature": "MonthlyIncome", "importance": 0.118, "value": 3500 }
  ],
  "model_info": {
    "type": "Random Forest Classifier",
    "n_estimators": 150,
    "max_depth": 14
  }
}
```

---

## 9. Environment Variables

| Variable | Service | Description |
|---|---|---|
| `PYTHONUNBUFFERED` | backend | Enable real-time logs |
| `VITE_API_URL` | frontend | API base URL (default: `/api`) |
