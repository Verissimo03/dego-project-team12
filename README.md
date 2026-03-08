# NovaCred — Credit Application Governance Analysis

> **DEGO 2606 · Data Ecosystems and Governance in Organizations**  
> MSc Business Analytics · Nova SBE  
> Group Project — Credit Application Governance Analysis

---

## Executive Summary

NovaCred is a fintech company under regulatory scrutiny for potential discrimination in its automated lending decisions. Acting as an independent **Data Governance Task Force**, we audited **500 raw credit applications** across three dimensions: data quality, algorithmic fairness, and privacy/GDPR compliance.

Our analysis surfaces **critical governance risks** requiring immediate remediation:

- **Data quality**: 18 records (3.59%) are affected by at least one data quality issue across 8 distinct issue categories spanning completeness, consistency, validity, and accuracy. All remediable issues were resolved programmatically with a full before/after audit trail.
- **Algorithmic bias**: The Disparate Impact (DI) ratio for gender is **0.767**, below the four-fifths (0.8) regulatory threshold. Female applicants are approved at **50.6%** vs **66.0%** for male applicants — a 15.4 percentage point gap confirmed statistically (p = 0.00069).
- **Proxy discrimination**: ZIP code is statistically associated with both gender (Cramér's V = 0.36, p < 0.001) and approval outcome (Cramér's V = 0.21), indicating a proxy discrimination risk even if gender is excluded from the model.
- **Privacy & governance**: Raw SSNs, email addresses, full names, exact dates of birth, and IP addresses are stored without protection across all 500 records. The dataset lacks consent tracking, retention metadata, audit trails, and human oversight documentation — all mandatory under GDPR and the EU AI Act.

---

## Team

| Name | Role | Student ID |
|---|---|---|
| Sarah Yousfi | Product Lead | 70987 |
| Edoardo Massimo Sirianni | Data Engineer | 72731 |
| Tiago Fernandes Cerejo Verissimo | Data Scientist | 72622 |
| Karl Harfouche | Governance Officer | 70044 |

---

## Repository Structure

```
project-team4/
├── README.md                             ← This file (primary written deliverable)
├── data/
│   └── processed/
│       ├── credit_applications_clean.parquet
│       └── applications_privacy_safe.csv
├── notebooks/
│   ├── 01-data-quality.ipynb             ← Data Engineer (Edoardo)
│   ├── 02-bias-analysis.ipynb            ← Data Scientist (Tiago)
│   └── 03_privacy_gdpr_ai_act.ipynb      ← Governance Officer (Karl)
├── src/
│   └── fairness_utils.py                 ← Reusable fairness helpers
└── presentation/                         ← Video link / file
```

---

## 1. Data Quality Assessment

**Notebook:** `notebooks/01-data-quality.ipynb` | **Role:** Data Engineer — Edoardo Massimo Sirianni

The raw JSON dataset was profiled across four quality dimensions — **completeness**, **consistency**, **validity**, and **accuracy** — using a reusable issue-logging pipeline (`show_issue`, `build_issue_table`). Every issue is quantified with counts and percentages and mapped to a quality dimension.

### Dataset Size Progression

| Stage | Records | Notes |
|---|---|---|
| Raw (`df`) | **500** | After loading `raw_credit_applications.json` |
| After deduplication (`df_clean`) | **498** | −2 duplicate records removed (keep-first policy) |
| Records with ≥1 issue | **18 (3.59%)** | Composite flag across all 8 issue masks |

### Issue Inventory — Before vs After Remediation

| Issue | Dimension | Count Before | % Before | Count After | % After |
|---|---|---|---|---|---|
| Missing/incomplete records (≥1 key field missing) | Completeness | 6 | 1.20% | 11 | 2.20% |
| Duplicate records (`_id`) | Consistency | 4 | 0.80% | 0 | 0.00% ✓ |
| Inconsistent/invalid date formats (DOB unparseable) | Validity | 4 | 0.80% | 4 | 0.80% |
| Inconsistent gender coding | Consistency | 2 | 0.40% | 0 | 0.00% ✓ |
| Negative `credit_history_months` | Validity | 2 | 0.40% | 0 | 0.00% ✓ |
| Negative `savings_balance` | Validity | 1 | 0.20% | 0 | 0.00% ✓ |
| `debt_to_income` out of range (<0 or >1.5) | Validity | 1 | 0.20% | 0 | 0.00% ✓ |
| Negative `annual_income` | Validity | 0 | 0.00% | 0 | 0.00% |

> **Note on completeness count increase (6 → 11 after cleaning):** Remediation steps that replace invalid values with `NaN` (e.g. negative credit history → `NaN`) expose previously hidden incompleteness in records that appeared complete in the raw data. This is the expected and correct behaviour of transparent nullification — it surfaces real gaps rather than masking them behind impossible values.

