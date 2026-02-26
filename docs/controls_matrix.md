# Controls Matrix 

This matrix links each key risk to: (1) how we detect it, (2) the control, (3) the owner, and (4) the evidence artifact.

## Summary table

| Risk | Detection / Evidence | Control | Owner | Frequency | Artifact |
|---|---|---|---|---|---|
| Duplicate identities | Duplicate SSN/email counts | Dedup + fraud review workflow | Data Eng + Governance | Daily/Weekly | reports/data_quality_summary.md |
| Inaccurate/incomplete data | Missingness + invalid values | Validation rules + exception queue | Data Eng | Daily | notebooks/01_data_quality.ipynb |
| Unfair outcomes | DI ratio by gender | Fairness threshold + escalation | Data Sci + Governance | Monthly | reports/fairness_summary.md |
| Proxy discrimination | ZIP ↔ gender/outcome link | Proxy review + feature coarsening | Data Sci + Governance | Monthly | notebooks/02_bias_and_proxy.ipynb |
| PII exposure | PII inventory + access logs | Tokenization + RBAC + encryption | Governance + Security | Continuous | docs/pii_inventory.md |
| Missing audit trail | Logging gaps | Transformation + access logs | Governance | Continuous | docs/governance_framework.md |
| Over-retention | No retention rules | Retention schedule + deletion workflow | Governance | Quarterly | docs/governance_framework.md |


## Required metadata to support compliance (recommended additions)
Add these fields to make governance auditable:
- `consent_timestamp`, `processing_purpose`, `data_source`, `retention_until`
- `decision_log_id`, `model_version`, `reason_codes`, `human_reviewer_id`, `review_timestamp`

These fields are not present in the provided dataset. They are recommended governance metadata to support GDPR accountability (lawful basis, retention, and data subject rights) and EU AI Act requirements (logging, documentation, human oversight).