# Governance Framework — NovaCred Credit Application Governance

## 1) Objectives
- Ensure credit decision data is **accurate**, **auditable**, and **fit for purpose**
- Protect **PII** and reduce privacy risk
- Detect and mitigate **bias** and **proxy discrimination**
- Support accountability: **logging, retention, human oversight**

## 2) Roles (RACI-style)
- **Data Engineer:** ingestion, validation rules, DQ monitoring, pipeline reproducibility
- **Data Scientist:** fairness metrics, proxy analysis, monitoring
- **Governance Officer:** privacy controls, GDPR/AI Act mapping, control framework, audit readiness

## 3) Data lifecycle controls

### Ingestion
- Schema validation (required fields + types)
- Duplicate detection (application ID, SSN, email)
- Format validation (DOB, ZIP, IP)

### Storage
- Separate PII store vs analytics store
- Encryption at rest + in transit
- RBAC (least privilege) + access logs

### Processing
- Pseudonymize direct IDs (email, SSN)
- Minimize features (remove direct identifiers from modeling)
- Log transformations (what changed, when, by whom)

### Decisioning
- Human-in-the-loop policy for edge cases
- Decision log: model version + reason codes + reviewer + timestamp
- Monitoring: drift + fairness checks

### Retention & deletion
- Retention period defined per record
- Erasure workflow (GDPR Art. 17) + audit evidence

## 4) Governance KPIs
- **Data Quality:** missingness %, duplicate rate, invalid values rate
- **Fairness:** DI ratio by gender (four-fifths rule threshold)
- **Privacy:** % records pseudonymized, access log completeness