> **Note on unparseable DOB (4 records, persistent):** These 4 records contain date strings that could not be resolved even with multi-format parsing (`dayfirst=True`). They remain flagged and are excluded from all age-derived analysis. Remediation of these records requires manual correction at source.

### Remediation Steps Applied

1. **Deduplication** — 4 flagged records collapsed to 2 unique `_id` entries (keep-first policy). Dataset reduced 500 → 498.
2. **Gender normalization** — all values mapped to controlled vocabulary: `Male`, `Female`, `Other/Unknown`. Non-standard encodings (`M`, `F`, mixed case) fully resolved (2 records, 0 remaining).
3. **Numeric type enforcement** — all four financial fields (`annual_income`, `credit_history_months`, `debt_to_income`, `savings_balance`) coerced to `float` with `errors="coerce"`.
4. **Invalid value nullification** — negative credit history months, negative savings balances, and out-of-range DTI values replaced with `NaN`. Flag columns (`flag_neg_credit_history_months`, `flag_neg_savings_balance`, `flag_dti_out_of_range`) preserve traceability for auditors.
5. **Date parsing** — `date_of_birth` parsed with European-first convention (`dayfirst=True`); `age_years` derived as a downstream analytical feature.

Cleaned dataset exported to `data/processed/credit_applications_clean.parquet` and `.csv`.

---

## 2. Bias Detection & Fairness

**Notebook:** `notebooks/02-bias-analysis.ipynb` | **Role:** Data Scientist — Tiago Fernandes Cerejo Verissimo

All fairness analysis is performed on the cleaned dataset. Three analytical samples are used:

| Sample | N | % of Total |
|---|---|---|
| Total (raw flat) | 500 | 100.0% |
| Fairness sample (non-missing outcome + gender) | 500 | 100.0% |
| Age sample (non-missing outcome + gender + age) | 496 | 99.2% |

### 2.1 Gender Disparate Impact

| Group | Approval Rate |
|---|---|
| Female | **50.60%** |
| Male | **65.99%** |

**DI ratio (female / male) = 0.506 / 0.660 = 0.767**

Since DI < 0.8, this dataset **triggers the four-fifths rule**, indicating a statistically and practically significant gender-based disparity. The raw approval gap is **15.4 percentage points**.

Chi-square test of independence (female vs male applicants):

| Statistic | Value |
|---|---|
| χ² | 11.18 |
| p-value | **0.00069** |
| Cramér's V | **0.152** (small-to-moderate effect) |
| Decision | p < 0.001 — reject H₀ of independence |

DI is a screening metric: it identifies regulatory risk but does not establish causality. A root-cause investigation is required to determine whether the disparity originates in the model, the input features, or the underlying applicant population.

### 2.2 Age-Based Patterns

| Age Group | Count | Approvals | Approval Rate |
|---|---|---|---|
| 18–25 | 12 | 6 | 50.00% |
| 26–35 | 149 | 67 | **44.97%** ← lowest |
| 36–45 | 176 | 116 | **65.91%** ← highest |
| 46–55 | 90 | 58 | 64.44% |
| 56–65 | 56 | 35 | 62.50% |
| 66+ | 13 | 7 | 53.85% |

Chi-square test on age bin × approval outcome:

| Statistic | Value |
|---|---|
| p-value | **0.005** |
| Cramér's V | **0.183** |
| Decision | p < 0.01 — statistically significant |

The 26–35 group is approved at a rate **21 percentage points below** the 36–45 group — the largest single-group gap in the dataset. The age association warrants further governance review.

### 2.3 Interaction Effects — Gender × Age

Approval rates by age bin and gender reveal that the gender gap is **consistent in direction across all age groups**, indicating a systemic pattern rather than a localised anomaly:

| Age Group | Female Approval | Male Approval | Gap (M − F) |
|---|---|---|---|
| 18–25 | 50.00% | 50.00% | 0 pp |
| 26–35 | **34.18%** | 57.14% | **22.97 pp** ← largest gap |
| 36–45 | 59.52% | 71.74% | 12.22 pp |
| 46–55 | 60.98% | 66.67% | 5.69 pp |
| 56–65 | 54.84% | 72.00% | **17.16 pp** |
| 66+ | 50.00% | 60.00% | 10.00 pp |

The gender gap is most severe in the **26–35 cohort** (23 pp) and the **56–65 cohort** (17 pp), suggesting intersectional risk. Younger and older female applicants are disproportionately disadvantaged. Only the 18–25 group shows no gender gap, likely due to the very small sample (n = 12).

### 2.4 Proxy Discrimination — ZIP Code

