# Cost

All figures are estimates in USD. Hourly prices other than the EKS control plane are approximate; confirm them on the AWS pricing pages for `eu-central-1`.

## Assumptions

- Budget: about 100 USD per month for the whole organization (`lab-infra-org/budget.tf`, variable `budget_limit_usd`).
- Environments run only while in use. `cluster` and `edge` are destroyed after each session; `base` stays.
- Dev and test use Spot nodes, so the baseline is an upper bound for them. Prod uses On-Demand nodes.

## Cost drivers

| Component | Source | Basis | USD/hour (approx.) |
|---|---|---|---|
| EKS control plane | `modules/cluster/eks.tf` | per cluster | 0.10 |
| 2x `t3.large` nodes | `node_instance_types`, `node_desired_size` | per instance | 0.20 |
| NAT gateway, one per VPC | `single_nat_gateway = true` | hour + per GB | 0.05 |
| Shared ALB | created by the Load Balancer Controller | hour + LCU | 0.03 |
| Valkey `cache.t4g.micro`, one node | `elasticache.tf` | per node | 0.02 |
| **Baseline** | | | **0.40** |

Billed while the stacks exist, not in the baseline: EBS gp3 (30 GiB per node, Postgres volumes of 5 GiB in dev/test and 10 GiB per instance in prod, two instances), two WAF web ACLs, CloudFront (`PriceClass_100`), CloudWatch logs (7-day retention), public IPv4 addresses on the NAT gateway and ALB, and the metrics Lambda (7 metrics per minute through `GetMetricData`, well under 0.01 USD/hour).

## Estimate and planned sessions

Formula: `cost = hours_running x 0.40` per environment.

| Plan item | Sessions x hours | Env-hours | Estimate (USD) |
|---|---|---|---|
| dev iteration | 10 x 6 | 60 | 24.00 |
| test checks | 4 x 4 | 16 | 6.40 |
| prod checks | 2 x 4 | 8 | 3.20 |
| **Total** | | **84** | **33.60** |

The budget covers about 250 environment-hours (100 / 0.40). Left running for a month (730 hours), one environment would cost about 292 USD.

## Guardrails

- AWS Budget `lab-monthly`: email at 50, 80 and 100 percent of actual spend. No forecast alert and no automatic shutdown.
- `destroy.yml` in `lab-infra-platform`: runs `scripts/teardown.sh`, then destroys `edge`, then `cluster`. The environment name must be typed again to confirm.
- Cost settings in code: one NAT gateway, one Valkey node without snapshots, 7-day log retention, ECR keeps 20 images, old S3 versions expire.
- SCP `lab-deny-unapproved-regions` blocks resources outside `eu-central-1`, `eu-west-1` and `us-east-1`.

Check for leftovers after a destroy:

```bash
aws eks list-clusters
aws elbv2 describe-load-balancers --query 'LoadBalancers[].LoadBalancerName'
aws ec2 describe-nat-gateways --filter Name=state,Values=available
aws ec2 describe-volumes --filters Name=status,Values=available
```

## What keeps costing after destroy

- `base` stack: ECR images, assets bucket, CNPG backup bucket, SNS topics. Storage only.
- Terraform state buckets (old versions expire after 90 days).
- Prod EBS volumes: the prod StorageClass uses `reclaimPolicy: Retain`. Delete them by hand if the data is not needed.
- Anything `teardown.sh` could not remove. It stops after 600 seconds and prints what is left.

## Actual spend

Cost Explorer lags by up to a day. Activating `Project`, `Env` and `Stack` (set by provider `default_tags`) as cost allocation tags allows filtering by environment.

| Month | Env | Hours up | Estimate (USD) | Actual (USD) | Note |
|---|---|---|---|---|---|
| | | | | | |
