# Config Cost Optimization — AWS DevOps Agent Skill

A read-only AWS Config cost optimization skill for [AWS DevOps Agent](https://docs.aws.amazon.com/devopsagent/latest/userguide/about-aws-devops-agent.html). It identifies, quantifies, and prioritizes AWS Config cost-reduction opportunities aligned with [Optimize AWS Config costs](https://repost.aws/knowledge-center/optimize-aws-config) and [AWS Config pricing](https://aws.amazon.com/config/pricing/), and generates a shareable report artifact.

## Purpose

AWS Config bills primarily per configuration item (CI) recorded, plus per rule and conformance-pack evaluation and S3 storage. Costs climb quietly when recorders capture all supported resource types, record global resources in every Region, run continuous recording on high-churn resources that would be cheaper on daily, or evaluate redundant rules. This skill teaches the agent the Config billing model — including the counterintuitive continuous-vs-daily tradeoff — and how to trace spend to specific resource types, recorders, and rules, then recommends compliance-aware reductions.

## Key Capabilities

- Inventories configuration recorders, recording mode (continuous/daily and per-resource-type overrides), delivery channels, rules, conformance packs, and aggregators via read-only APIs
- Flags high-churn resource types recorded continuously that would be cheaper on daily recording — and, conversely, avoids blanket daily recommendations for low-churn security-critical types
- Detects over-broad `allSupported` recording and duplicate global-resource recording across Regions (the classic silent CI multiplier)
- Identifies configuration-item drivers using Cost Explorer, Athena-based CI-driver analysis, or resource-count signals
- Flags redundant/unnecessary rules, standalone-vs-conformance-pack overlap, and recorders running with no downstream consumer
- Reviews the delivery S3 bucket for lifecycle hygiene
- Produces a severity-ranked report artifact with a CI-driver table, priority matrix, and next steps

## Prerequisites

### 1. An AWS DevOps Agent Space with the target AWS account configured as a cloud source

### 2. IAM permissions for the DevOps Agent's primary cloud-source role

Read-only Config, CloudWatch, S3, Organizations, and (recommended) Cost Explorer access:

- `config:DescribeConfigurationRecorders`, `config:DescribeConfigurationRecorderStatus`, `config:DescribeDeliveryChannels`
- `config:DescribeConfigRules`, `config:DescribeConformancePacks`, `config:DescribeConfigurationAggregators`
- `config:GetDiscoveredResourceCounts`
- `cloudwatch:GetMetricData`, `cloudwatch:GetMetricStatistics`
- `s3:ListBucket`, `s3:GetBucketLifecycleConfiguration` (delivery bucket hygiene)
- `organizations:DescribeOrganization`, `organizations:ListAccounts` (organization scope)
- `ce:GetCostAndUsage` (recommended — the most direct dollar signal for sizing opportunities)
- `athena:StartQueryExecution`, `athena:GetQueryResults` (optional — authoritative CI-driver attribution over Config S3 data)

Most read APIs are covered by the AWS managed `AIDevOpsAgentAccessPolicy`. `ce:GetCostAndUsage` and the S3 lifecycle read may need to be added — see [`cloudformation/devops-agent-skill-policies.yaml`](../../cloudformation/devops-agent-skill-policies.yaml) (`EnableConfigCostOptimization`).

The skill operates entirely in **read-only** mode — it never calls `PutConfigurationRecorder`, `StopConfigurationRecorder`, `PutConfigRule`, `DeleteConfigRule`, or any conformance-pack/delivery-channel mutation.

## Limitations

- The most accurate CI-driver attribution requires Athena over the Config S3 data; without it, the skill uses resource-count and known-high-churn signals and labels estimates as approximate.
- Without Cost Explorer access, findings are still reported but dollar impact is labeled "not quantified".
- In Control Tower / Organizations environments, recorder settings may be centrally managed and reset on account provisioning — recommendations may need to be applied via the landing-zone customization path.
- Compliance requirements (which resource types must be recorded, and at what frequency) are the user's to confirm; the skill surfaces the tradeoff but does not decide it.

## Agent Types

- **On-demand (Chat tasks)** — "why is my Config bill so high", "should I use daily or continuous recording".
- **Evaluation** — proactive cost-optimization recommendations.

Select **Generic** (All agents) if you want the skill available to all agent types — required if you use it with the `unified-security-cost-optimizer` custom agent.

## Uploading to AWS DevOps Agent

> Reference: [Uploading a skill](https://docs.aws.amazon.com/devopsagent/latest/userguide/about-aws-devops-agent-devops-agent-skills.html#uploading-a-skill)

Build the zip from **inside** the skill directory so `SKILL.md` sits at the archive root, excluding development-only files:

```bash
cd skills/config-cost-optimization
zip -r ../config-cost-optimization.zip . \
  -x 'README.md' 'CHANGELOG.md' '.skilleval.yaml' '.skilleval.yml' 'evals/*'
```

Then in the Operator Web App: **Skills → Add skill → Upload skill**, drag in the zip, select agent types (**On-demand** and **Evaluation**, or **Generic** for the custom agent), review validation, and **Upload**.

## How to Use This Skill

In DevOps Agent Chat:

- *"Review my AWS Config costs for account 123456789012 across all regions."*
- *"Why did my Config bill go up this month?"*
- *"Which resource types are driving my configuration item count?"*
- *"Should I switch to daily Config recording?"*
- *"Am I recording global resources in more than one region?"*

The agent will inventory the Config setup, attribute CI drivers with any available cost signal, and generate `config-cost-optimization-<account-id>-<YYYY-MM-DD>.md`.

## Non-production disclaimer

> ⚠️ This skill is sample code, not intended for production use without additional review and testing. Validate in a non-production environment first. Recommendations are derived from observed configuration and usage — review the compliance impact and narrow them before applying, and never apply a Config change you have not read.
