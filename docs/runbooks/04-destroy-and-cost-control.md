# 04 Destroy and cost control

Estimate: about 0.40 USD per hour per running environment (EKS control plane 0.10, 2x t3.large, NAT, ALB, cache). Budget `lab-monthly` is 100 USD with alerts at 50, 80 and 100 percent.

| Destroyed | Kept |
|---|---|
| `edge`: CloudFront, WAF, metrics Lambda, alarms, dashboard | `base`: ECR repositories, `lab-<env>-assets`, `lab-<env>-cnpg-backups`, SNS alert topics |
| `cluster`: VPC, NAT, EKS, nodes, Valkey, SSM parameters, Pod Identity roles | Accounts, state buckets, CI roles, budget |
| ALB and EBS volumes created by Kubernetes (removed by `scripts/teardown.sh`; in `prod` the volumes are retained, see step 7) | Postgres backups in S3 (restore with [05](05-database-restore.md)) |

Counters in Valkey are lost. Postgres history is restored from the S3 backup only if you follow runbook 05 on the next deploy.

1. Optional: take a fresh backup before destroying: `kubectl -n lab apply -f - <<< '{"apiVersion":"postgresql.cnpg.io/v1","kind":"Backup","metadata":{"name":"pre-destroy"},"spec":{"cluster":{"name":"lab-postgres"},"method":"barmanObjectStore"}}'`
2. Wait for it: `kubectl -n lab get backup pre-destroy` shows `completed`.
3. Start the destroy. `confirm` must equal the environment name.
   ```
   gh workflow run destroy.yml --repo <owner>/lab-infra-platform -f environment=<env> -f confirm=<env>
   ```
4. Approve the GitHub Environment if it requires a reviewer, then follow the run: `gh run watch --repo <owner>/lab-infra-platform`
5. The job runs `scripts/teardown.sh` (scale Argo CD to zero, delete Ingresses, CNPG clusters and PVCs, wait up to 600 s for the ALB and volumes to go), then destroys `edge`. It does not destroy `cluster`: the stack must reach the Kubernetes API, which is restricted to `api_allowed_cidrs`. Do step 6 for `cluster`.
6. Destroy `cluster` from your machine (it needs `terraform.tfvars` and access to the API). To run everything locally instead of the workflow:
   ```
   cd lab-infra-platform && scripts/teardown.sh <env> eu-central-1
   cd envs/<env>/edge && terraform init -backend-config="bucket=lab-tfstate-platform-<shared-id>" && terraform destroy
   cd ../cluster && terraform init -backend-config="bucket=lab-tfstate-platform-<shared-id>" && terraform destroy
   ```
7. If `teardown.sh` prints `TIMEOUT`, delete the listed ALB or volumes by hand, then rerun the destroy. The script is safe to run twice. In `prod` the gp3 StorageClass has `reclaimPolicy: Retain`, so the EBS volumes stay and `TIMEOUT` is the expected result. Delete the retained volumes by hand after checking the data is not needed (see [cost](../cost.md)), then rerun the destroy.

Confirm nothing is running (environment account, `--region eu-central-1`; every command returns empty or `0`; in `prod`, step 12 lists the retained volumes, see [cost](../cost.md)):

8. Clusters: `aws eks list-clusters --query clusters`
9. Load balancers: `aws elbv2 describe-load-balancers --query 'LoadBalancers[].LoadBalancerName'`
10. NAT gateways: `aws ec2 describe-nat-gateways --filter Name=state,Values=available,pending --query 'NatGateways[].NatGatewayId'`
11. Cache: `aws elasticache describe-replication-groups --query 'ReplicationGroups[].ReplicationGroupId'`
12. Volumes: `aws ec2 describe-volumes --filters Name=tag:kubernetes.io/cluster/lab-<env>,Values=owned --query 'length(Volumes)'`
13. Elastic IPs: `aws ec2 describe-addresses --query 'Addresses[].PublicIp'`
14. CloudFront: `aws cloudfront list-distributions --query "DistributionList.Items[].Comment"`
15. Spend this month: Billing console, Cost Explorer, in the management account, grouped by linked account.
