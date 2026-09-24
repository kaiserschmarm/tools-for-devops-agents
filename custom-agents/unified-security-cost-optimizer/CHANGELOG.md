# Changelog

## 1.0.0

- Initial version
- System prompt with Goal/Approach/Constraints/Output structure
- Routes to the `cloudtrail-cost-optimization`, `config-cost-optimization`, and `guardduty-cost-optimization` skills for domain knowledge
- Requires the `use_aws` tool for read-only resource and usage inspection
- Read-only, security-and-compliance-first: frames every reduction as a cost-vs-risk tradeoff and defers the decision to the customer
- Produces per-skill artifacts for single-service reviews and a consolidated `security-cost-optimization-<account-id>-<YYYY-MM-DD>.md` for multi-service reviews
- Output includes executive summary, opportunities by service, cross-service observations, consolidated priority matrix, and next steps
- Severity-based prioritization (CRITICAL, HIGH, MEDIUM, LOW, INFO) with estimated monthly savings
