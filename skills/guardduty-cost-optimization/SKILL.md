---
name: guardduty-cost-optimization
description: Identify and quantify Amazon GuardDuty cost optimization opportunities.
  Use this skill when a user asks to reduce, review, audit, or optimize GuardDuty
  spend, or reports an unexpected GuardDuty cost increase or an expensive protection
  plan. Activate on requests like "why is my GuardDuty bill so high", "reduce
  GuardDuty costs", "GuardDuty cost review", "which GuardDuty protection plan costs
  the most", "is GuardDuty S3 Protection worth it", or "project my GuardDuty spend
  after the free trial". This skill analyzes enabled protection plans and their
  per-data-source usage from the AWS/GuardDuty CloudWatch usage metrics through
  read-only APIs to surface high-cost/low-signal protection plans, VPC-Flow-Log
  charges offset by Runtime Monitoring, expensive S3/data-event analysis,
  free-trial cost projection, and duplicate multi-account coverage, producing a
  severity-ranked report of savings.
metadata:
  author: holmalla
  version: "1.0.0"
  aws-devops-agent-skills.agent-types: "Chat tasks, Evaluation"
  aws-devops-agent-skills.aws-services: "Amazon GuardDuty"
  aws-devops-agent-skills.technical-domains: "Security, Cost Optimization"
---

# Amazon GuardDuty Cost Optimization

