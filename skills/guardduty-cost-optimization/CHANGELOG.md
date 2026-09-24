# Changelog

## [1.0.0] - 2026-09-24
### Added
- Initial release of the Amazon GuardDuty cost optimization skill for AWS DevOps Agent
- Read-only inventory of detectors, enabled protection plans, Runtime Monitoring coverage, and organization member coverage
- Per-protection-plan spend attribution from the `AWS/GuardDuty` and `AWS/GuardDuty/MalwareProtection` CloudWatch usage metrics, reconciled against Cost Explorer when available
- Cost optimization checks across seven areas: high-cost/low-signal protection plans, the Runtime Monitoring ↔ VPC Flow Log charge offset, S3 Protection cost-vs-value, Malware Protection for S3 scan volume, 30-day free-trial post-trial cost projection, duplicate/inconsistent multi-account and unused-Region coverage, and Security Hub consolidated-pricing awareness
- Cost-vs-security framing throughout, using `GetFindingsStatistics` as the value signal
- Severity-ranked findings (CRITICAL, HIGH, MEDIUM, LOW, INFO) and a shareable Markdown report artifact named `guardduty-cost-optimization-<account-id>-<YYYY-MM-DD>.md`
- Read-only guidance — no detector or protection-plan mutations
- Evaluation test cases (5 functional evals, 6 trigger queries)
