# 📋 Data Card & Model Card — CyberGuard AI

> Version 2.1.0 | Last updated: 2026-03-17

---

## 📊 Data Card — HRDataset

### Overview

| Field | Value |
|---|---|
| **Dataset Name** | IBM HR Analytics Employee Attrition & Performance |
| **File** | `HRDataset.csv` |
| **Rows (Employees)** | ~1,470 |
| **Columns (Features)** | 35 |
| **Target Variable** | `Attrition` (Yes / No) |
| **Class Balance** | ~16% Attrition (Yes), ~84% No Attrition |
| **Source** | IBM / Kaggle public dataset (fictional, for research) |
| **License** | Open / Creative Commons (Kaggle) |
| **Language** | English (column headers) |
| **Encoding** | UTF-8 with BOM (read with `utf-8-sig`) |

---

### Feature Description

#### Numeric Features (used in model)

| Feature | Description | Range / Unit |
|---|---|---|
| `Age` | Employee age | 18–60 years |
| `DailyRate` | Daily pay rate | $ |
| `DistanceFromHome` | Distance to office | km |
| `Education` | Education level | 1 (Below College) – 5 (Doctor) |
| `EnvironmentSatisfaction` | Workplace satisfaction | 1 (Low) – 4 (Very High) |
| `HourlyRate` | Hourly rate | $ |
| `JobInvolvement` | Level of job involvement | 1–4 |
| `JobLevel` | Seniority level | 1–5 |
| `JobSatisfaction` | Job satisfaction | 1 (Low) – 4 (Very High) |
| `MonthlyIncome` | Monthly salary | $ |
| `MonthlyRate` | Monthly rate | $ |
| `NumCompaniesWorked` | Previous employers | 0–9 |
| `PercentSalaryHike` | Salary hike % last year | % |
| `PerformanceRating` | Last performance review | 1–4 |
| `RelationshipSatisfaction` | Relationship with manager | 1–4 |
| `StockOptionLevel` | Stock option grant level | 0–3 |
| `TotalWorkingYears` | Total career years | years |
| `TrainingTimesLastYear` | Training sessions last year | count |
| `WorkLifeBalance` | Work-life balance rating | 1–4 |
| `YearsAtCompany` | Tenure at current company | years |
| `YearsInCurrentRole` | Years in current role | years |
| `YearsSinceLastPromotion` | Years since last promotion | years |
| `YearsWithCurrManager` | Years with current manager | years |

#### Categorical Features (used in model)

| Feature | Description | Values |
|---|---|---|
| `BusinessTravel` | Travel frequency | Non-Travel, Travel_Rarely, Travel_Frequently |
| `Department` | Employee department | Sales, R&D, HR |
| `EducationField` | Field of study | Life Sciences, Medical, Marketing, Technical Degree, Human Resources, Other |
| `Gender` | Gender | Male, Female |
| `JobRole` | Job title | 9 roles (Manager, Researcher, Sales Exec, etc.) |
| `MaritalStatus` | Marital status | Single, Married, Divorced |
| `OverTime` | Overtime worked | Yes, No |

---

### Data Quality

| Concern | Status | Mitigation |
|---|---|---|
| Missing values | Rare | Numeric: median imputation; Categorical: fill 'Unknown' |
| Class imbalance | ~16% attrition | `class_weight='balanced'` in RandomForest |
| Encoding issues | BOM present | Read with `utf-8-sig` |
| Personal identifiers | `EmployeeNumber` present | Anonymized in all API responses as `Emp_{ID}` |
| Sensitive attributes | `Gender`, `Age` present | Not directly used as decision features; monitored for bias |

---

### Data Limitations

- Dataset is **fictional** (IBM synthetic data), not from a real company
- No temporal dimension — reflects a single snapshot, not longitudinal trends
- `Gender` is binary — does not represent full gender spectrum
- Cultural context is **US-centric** — may not generalize to other regions
- No free-text fields — qualitative reasons for departure are absent
- Dataset is static — **must be retrained** if used with real organizational data

---

### GDPR Considerations

| Principle | Status |
|---|---|
| Data Minimization | ✅ Only HR-relevant features used; no biometric/surveillance data |
| Purpose Limitation | ✅ Used exclusively for attrition risk insight, not hiring/firing decisions |
| Transparency | ✅ Employees informed of AI system existence (required before production use) |
| Right to Access | ✅ GDPR audit endpoint `/api/hr/audit-rgpd/{emp_id}` |
| Right to Erasure | ⚠️ To implement: employee data deletion pipeline |
| Data Retention | ✅ Audit logs retention 2 years minimum |

---

---

## 🤖 Model Card — Random Forest Classifier

### Model Overview

| Field | Value |
|---|---|
| **Model Name** | CyberGuard HR Attrition Predictor |
| **Model Type** | Random Forest Classifier |
| **Task** | Binary Classification — Predict employee attrition (Yes/No) |
| **Output** | Probability score ∈ [0, 1] → risk level (Low / Moderate / High / Critical) |
| **Library** | scikit-learn 1.4.2 |
| **Version** | v1.0.0 |
| **Training Date** | 2026-03-16 |

