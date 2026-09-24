---
name: cloudtrail-cost-optimization
description: Identify and quantify AWS CloudTrail cost optimization opportunities.
  Use this skill when a user asks to reduce, review, audit, or optimize CloudTrail
  spend, or reports an unexpected CloudTrail cost or usage increase. Activate on
  requests like "why is my CloudTrail bill so high", "reduce CloudTrail costs",
  "find duplicate CloudTrail trails", "CloudTrail cost review", "optimize CloudTrail
  Lake", or "CloudTrail data events are expensive". This skill analyzes trails,
  event selectors, and CloudTrail Lake event data stores through read-only AWS APIs
  to surface duplicate management-event trails, unnecessary read events, high-volume
  noise events (KMS, RDS Data API), overly broad data event logging, and Lake
  ingestion/retention waste, producing a severity-ranked report of savings.
metadata:
  author: holmalla
  version: "1.0.0"
  aws-devops-agent-skills.agent-types: "Chat tasks, Evaluation"
  aws-devops-agent-skills.aws-services: "AWS CloudTrail"
  aws-devops-agent-skills.technical-domains: "Security, Cost Optimization"
---

# AWS CloudTrail Cost Optimization

Identify, quantify, and prioritize AWS CloudTrail cost optimization opportunities
aligned with [Managing CloudTrail trail costs](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-trail-manage-costs.html)
and [CloudTrail pricing](https://aws.amazon.com/cloudtrail/pricing/).

This skill uses **read-only CloudTrail, CloudWatch, S3, and Organizations APIs
only**. It never creates, updates, or deletes a trail, event data store, or event
selector — all remediation is delivered as recommendations for a human to review
and apply. It does not read the content of any logged event.

## When to Use

Activate this skill when the user asks to:
- Reduce or optimize AWS CloudTrail costs
- Investigate an unexpected CloudTrail cost or usage spike
- Find duplicate or redundant trails across accounts/regions
- Review event selectors, data event logging, or CloudTrail Lake spend
- Perform a CloudTrail cost review or FinOps assessment

## How CloudTrail Billing Works

Understanding the pricing model is the foundation of every finding below.

| Charge | Billed | Free allowance |
|--------|--------|----------------|
| Management events | Per event delivered to a trail, beyond the first copy per Region | First copy per Region is **free** |
| Data events | Per event delivered — **every** copy is billed, including the first | None |
| Network activity events | Per event delivered | None |
| CloudTrail Lake ingestion | Per GB ingested (pricing depends on the event data store's pricing option) | None |
| CloudTrail Lake storage | Per GB-month beyond the included retention | Varies by pricing option |
| S3 storage of log files | Standard S3 storage on the destination bucket | None |

Key consequences the skill reasons about:
- A **second copy** of the same management events in a Region always costs money.
  A multi-Region trail already covers every Region, so any additional single-Region
  trail capturing the same management events is a paid duplicate.
- An **Organizations trail** is replicated into every member account. A member
  account that also runs its own trail for the same management events pays for a
  second copy.
- **Data events are never free** — narrowing their scope with advanced event
  selectors is almost always a direct saving.
- **KMS and RDS Data API events** can dominate management-event volume (e.g.
  SSE-KMS on busy S3 buckets), and can be excluded via event selectors.

## Step 1: Identify Target Scope

Ask the user which accounts and Regions to review, and whether the account is a
standalone account, an Organizations management/delegated-administrator account, or
a member account. Accept:
- Specific account IDs and Regions
- "all regions" for a given account
- "organization" to reason about org-wide trail duplication

If no scope is given, default to the current account across all Regions, and set the
CloudWatch/usage analysis window to the last 30 days unless the user specifies a
different range.

## Step 2: Inventory Trails and Event Data Stores

Collect the complete trail and Lake inventory:

```
cloudtrail.DescribeTrails (includeShadowTrails=true)   # all trails visible in the Region,
                                                        # including multi-Region shadow copies
cloudtrail.GetTrailStatus                               # is the trail logging? IsLogging
cloudtrail.GetTrail                                     # per-trail config
cloudtrail.GetEventSelectors                            # basic + advanced event selectors,
                                                        # read/write type, KMS/RDS exclusions,
                                                        # data event resource scope
cloudtrail.ListTrails                                   # enumerate across Regions
cloudtrail.ListEventDataStores / GetEventDataStore      # CloudTrail Lake stores: pricing
                                                        # option, retention period, multi-region,
                                                        # org enablement, event category
```

For organization scope, use `organizations.DescribeOrganization` and
`organizations.ListAccounts` to understand how many member accounts an Organizations
trail replicates into.

Capture per trail: name, ARN, `IsMultiRegionTrail`, `IsOrganizationTrail`,
`IsLogging`, home Region, S3 destination bucket, whether it logs management events
(and read/write type), whether it logs data events (and their scope), and any
KMS/RDS Data API exclusion.

## Step 3: Collect Usage and Volume Signals

CloudTrail does not publish per-trail event counts as a first-class metric, so
combine these signals to size each opportunity:

- **CloudWatch `AWS/CloudTrail` usage metrics** (via `cloudwatch.GetMetricData`) where
  available for the account, to trend delivered event volume over the window.
- **S3 destination bucket size** (`s3.ListObjectsV2` / CloudWatch `BucketSizeBytes`
  on the log bucket prefix) as a proxy for relative trail volume when comparing
  trails that write to distinct buckets/prefixes.
- **CloudTrail Lake** event data store size and retention from `GetEventDataStore`.
- **Cost Explorer** (`ce.GetCostAndUsage`, filtered to the `AWSCloudTrail` service,
  grouped by `USAGE_TYPE`) to attribute spend to `PaidEventsRecorded`,
  data events, and Lake ingestion/storage usage types. This is the most direct
  dollar signal — prefer it when the role has Cost Explorer access.

If neither Cost Explorer nor usage metrics are available, still report the
configuration findings (duplicates, read events, data event scope) and label the
dollar impact as "not quantified — enable Cost Explorer for sizing".

## Step 4: Analyze Cost Optimization Opportunities

Evaluate every trail and event data store against the checks below. Assign each
finding a severity (CRITICAL, HIGH, MEDIUM, LOW, INFO) and, wherever a usage or cost
signal exists, an estimated monthly saving.

### 4.1 Duplicate management-event trails (highest-impact, most common)
Ref: [Troubleshoot CloudTrail cost and usage increases](https://repost.aws/knowledge-center/remove-duplicate-cloudtrail-events)

- More than one trail delivering **management events** in the same Region → every
  copy after the first is billable. Keep one trail (ideally the org or multi-Region
  trail) logging management events and turn management event logging **off** on the
  duplicates → **HIGH** (frequently the single largest CloudTrail line item; org-wide
  de-duplication can cut CloudTrail spend substantially).
- A **multi-Region trail plus an additional single-Region trail** capturing the same
  management events → the single-Region trail is a paid duplicate → **HIGH**.
- A **member account trail** duplicating the management events already captured by an
  **Organizations trail** → **HIGH**.
- Report which trail to keep (prefer the broadest-scope, org/multi-Region, actively
  logging trail) and which to convert to data-events-only or disable.

### 4.2 Read management events that aren't needed
Ref: [Optimize CloudTrail costs and maintain compliance](https://repost.aws/knowledge-center/optimize-cloudtrail-compliance)

- A **paid** (non-free-copy) trail logging **Read** management events when the use
  case only needs Write events → drop Read events → **MEDIUM**. (The free first copy
  can safely log both; this applies to the duplicate/paid copies.)

### 4.3 High-volume noise events
Ref: [Controlling CloudTrail costs using KMS event filtering](https://aws.amazon.com/blogs/aws-cost-management/launch-controlling-aws-cloudtrail-costs-using-aws-kms-event-filtering/)

- A **paid** trail not excluding **AWS KMS** events on accounts with heavy SSE-KMS
  usage (busy S3, EBS, Secrets Manager) → KMS events can dominate volume → exclude
  via event selectors → **MEDIUM**.
- A **paid** trail not excluding **RDS Data API** events on accounts using the Data
  API heavily → exclude → **MEDIUM**.
- Note: exclusions apply to paid copies. Do not recommend excluding events from the
  single authoritative trail if the user needs them for security/audit.

### 4.4 Overly broad data event logging
Ref: [Filtering data events with advanced event selectors](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/filtering-data-events.html)

- Data events logged for **all** S3 buckets / Lambda functions / DynamoDB tables
  when only a subset is security-relevant → every data event delivery is billed →
  narrow with advanced event selectors (by `resources.ARN`, `eventName`, or
  `readOnly`) → **HIGH** when data event volume is large.
- The **same data events delivered by multiple trails** → each delivery is billed
  separately (no free copy for data events) → consolidate → **HIGH**.

### 4.5 CloudTrail Lake spend
Ref: [Managing CloudTrail Lake costs](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-lake-manage-costs.html)

- An event data store's **pricing option** mismatched to its query pattern
  (one-time-query data ingested on the higher-priced flexible-retention option, or a
  frequently queried store on a suboptimal option) → **MEDIUM**.
- **Retention** far longer than the compliance requirement → storage waste →
  **MEDIUM**.
- A Lake store capturing the **same events already captured by a trail** with no
  distinct query need → duplicate ingestion → **MEDIUM**.

### 4.6 S3 destination hygiene
Ref: [S3 lifecycle management](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)

- The trail's destination S3 bucket has **no lifecycle policy** transitioning old
  logs to cheaper tiers (S3 Glacier/Deep Archive) or expiring them → storage grows
  unbounded → **LOW**.

### 4.7 Idle / stopped trails
- A trail with `IsLogging=false` still configured → confirm it is intentional; if
  abandoned, delete to reduce management overhead → **INFO** (no direct charge while
  stopped, but signals drift).

## Step 5: Generate Report

Generate a shareable Markdown report artifact.

Artifact naming: `cloudtrail-cost-optimization-<account-id>-<YYYY-MM-DD>.md`
Example: `cloudtrail-cost-optimization-123456789012-2026-09-24.md`

Structure:

### Report Header
```
# AWS CloudTrail Cost Optimization — <account-id>
Date: <YYYY-MM-DD> | Scope: <regions / organization> | Analysis window: <start> to <end>
```

### Executive Summary
- Estimated total monthly savings (sum of quantified opportunities) or "not quantified"
- Finding counts by severity
- Top 3 opportunities by estimated saving

### Trail & Lake Inventory
| Trail / EDS | Multi-Region | Org | Logging | Mgmt (R/W) | Data events | KMS/RDS excl. | S3 bucket |
|-------------|-------------|-----|---------|-----------|-------------|---------------|-----------|

### Cost Optimization Opportunities
| # | Opportunity | Severity | Current State | Recommendation | Est. Monthly Saving |
|---|-------------|----------|---------------|----------------|---------------------|

### Cost Attribution (if Cost Explorer available)
| Usage Type | 30-Day Cost | Share |
|------------|-------------|-------|

### Priority Matrix
| # | Opportunity | Severity | Effort | Est. Saving |
|---|-------------|----------|--------|-------------|

### Next Steps
- Immediate (HIGH — within 7 days)
- Short-term (MEDIUM — within 30 days)
- Long-term (LOW — within 90 days)

### Appendix — Reference Links
- [Managing CloudTrail trail costs](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-trail-manage-costs.html)
- [CloudTrail pricing](https://aws.amazon.com/cloudtrail/pricing/)
- [Remove duplicate CloudTrail events](https://repost.aws/knowledge-center/remove-duplicate-cloudtrail-events)
- [Optimize CloudTrail costs and maintain compliance](https://repost.aws/knowledge-center/optimize-cloudtrail-compliance)
- [Filtering data events with advanced event selectors](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/filtering-data-events.html)
- [Managing CloudTrail Lake costs](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-lake-manage-costs.html)

## Severity Definitions

| Severity | Definition | SLA |
|----------|------------|-----|
| CRITICAL | Runaway cost (e.g. multiple duplicated data-event trails) causing large ongoing overspend | Fix within 24–48 hours |
| HIGH | Clear, sizable recurring saving (duplicate management trails, broad data events) | Fix within 1 week |
| MEDIUM | Notable saving (read events, KMS/RDS noise, Lake tuning) | Plan within 30 days |
| LOW | Minor saving or hygiene (S3 lifecycle) | Address when convenient |
| INFO | Observation, no direct charge | N/A |

## Safety and Boundaries

- **Read-only.** The skill calls only `Describe*`, `Get*`, `List*` APIs. It never
  calls `CreateTrail`, `UpdateTrail`, `DeleteTrail`, `PutEventSelectors`,
  `StopLogging`, or any Lake mutation.
- **Compliance first.** Before recommending disabling a trail, dropping Read events,
  or excluding KMS/RDS events, state the audit/compliance tradeoff. Never recommend
  reducing the single authoritative security trail below the organization's logging
  requirements. When in doubt, recommend converting a duplicate to
  data-events-only rather than deleting it.
- **Proposed changes are suggestions.** Every recommendation is for a human to review
  and apply. Do not apply an event-selector or trail change you have not surfaced for
  review.

## Known Quirks

- The **first copy of management events per Region is free** — do not flag a single
  management trail per Region as a duplicate.
- **Data events have no free copy** — even a single data-event trail is billed; the
  opportunity there is scope, not de-duplication.
- CloudTrail does not expose reliable per-trail event counts; rely on Cost Explorer
  usage types and S3 bucket size as volume proxies, and clearly label estimates as
  approximate.
- Organizations trails appear as **shadow trails** in member accounts — set
  `includeShadowTrails=true` and do not double-count them as member-created duplicates.
