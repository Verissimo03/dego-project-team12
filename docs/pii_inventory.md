# PII Inventory & Handling Plan

## PII fields in this dataset (flattened names)
- `applicant_info.full_name`
- `applicant_info.email`
- `applicant_info.ssn`
- `applicant_info.ip_address`
- `applicant_info.date_of_birth`
- `applicant_info.zip_code`

## Treatment plan (analytics dataset)

| Field | Type | Risk | Treatment |
|---|---|---|---|
| full_name | Direct ID | Re-identification | Remove from analytics dataset |
| email | Direct ID | Re-identification | Hash/tokenize (stable ID if needed) |
| ssn | Direct ID | High sensitivity | Hash/tokenize + store separately; never used in analytics |
| ip_address | Direct ID | Tracking | Mask (e.g., /24) or remove |
| date_of_birth | Direct ID | Re-identification | Convert to age band; drop exact DOB |
| zip_code | Quasi-ID | Proxy + re-identification | Coarsen (region/prefix) + proxy risk review |

## Separation of environments
- **PII store:** restricted access (security/compliance only)
- **Analytics store:** pseudonymized dataset for engineering/science
- **Modeling features:** only derived fields (age band, region/prefix, income band)