---

### Model Configuration

```python
RandomForestClassifier(
    n_estimators = 300,    # 300 decision trees
    max_depth    = 14,     # Limit tree depth to reduce overfitting
    class_weight = 'balanced',  # Compensate for class imbalance (16% attrition)
    random_state = 42      # Reproducibility
)
```

---

### Training Data

| Split | Description |
|---|---|
| Training set | Full HRDataset.csv (~1,470 employees) |
| Validation | Not separately held out (hackathon scope) — production use requires train/test split |

> ⚠️ **Production note**: A proper train/test split (e.g. 80/20) with cross-validation is required before deploying on real data.

---

### Input Features

The model takes **30 raw features** → expands to ~50+ columns after one-hot encoding.

Top contributing features (by Gini importance, approximate order):
1. `OverTime` — Strongest predictor; employees working overtime have significantly higher attrition
2. `MonthlyIncome` — Lower income correlates with higher departure probability
3. `YearsAtCompany` — Shorter tenure → higher risk
4. `Age` — Younger employees more likely to leave
5. `JobSatisfaction` — Low satisfaction → elevated risk
6. `TotalWorkingYears` — More experience → lower attrition
7. `YearsWithCurrManager` — Short tenure with manager → risk signal
8. `DistanceFromHome` — Longer commute → risk factor

---

### Model Performance

> Evaluated on full training set (for hackathon purposes). **Must be re-evaluated on held-out test set for production.**

| Metric | Value (indicative) |
|---|---|
| Accuracy | ~87% (imbalanced — misleading alone) |
| Recall (Attrition=Yes) | ~75% (with balanced class weights) |
| Precision (Attrition=Yes) | ~60% (trade-off favoring recall) |
| AUC-ROC | ~0.82 |

Class weights are balanced, biasing the model to **prefer recall over precision** for the minority class — better for HR use cases (missing a departure is more costly than a false positive).

---

### Explainability

| Method | Description |
|---|---|
| **Gini Feature Importance** | Global importance ranking across all trees |
| **Per-prediction factors** | Top 8 features driving each individual prediction |
| **SHAP** | Library included (`shap==0.45.0`), available for deeper local explanations |
| **Counterfactuals** | Shows minimum changes needed to flip prediction (via binary search) |
| **Decision boundaries** | 4 risk thresholds with explicit human-readable justifications |

---

### Fairness & Bias Assessment

| Protected Attribute | Monitoring Method | Status |
|---|---|---|
| Gender | 4/5 rule (disparate impact) | ⚠️ Requires external audit |
| Age | Statistical parity test | ⚠️ Requires external audit |
| Department | Attrition rate comparison | ✅ Available via `/api/advanced/bias-audit` |
| Marital Status | Prediction rate analysis | ⚠️ Requires external audit |

**Mitigation measures applied:**
- No directly discriminatory features used for final decision (no explicit race/origin input)
- `class_weight='balanced'` avoids over-predicting the majority group
- Bias audit endpoint built-in, using the **4/5 rule** from US employment law (EEO)

---

### Known Limitations

| Limitation | Impact | Recommendation |
|---|---|---|
| Trained on fictional data | Cannot directly deploy | Retrain on real organizational data |
| No temporal modeling | Ignores seasonal patterns | Use time-series features in production |
| Static model | Degrades with data drift | Monitor and retrain if drift > 5% |
| Binary attrition label | No resignation reason | Supplement with exit interview data |
| Gini importance biased toward high-cardinality features | Some features may appear artificially important | Use SHAP for more reliable importance |

---

### Intended Use

✅ **Appropriate uses:**
- Support HR managers in identifying at-risk employees
- Prioritize retention conversations
- Test impact of hypothetical HR policies (simulation)
- Audit AI fairness and compliance

❌ **Prohibited uses:**
- Automated firing or hiring decisions
- Performance reviews based solely on this score
- Sharing individual risk scores with employees without proper process
- Deployment without human oversight

---

### EU AI Act Classification

| Item | Value |
|---|---|
| **Risk Category** | **HIGH-RISK** (Annex III — Employment decisions) |
| **Requirements** | Transparency, Explainability, Human Oversight, Bias Monitoring, Audit Trail |
| **Compliance Status** | Partially compliant (hackathon scope) — production requires full conformity assessment |
| **Human Oversight** | ✅ Mandatory: no decision executed without human validation |
| **Right to Explanation** | ✅ Available via `/api/hr/ia-act/{emp_id}` |
| **Right to Contest** | ✅ Documented in audit endpoint |

---

### Model Versioning & Monitoring

| Practice | Status |
|---|---|
| Model artifact persistence | ✅ Docker volume `model-artifacts` |
| Drift monitoring | ⚠️ To implement: trigger retraining if distribution drift > 5% |
| Retraining schedule | Recommended: quarterly or on significant HR data updates |
| Version tracking | `model_info.version` returned in every API response |