Identify, quantify, and prioritize Amazon GuardDuty cost optimization opportunities
aligned with [Monitoring GuardDuty usage and estimating costs](https://docs.aws.amazon.com/guardduty/latest/ug/monitoring_costs.html)
and [GuardDuty pricing](https://aws.amazon.com/guardduty/pricing/).

This skill uses **read-only GuardDuty, CloudWatch, and Organizations APIs only**. It
never enables, disables, or reconfigures a detector or protection plan — all
remediation is delivered as recommendations for a human to review and apply. It reads
usage metrics and findings statistics only; it does not read finding detail content.

## When to Use

Activate this skill when the user asks to:
- Reduce or optimize Amazon GuardDuty costs
- Investigate an unexpected GuardDuty cost increase
- Understand which protection plan or data source drives GuardDuty spend
- Decide whether a protection plan (S3 Protection, Runtime Monitoring, etc.) is worth its cost
- Project GuardDuty spend after the 30-day free trial
- Perform a GuardDuty cost review or FinOps assessment

## How GuardDuty Billing Works

GuardDuty is pay-as-you-go, **per protection plan**, priced on the volume of data
each plan analyzes. There are no upfront costs and no per-detector fee — cost is
driven entirely by analyzed volume. Each plan meters on its own unit:

| Protection Plan | Data Source | Metric (namespace `AWS/GuardDuty`) | Unit | Priced on |
|-----------------|-------------|------------------------------------|------|-----------|
| Foundational Threat Detection | CloudTrailEvents | AnalyzedCount | Count | Management events analyzed |
| Foundational Threat Detection | VPCFlowLogDNSLogEvents | AnalyzedBytes | Bytes | VPC flow + DNS log volume |
| S3 Protection | S3DataEvents | AnalyzedCount | Count | S3 data events analyzed |
| EKS Protection | KubernetesAuditLogs | AnalyzedCount | Count | EKS audit log events |
| Runtime Monitoring | RuntimeMonitoringEC2 / EKS / Fargate | MonitoredVcpuHours | vCPU-Hours | vCPU hours monitored |
| Malware Protection for EC2 | MalwareProtectionEBS / OnDemandEBS* | ScannedBytes | Bytes | EBS data scanned |
| RDS Protection | RDS / RDSLimitless / AuroraScaleout | MonitoredAcuHours / MonitoredVcpuHours | ACU/vCPU-Hours | RDS/Aurora capacity monitored |
| Lambda Protection | LambdaNetworkLogs | AnalyzedBytes | Bytes | Lambda network log volume |
| AI Protection | AIDataEvents | AnalyzedBytes | Bytes | AI data events analyzed |

Malware Protection for S3 meters separately under the `AWS/GuardDuty/MalwareProtection`
namespace (`CompletedScanBytes`, `CompletedScanCount`, etc.).

Two critical billing behaviors the skill reasons about:
- **Runtime Monitoring offsets VPC Flow Log charges.** For instances actively
  monitored by the Runtime Monitoring agent, GuardDuty does **not** charge for VPC
  Flow Logs processing on those instances. Enabling Runtime Monitoring *decreases*
  `VPCFlowLogDNSLogEvents` usage; disabling it restores the charge. The two line items
  trade against each other.
- **Service-log configuration does not reduce GuardDuty cost.** Filtering or disabling
  your own VPC Flow Logs, CloudTrail, or S3 data event logging does **not** reduce
  what GuardDuty analyzes — GuardDuty ingests from independent internal sources. The
  only cost lever is the GuardDuty **protection-plan** configuration itself.

## Step 1: Identify Target Scope

Ask the user which accounts and Regions to review, and whether this is a standalone
account, a GuardDuty delegated-administrator account, or a member account. Accept
specific account IDs and Regions, "all regions", or "organization". If no scope is
given, default to the current account across all Regions with a 30-day analysis
window.

Delegated-administrator accounts additionally receive **aggregated** organization
usage metrics — use them for org-wide sizing.

## Step 2: Inventory Detectors and Protection Plans

```
guardduty.ListDetectors / GetDetector             # detector status, enabled features,
                                                   # data sources, per-plan config
guardduty.ListMembers / GetMemberDetectors        # org member coverage (deleg. admin)
guardduty.GetMasterAccount / ListOrganizationAdminAccounts
guardduty.GetFindingsStatistics                   # finding counts by type/severity
                                                   # (value signal — statistics only,
                                                   # not finding detail content)
```

Capture per Region: whether GuardDuty is enabled, which protection plans/features are
on, Runtime Monitoring agent coverage, and member-account coverage.

## Step 3: Collect Per-Plan Usage Metrics

Pull the `AWS/GuardDuty` usage metrics with `cloudwatch.GetMetricData`, using the
`DataSource` dimension (and `AccountId`) to break usage down by protection plan over
the analysis window. Also pull `AWS/GuardDuty/MalwareProtection` for S3 malware scans.

- Usage metrics are published **hourly** and can lag up to ~24 hours.
- On a delegated-administrator account, the aggregated `DataSource` dimensions give
  org-wide totals per plan.
- **Cost Explorer** (`ce.GetCostAndUsage`, filtered to the `AmazonGuardDuty` service,
  grouped by `USAGE_TYPE` and/or `REGION`) is the most direct dollar signal — prefer
  it when available and reconcile it against the per-plan usage metrics.

Convert byte metrics to GB/TB when sizing (1 GB = 1,073,741,824 bytes; 1 TB =
1,099,511,627,776 bytes) to match pricing units.

## Step 4: Analyze Cost Optimization Opportunities

Rank each enabled protection plan by its share of total GuardDuty spend, then evaluate
the checks below. Assign each finding a severity (CRITICAL, HIGH, MEDIUM, LOW, INFO)
and, where usage/cost signal exists, an estimated monthly saving. **Frame every
recommendation against security value** — GuardDuty is a security control, and cost
reductions must not silently remove needed coverage.

### 4.1 High-cost / low-signal protection plans
Ref: [GuardDuty pricing](https://aws.amazon.com/guardduty/pricing/)

- A protection plan consuming a large share of spend while producing few or no
  findings over a representative window → review whether its coverage is warranted for
  the workload → **MEDIUM** (present as a value/cost tradeoff, not an automatic
  "disable"). Use `GetFindingsStatistics` for the value side.

### 4.2 Runtime Monitoring ↔ VPC Flow Log offset
Ref: [Monitoring GuardDuty usage and estimating costs](https://docs.aws.amazon.com/guardduty/latest/ug/monitoring_costs.html)

- High `VPCFlowLogDNSLogEvents` (AnalyzedBytes) spend **and** EC2/EKS workloads not
  covered by the Runtime Monitoring agent → enabling Runtime Monitoring stops VPC Flow
  Log processing charges on monitored instances and adds deeper runtime detection →
  compare `MonitoredVcpuHours` cost vs the avoided VPC Flow Log cost → **MEDIUM**
  opportunity when the offset is favorable.
- Runtime Monitoring enabled but the **agent not actually transmitting** on many
  instances → paying for VPC Flow Logs *and* getting no runtime coverage → fix agent
  coverage → **MEDIUM**.

### 4.3 S3 Protection cost vs value
Ref: [GuardDuty S3 Protection](https://docs.aws.amazon.com/guardduty/latest/ug/s3-protection.html)

- High `S3DataEvents` (AnalyzedCount) spend on buckets with predictable,
  high-volume, low-risk access patterns (e.g. internal data-lake churn) → weigh S3
  Protection cost against exfiltration/destruction risk for those buckets → **MEDIUM**
  tradeoff.

### 4.4 Malware Protection for S3 scan volume
Ref: [Pricing in GuardDuty](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty-pricing.html)

- High `CompletedScanBytes` (namespace `AWS/GuardDuty/MalwareProtection`) driven by
  scanning large, low-risk, or frequently-rewritten objects → scope Malware Protection
  for S3 to the buckets/prefixes that need it → **MEDIUM**. Note On-demand malware
  scan has **no** free tier.

### 4.5 Free-trial cost projection (proactive)
Ref: [Estimating GuardDuty cost](https://docs.aws.amazon.com/guardduty/latest/ug/monitoring_costs.html#estimating_guardduty_cost)

- One or more plans within the **30-day free trial** → project post-trial monthly cost
  from the observed trial usage metrics **before** the bill lands, per plan → **HIGH**
  visibility (prevents bill shock; lets the user disable a plan before it starts
  charging if the projected cost outweighs value).

### 4.6 Duplicate / inconsistent multi-account coverage
Ref: [GuardDuty pricing](https://aws.amazon.com/guardduty/pricing/)

- In an organization, protection plans enabled inconsistently across members, or
  enabled on accounts/Regions with no meaningful workload → align coverage to where
  workloads and risk actually are → **MEDIUM**.
- GuardDuty enabled in Regions the organization does not use → disable in unused
  Regions → **MEDIUM**.

### 4.7 Security Hub consolidated pricing (informational)
Ref: [Monitoring GuardDuty usage and estimating costs](https://docs.aws.amazon.com/guardduty/latest/ug/monitoring_costs.html#security-hub-customers)

- If the account uses (or is considering) the Security Hub Threat Analytics plan, note
  that it consolidates metering of multiple GuardDuty data sources and can change the
  effective cost model → **INFO** (surface for the user's FinOps decision; the free
  trial status is independent of Security Hub).

## Step 5: Generate Report

Generate a shareable Markdown report artifact.

Artifact naming: `guardduty-cost-optimization-<account-id>-<YYYY-MM-DD>.md`
Example: `guardduty-cost-optimization-123456789012-2026-09-24.md`

Structure:

### Report Header
```
# Amazon GuardDuty Cost Optimization — <account-id>
Date: <YYYY-MM-DD> | Scope: <regions / organization> | Analysis window: <start> to <end>
```

### Executive Summary
- Estimated total monthly savings (sum of quantified opportunities) or "not quantified"
- Finding counts by severity
- Top 3 opportunities by estimated saving
- Any plan still in the 30-day free trial with projected post-trial cost

### Protection Plan Usage & Spend
| Protection Plan | Data Source | Usage (window) | Est. Monthly Cost | Findings (window) | Share |
|-----------------|-------------|----------------|-------------------|-------------------|-------|

### Cost Optimization Opportunities
| # | Opportunity | Severity | Current State | Recommendation (value tradeoff) | Est. Monthly Saving |
|---|-------------|----------|---------------|----------------------------------|---------------------|

### Free-Trial Projection (if any plan in trial)
| Protection Plan | Trial usage rate | Projected monthly cost | Trial ends |
|-----------------|------------------|------------------------|------------|

### Priority Matrix
| # | Opportunity | Severity | Effort | Est. Saving |
|---|-------------|----------|--------|-------------|

### Next Steps
- Immediate (HIGH — within 7 days, e.g. free-trial decisions)
- Short-term (MEDIUM — within 30 days)
- Long-term (LOW/INFO)

### Appendix — Reference Links
- [Monitoring GuardDuty usage and estimating costs](https://docs.aws.amazon.com/guardduty/latest/ug/monitoring_costs.html)
- [GuardDuty pricing](https://aws.amazon.com/guardduty/pricing/)
- [Pricing in GuardDuty](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty-pricing.html)
- [GuardDuty S3 Protection](https://docs.aws.amazon.com/guardduty/latest/ug/s3-protection.html)
- [Runtime Monitoring](https://docs.aws.amazon.com/guardduty/latest/ug/runtime-monitoring.html)

## Severity Definitions

| Severity | Definition | SLA |
|----------|------------|-----|
| CRITICAL | Runaway cost causing large ongoing overspend | Fix within 24–48 hours |
| HIGH | Clear, sizable recurring saving, or a time-boxed free-trial decision | Fix within 1 week |
| MEDIUM | Notable saving with a value tradeoff to weigh | Plan within 30 days |
| LOW | Minor saving or hygiene | Address when convenient |
| INFO | Observation, no action required | N/A |

## Safety and Boundaries

- **Read-only.** The skill calls only `List*`, `Get*`, `Describe*` APIs and CloudWatch
  reads. It never calls `CreateDetector`, `UpdateDetector`, `DeleteDetector`,
  `DisableOrganizationAdminAccount`, or any protection-plan mutation.
- **Security value first.** GuardDuty is a security control. Never recommend disabling
  a protection plan purely on cost — always frame it as a cost-vs-risk tradeoff, cite
  the finding activity for that plan, and defer the decision to the user's security
  posture. Removing coverage can create undetected exposure.
- **Proposed changes are suggestions.** Every recommendation is for a human to review
  and apply.

## Known Quirks

- **Your own log configuration does not change GuardDuty cost** — do not recommend
  turning off customer VPC Flow Logs, CloudTrail, or S3 data events to reduce
  GuardDuty spend; GuardDuty reads independent internal sources. The lever is the
  GuardDuty protection-plan config.
- **Runtime Monitoring and VPC Flow Log charges trade against each other** — size the
  net effect, not either line item alone. If the agent stops transmitting, VPC Flow
  Log charges silently resume.
- Usage metrics lag up to ~24 hours and are hourly — use a multi-day window, not a
  single hour, for sizing.
- Byte-unit metrics must be converted to GB/TB to match pricing tiers.
- The 30-day free trial is **per account, per plan**, and its status is independent of
  Security Hub integration — enabling Security Hub does not grant, extend, or restart a
  trial.
- Malware Protection for S3 lives in a **separate** CloudWatch namespace
  (`AWS/GuardDuty/MalwareProtection`) from the other plans (`AWS/GuardDuty`).
