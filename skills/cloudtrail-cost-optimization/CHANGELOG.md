# Changelog

## [1.0.0] - 2026-09-24
### Added
- Initial release of the CloudTrail cost optimization skill for AWS DevOps Agent
- Read-only inventory of trails (including multi-Region shadow and Organizations trails) and CloudTrail Lake event data stores
- Cost optimization checks across seven areas: duplicate management-event trails, unnecessary Read management events, high-volume noise events (AWS KMS, RDS Data API), overly broad and duplicate data event logging, CloudTrail Lake pricing option/retention/duplicate ingestion, S3 destination lifecycle hygiene, and idle/stopped trails
- Cost attribution via Cost Explorer usage types and volume proxies (CloudWatch usage metrics, S3 bucket size) with estimated monthly savings
- Severity-ranked findings (CRITICAL, HIGH, MEDIUM, LOW, INFO) and a shareable Markdown report artifact named `cloudtrail-cost-optimization-<account-id>-<YYYY-MM-DD>.md`
- Compliance-aware, read-only guidance — no trail, event selector, or Lake mutations
- Evaluation test cases (5 functional evals, 6 trigger queries)
