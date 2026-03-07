# NovaCred — Credit Application Governance Analysis

> **DEGO 2606 · Data Ecosystems and Governance in Organizations**  
> MSc Business Analytics · Nova SBE  
> Group Project — Credit Application Governance Analysis

---

## Executive Summary

NovaCred is a fintech company under regulatory scrutiny for potential discrimination in its automated lending decisions. Acting as an independent **Data Governance Task Force**, we audited a dataset of 500+ raw credit applications across three dimensions: data quality, algorithmic fairness, and privacy/GDPR compliance.

Our analysis surfaces **critical governance risks** that require immediate remediation:

- **Data quality**: Multiple categories of defects identified — including duplicate records, type mismatches, invalid financial values, and inconsistent demographic encoding — affecting a material share of the dataset.
- **Algorithmic bias**: The Disparate Impact (DI) ratio for gender is **0.767**, below the four-fifths (0.8) regulatory threshold, indicating potential gender-based discrimination. Age-related disparities are also statistically significant.
- **Proxy discrimination**: ZIP code is statistically associated with both gender (Cramér's V = 0.36) and approval outcome, suggesting it acts as an indirect discriminatory feature even if gender is excluded from the model.
- **Privacy & governance**: Raw SSNs, email addresses, full names, exact dates of birth, and IP addresses are stored without protection. The dataset lacks consent tracking, retention metadata, audit trails, and human oversight documentation — all required under GDPR and the EU AI Act.

---

## Repository Structure

```
project-teamX/
├── README.md                          ← This file
├── data/
│   └── processed/
│       ├── credit_applications_clean.parquet
│       └── applications_privacy_safe.csv
├── notebooks/
│   ├── 01-data-quality.ipynb          ← Data Engineer
│   ├── 02-bias-analysis.ipynb         ← Data Scientist
│   └── 03_privacy_gdpr_ai_act.ipynb   ← Governance Officer
├── src/
│   └── fairness_utils.py              ← Reusable fairness helpers
└── presentation/                      ← Video link / file
```

---

## 1. Data Quality Findings

**Notebook:** `notebooks/01-data-quality.ipynb`

We profiled the raw JSON dataset across four quality dimensions — **completeness**, **consistency**, **validity**, and **accuracy** — and produced a fully documented issue inventory.

### Issue Inventory

| Issue | Dimension | Details |
|---|---|---|
| Duplicate `_id` records | Consistency | Same application ID appearing multiple times |
| Incomplete records (≥1 key field missing) | Completeness | At least one required field absent per record |
| `annual_income` stored as non-numeric | Consistency | Income values encoded as strings in a subset of records |
| `credit_history_months` stored as non-numeric | Consistency | Type mismatch preventing range validation |
| Negative `credit_history_months` | Validity | Impossible values indicating data entry errors |
| Negative `savings_balance` | Validity | Negative balances treated as invalid |
| `debt_to_income` out of range (<0 or >1.5) | Validity | Likely scaling or encoding errors |
| Invalid / unparseable `date_of_birth` | Validity | Entries that cannot be parsed into a valid date |
| Non-standard gender coding | Consistency | Mixed use of `male/female`, `m/f`, and other variants |
| Implausible age (<18 or >100) | Accuracy | Ages outside the plausible range for a credit applicant |

### Remediation Steps Applied

1. **Deduplication** — duplicate records removed by `_id`, keeping the first occurrence.
2. **Gender normalization** — all values mapped to `Male`, `Female`, or `Other/Unknown`.
3. **Numeric type enforcement** — all financial fields coerced to numeric with failed conversions set to `NaN`.
4. **Invalid value nullification** — negative credit history months, negative savings balances, and out-of-range DTI values replaced with `NaN` and flagged in dedicated indicator columns for audit traceability.
5. **Date parsing** — `date_of_birth` parsed using `dayfirst=True` to handle European formats; `age_years` derived as an analytical feature.

The cleaned dataset is exported to `data/processed/credit_applications_clean.parquet` and `.csv`.

---

## 2. Bias Detection & Fairness

**Notebook:** `notebooks/02-bias-analysis.ipynb`

### 2.1 Gender Disparate Impact

| Group | Approval Rate |
|---|---|
| Male | 65.99% |
| Female | 50.60% |

**DI ratio (female / male) = 0.767**

Since DI < 0.8, this dataset **triggers the four-fifths rule**, indicating a statistically and practically significant gender-based disparity in lending outcomes.

A chi-square test confirms the association is not due to chance (p = 0.00069). Effect size is small-to-moderate (Cramér's V = 0.152).

### 2.2 Age-Based Patterns

Approval rates vary significantly across age groups. A chi-square test on (age bin × approval) yielded p = 0.005 and Cramér's V = 0.183, indicating a statistically significant association between applicant age and loan outcome.

### 2.3 Proxy Discrimination — ZIP Code

Two separate chi-square tests reveal that ZIP code acts as a proxy variable:

| Test | p-value | Cramér's V |
|---|---|---|
| ZIP vs Gender | < 0.001 | 0.36 |
| ZIP vs Approval | < 0.05 | 0.21 |

Because ZIP code correlates strongly with gender and moderately with the outcome, including it in a model introduces **indirect discrimination** even when gender itself is excluded. This meets the definition of proxy discrimination under EU non-discrimination law.

### 2.4 Interaction Effects

Approval rates were computed for (age bin × gender) subgroups. The gender gap is not uniform across age categories — certain combinations (e.g., younger female applicants) show disproportionately low approval rates, suggesting intersectional discrimination patterns.

---

## 3. Privacy & GDPR Compliance

**Notebook:** `notebooks/03_privacy_gdpr_ai_act.ipynb`

### 3.1 PII Identified

The following personally identifiable information fields are present in the raw dataset **without protection**:

| Field | Risk |
|---|---|
| `applicant_info.full_name` | Direct identifier |
| `applicant_info.email` | Direct identifier |
| `applicant_info.ssn` | Highly sensitive — government ID |
| `applicant_info.ip_address` | Quasi-identifier; locatable |
| `applicant_info.date_of_birth` | Quasi-identifier |
| `applicant_info.zip_code` | Quasi-identifier |

### 3.2 Privacy Controls Demonstrated

| PII Field | Technique Applied |
|---|---|
| Email, SSN | SHA-256 hash (pseudonymization); raw values dropped |
| IP address | Last octet masked (`x.x.x.0`); raw field dropped |
| Date of birth | Converted to age bands (`18-25`, `26-35`, …); exact DOB dropped |
| Full name | Removed from analytics dataset |

The resulting privacy-safe dataset is saved to `data/processed/applications_privacy_safe.csv`.

### 3.3 GDPR Mapping

| GDPR Principle | Status | Article |
|---|---|---|
| Lawful basis | **Gap** — no consent timestamp or processing basis stored in records | Art. 6 |
| Transparency | **Gap** — no evidence of applicant notice or rights disclosure | Art. 13–14 |
| Data minimization | **Partially addressed** — PII removed/pseudonymized in analytics pipeline | Art. 5(1)(c) |
| Accuracy | **Addressed** — completeness checks and remediation documented | Art. 5(1)(d) |
| Storage limitation | **Gap** — no retention metadata or deletion workflow in dataset | Art. 5(1)(e) |
| Right to erasure | **Gap** — no mechanism to locate and delete a subject's records on request | Art. 17 |
| Security | **Partially addressed** — pseudonymization applied; RBAC/encryption not evidenced | Art. 32 |
| Accountability | **Partially addressed** — controls matrix documented; audit trail absent | Art. 30, Art. 5(2) |

### 3.4 EU AI Act Classification

NovaCred's credit scoring system qualifies as a **high-risk AI system** under Annex III of the EU AI Act (automated decisions with significant effect on individuals' access to financial services). This triggers mandatory requirements for:

- Documented data governance and quality management
- Technical documentation and logging
- Transparency and explainability for affected individuals
- Human oversight for edge-case decisions
- Ongoing bias monitoring and risk management

---

## 4. Governance Recommendations

### Immediate Actions (High Priority)

1. **Stop storing raw SSNs** in the operational database without encryption. Apply pseudonymization at ingestion.
2. **Add consent infrastructure** — record `consent_timestamp`, `processing_purpose`, and lawful basis per application before further processing.
3. **Freeze ZIP code as a model feature** pending proxy discrimination review; assess whether any geographic feature can be used without reproducing discriminatory patterns.
4. **Implement a retention schedule** — define `retention_until` per record and build an automated deletion workflow.

### Medium-Term Controls

5. **Introduce an audit log** capturing: `decision_log_id`, `model_version`, `decision_reason_codes`, `human_reviewer_id`, and `review_timestamp`.
6. **Add human oversight for borderline decisions** — cases near the approval threshold should require human review before a final outcome is issued (EU AI Act Art. 14).
7. **Run DI monitoring quarterly** — automate the DI ratio calculation on production decisions and alert when DI drops below 0.85 (buffer above the 0.8 threshold).
8. **Commission a data minimization audit** — evaluate whether `spending_behavior` categories are actually used in decisioning; if not, stop collecting them.

### Structural / Governance Controls

9. **Appoint a model risk officer** responsible for fairness audits and EU AI Act compliance documentation.
10. **Maintain a Records of Processing Activity (RoPA)** under Art. 30 GDPR, covering all processing operations on credit application data.

---

## 5. Individual Contributions

| Role | Responsibilities |
|---|---|
| Data Engineer | Data loading, cleaning pipeline, type enforcement, deduplication, `01-data-quality.ipynb`, repository structure |
| Data Scientist | Bias analysis, DI ratio, statistical testing, proxy analysis, interaction effects, `02-bias-analysis.ipynb` |
| Governance Officer | PII identification, pseudonymization demo, GDPR mapping, AI Act classification, `03_privacy_gdpr_ai_act.ipynb` |
| Product Lead | Coordination, video presentation, README, executive summary |

> Detailed individual contribution and commit history available via the GitHub commit log.

---

## 6. Technical Setup

```bash
# Install dependencies
pip install pandas numpy scipy matplotlib seaborn python-dateutil

# Optional: fairness library
pip install fairlearn

# Run notebooks in order
jupyter notebook notebooks/01-data-quality.ipynb
jupyter notebook notebooks/02-bias-analysis.ipynb
jupyter notebook notebooks/03_privacy_gdpr_ai_act.ipynb
```

The dataset (`raw_credit_applications.json`) must be placed in the repository root before running the notebooks. The data quality notebook must be run first, as it produces the cleaned CSV consumed by the bias analysis notebook.

---
