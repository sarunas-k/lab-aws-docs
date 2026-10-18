# 04 Destroy and cost control

Estimate: about 0.40 USD per hour per running environment (EKS control plane 0.10, 2x t3.large, NAT, ALB, cache). Budget `lab-monthly` is 100 USD with alerts at 50, 80 and 100 percent.

| Destroyed | Kept |
|---|---|
| `edge`: CloudFront, WAF, metrics Lambda, alarms, dashboard | `base`: ECR repositories, `lab-<env>-assets`, `lab-<env>-cnpg-backups`, SNS alert topics |
| `cluster`: VPC, NAT, EKS, nodes, Valkey, SSM parameters, Pod Identity roles | Accounts, state buckets, CI roles, budget |
| ALB and EBS volumes created by Kubernetes (removed by `scripts/teardown.sh`; in `prod` the volumes are retained, see step 5) | Postgres backups in S3 (restore with [05](05-database-restore.md)) |

Counters in Valkey are lost. Postgres history is restored from the S3 backup only if you follow runbook 05 on the next deploy.

1. Optional: take a fresh backup before destroying: `kubectl -n lab apply -f - <<< '{"apiVersion":"postgresql.cnpg.io/v1","kind":"Backup","metadata":{"name":"pre-destroy"},"spec":{"cluster":{"name":"lab-postgres"},"method":"barmanObjectStore"}}'`
2. Wait for it: `kubectl -n lab get backup pre-destroy` shows `completed`.
3. Destroy locally, in `lab-aws-infra-environments`. Credentials must be an IAM Identity Center (SSO) role in the environment account, which the state bucket policy admits; `EXPECTED_ACCOUNT_ID` aborts the run on a mismatch and is required for `prod`. The terminal must be interactive. The operator needs Kubernetes API access: the CIDR in `api_allowed_cidrs`, and the operator role in `ADMIN_ROLE_ARNS` (or the one that created the cluster).
   ```
   export PLATFORM_STATE_BUCKET=lab-tfstate-platform-<shared-id> EXPECTED_ACCOUNT_ID=<12-digit-id>
   scripts/destroy.sh <env> eu-central-1
   ```
4. Type the environment name when asked. The script destroys `edge` first, while the ALB still exists, then runs `scripts/teardown.sh`, then destroys `cluster`. `base` is never destroyed. There is no destroy workflow. The script checks the Kubernetes API first, and sets placeholder values for the required `cluster` variables, which a destroy does not use.
5. `teardown.sh` scales Argo CD to zero, deletes Ingresses, CNPG clusters and PVCs, and waits up to 600 s for the ALB and for volumes tagged `lab-cluster=lab-<env>` to go. If `edge` is still in state but the ALB is already gone, its destroy fails at the ALB lookup: run `terraform destroy -refresh=false` in `envs/<env>/edge`, then rerun the script. If it prints `TIMEOUT`, delete the listed ALB or volumes by hand and rerun; the script is safe to run twice. In `prod` the gp3 StorageClass has `reclaimPolicy: Retain`, so `teardown.sh` lists the retained volumes and exits 0. Delete them by hand after checking the data is not needed (see [cost](../cost.md)).
6. The `cost-guard` workflow in `lab-aws-infra-environments` runs every 6 hours (and on manual dispatch). It fails, which sends a GitHub notification, when an environment has an EKS cluster or an available NAT gateway, and prints the estimated hourly cost (0.40 USD per environment). Run it now: `gh workflow run cost-guard.yml --repo <owner>/lab-aws-infra-environments`

Confirm nothing is running (environment account, `--region eu-central-1`; every command returns empty or `0`; in `prod`, step 11 lists the retained volumes, see [cost](../cost.md)):

7. Clusters: `aws eks list-clusters --query clusters`
8. Load balancers: `aws elbv2 describe-load-balancers --query 'LoadBalancers[].LoadBalancerName'`
9. NAT gateways: `aws ec2 describe-nat-gateways --filter Name=state,Values=available,pending --query 'NatGateways[].NatGatewayId'`
10. Cache: `aws elasticache describe-replication-groups --query 'ReplicationGroups[].ReplicationGroupId'`
11. Volumes: `aws ec2 describe-volumes --filters Name=tag:lab-cluster,Values=lab-<env> --query 'length(Volumes)'`
12. Elastic IPs: `aws ec2 describe-addresses --query 'Addresses[].PublicIp'`
13. CloudFront: `aws cloudfront list-distributions --query "DistributionList.Items[].Comment"`
14. Spend this month: Billing console, Cost Explorer, in the management account, grouped by linked account.