| Test | p-value | Cramér's V | Interpretation |
|---|---|---|---|
| ZIP vs Gender | **< 0.001** | **0.36** | Strong association |
| ZIP vs Approval | **< 0.05** | **0.21** | Moderate association |

Because ZIP code is strongly associated with gender and moderately associated with the approval outcome, using it as a model feature introduces **indirect discrimination** even when gender itself is excluded. This meets the legal definition of proxy discrimination under EU non-discrimination law and the EU AI Act's data governance requirements (Art. 10).

**Governance implication:** ZIP code should be suspended as a model feature pending a formal proxy discrimination review.

---

## 3. Privacy & GDPR Compliance

**Notebook:** `notebooks/03_privacy_gdpr_ai_act.ipynb` | **Role:** Governance Officer — Karl Harfouche

### 3.1 PII Identified

The notebook identifies **7 PII fields** stored without protection across all 500 records:

| Field | PII Category | Risk Level |
|---|---|---|
| `applicant_info.full_name` | Direct identifier | High |
| `applicant_info.email` | Direct identifier | High |
| `applicant_info.ssn` | Direct identifier — government ID | **Critical** |
| `applicant_info.ip_address` | Online identifier (GDPR Art. 4) | High |
| `applicant_info.date_of_birth` | Quasi-identifier | Medium |
| `applicant_info.zip_code` | Quasi-identifier (also proxy variable — see Section 2.4) | Medium–High |
| `applicant_info.gender` | Protected attribute — special category risk | **High (Art. 9)** |

> **Gender as Art. 9 risk:** Gender is a protected attribute. Its use in a credit decision system creates heightened GDPR Art. 9 and anti-discrimination concerns. The DI = 0.767 finding further elevates the legal exposure. Gender should not be used as a model feature without a valid special-category legal basis.

> **`spending_behavior` gap:** The dataset contains granular behavioral transaction data (spending categories and amounts) without documented processing purpose or retention controls. Under Art. 5(1)(c), if this data is not a required model input, its collection must stop.

### 3.2 Privacy Controls Demonstrated

| PII Field | Technique Applied | Output |
|---|---|---|
| Email | SHA-256 pseudonymization; raw field dropped | `116648a7761525...` (64-char hash) |
| SSN | SHA-256 pseudonymization; raw field dropped | `2caf30528c21a1...` (64-char hash) |
| IP address | Last-octet masking (`x.x.x.0`); raw field dropped | e.g. `192.168.48.0` |
| Date of birth | Converted to age band; exact DOB dropped | e.g. `26-35` |
| Full name | Removed from analytics dataset | — |

**Age band distribution in privacy-safe dataset (500 records):**

| Age Band | Count |
|---|---|
| 36–45 | 170 |
| 26–35 | 158 |
| 46–60 | 108 |
| 60+ | 39 |
| 18–25 | 22 |
| Unparseable | 5 |

The privacy-safe dataset retains **20 columns** with all 5 direct identifiers removed. Saved to `data/processed/applications_privacy_safe.csv`.

> **Pseudonymization vs anonymization (GDPR Art. 4(5)):** SHA-256 hashing is pseudonymization, not anonymization. If an attacker holds the original value, they can hash and match it. Pseudonymized data remains personal data under GDPR and requires full compliance controls. True anonymization would require k-anonymity or differential privacy techniques.

### 3.3 GDPR Mapping

| GDPR Principle | Status | Gap Identified | Recommended Action | Article |
|---|---|---|---|---|
| Lawful basis | **Gap** | No `consent_timestamp` or processing basis stored per record | Add `consent_timestamp` and `processing_purpose` at ingestion | Art. 6 |
| Transparency | **Gap** | No evidence of applicant notice or rights disclosure | Implement pre-application data notice and Art. 22 disclosure for automated decisions | Art. 13–14 |
| Special categories | **Gap** | Gender collected and used without documented Art. 9 basis | Establish valid legal basis or remove gender from model features | Art. 9 |
| Data minimization | **Partial** | PII pseudonymized in analytics; `spending_behavior` retained without documented purpose | Commission minimization audit; remove unused behavioral data | Art. 5(1)(c) |
| Accuracy | **Addressed** | Completeness checks and remediation documented with before/after evidence | Maintain pipeline; add source validation at ingestion | Art. 5(1)(d) |
| Storage limitation | **Gap** | No `retention_until` field; no deletion workflow exists | Define retention schedule; implement automated purge | Art. 5(1)(e) |
| Right to erasure | **Gap** | No mechanism to locate and delete a subject's records | Implement erasure workflow; ensure hashed IDs support deletion lookup | Art. 17 |
| Automated decisions | **Gap** | 0% human review; no contestability mechanism for rejected applicants | Implement Art. 22(3) human review queue | Art. 22 |
| Security | **Partial** | Pseudonymization applied in analytics; RBAC and encryption not evidenced | Enforce RBAC + encryption at rest; add field-access audit logging | Art. 32 |
| Accountability | **Partial** | Controls documented; no production audit trail | Add `decision_log_id`, `model_version`, `human_reviewer_id` to decision records | Art. 30, Art. 5(2) |

### 3.4 EU AI Act Classification

NovaCred's credit scoring system qualifies as a **high-risk AI system** under **Annex III, Point 5(b)** of the EU AI Act — automated decisions with significant effect on individuals' access to financial services. This triggers mandatory obligations:

- Technical documentation and logging (Art. 11)
- Data governance and quality management
- Transparency and explainability for affected individuals
- Human oversight for edge-case and rejected decisions (Art. 14)
- Ongoing bias monitoring and risk management
- Formal conformity assessment before production deployment

---

## 4. Governance Recommendations

### Immediate Actions (High Priority)

1. **Suspend ZIP code as a model feature** pending a formal proxy discrimination review. Cramér's V = 0.36 with gender meets the threshold for indirect discrimination risk under EU law.
2. **Stop storing raw SSNs without encryption.** Apply pseudonymization at the point of ingestion — not post-hoc in analytics pipelines.
3. **Add consent infrastructure** — record `consent_timestamp`, `processing_purpose`, and lawful basis per application before any processing occurs.
4. **Implement a GDPR Art. 22(3) human review mechanism.** Currently 0% of decisions have human oversight. Rejected applicants must be able to request and receive human review — this is a critical compliance gap.

### Medium-Term Controls

5. **Define a retention schedule** with a `retention_until` field per record and an automated deletion workflow. Suggested periods: IP address 90 days; name/email/DOB/gender 5 years; SSN and decision records 7 years; audit logs 10 years.
6. **Introduce a decision audit log** capturing: `decision_log_id`, `model_version`, `decision_reason_codes`, `human_reviewer_id`, and `review_timestamp` for every application outcome.
7. **Run DI monitoring quarterly** — automate the DI ratio calculation on production decisions and trigger a governance review when DI drops below 0.85 (buffer above the 0.8 regulatory threshold).
8. **Commission a data minimization audit** on `spending_behavior` — if spending categories are not model inputs, their collection must stop under Art. 5(1)(c).

### Structural Governance Controls

9. **Appoint a model risk officer** responsible for fairness audits, EU AI Act technical documentation, and ongoing DI monitoring.
10. **Maintain a Records of Processing Activity (RoPA)** under Art. 30 GDPR, covering all operations on credit application data.
11. **Formalize a DPIA** (Art. 35) before any production deployment, given the high-risk AI classification and the fully automated decision-making exposure under Art. 22.

---

## 5. Individual Contributions

| Name | Role | Key Contributions |
|---|---|---|
| Edoardo Massimo Sirianni | Data Engineer | JSON loading and normalization, `show_issue` / `build_issue_table` pipeline, remediation code, before/after validation table, flag columns, cleaned dataset export — `01-data-quality.ipynb` |
| Tiago Fernandes Cerejo Verissimo | Data Scientist | DI ratio calculation, chi-square and Cramér's V tests, age-based analysis, interaction pivot, ZIP proxy analysis, all visualizations — `02-bias-analysis.ipynb` |
| Karl Harfouche | Governance Officer | PII identification and classification, pseudonymization and masking demos, GDPR article mapping, EU AI Act classification, governance gap analysis — `03_privacy_gdpr_ai_act.ipynb` |
| Sarah Yousfi | Product Lead | Project coordination, video presentation, README, executive summary, milestone tracking |

> Individual commit history is verifiable via the GitHub commit log. All team members contributed under their own GitHub accounts.

---

## 6. Technical Setup

```bash
# Install dependencies
pip install pandas numpy scipy matplotlib seaborn python-dateutil

# Optional: fairness library (used in bias notebook)
pip install fairlearn

# Run notebooks in order — notebook 02 depends on the CSV produced by notebook 01
jupyter notebook notebooks/01-data-quality.ipynb
jupyter notebook notebooks/02-bias-analysis.ipynb
jupyter notebook notebooks/03_privacy_gdpr_ai_act.ipynb
```

**Important:** Place `raw_credit_applications.json` in the **repository root** (not inside `notebooks/`) before running. All three notebooks resolve paths relative to the repo root. Notebook 01 must complete first as it produces `outputs/credit_applications_clean.csv`, which notebook 02 loads as its primary input.

---

*Version 1.0 · DEGO 2606 · Nova SBE · Team 4*