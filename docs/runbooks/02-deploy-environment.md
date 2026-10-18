# 02 Deploy an environment

Order: `base`, `cluster`, Argo CD sync, gitops values, `edge`. Replace `<env>` (`dev`, `test`, `prod`) and `<...>` values.
Apply `cluster` locally: it installs Argo CD through the Kubernetes API, which only `api_allowed_cidrs` may reach, and its variables come from `terraform.tfvars`. `base` can run locally as below, or through `apply.yml` once the `ALERT_EMAIL` repository variable is set. `edge` runs through `apply.yml`.
Needs: runbook 01 done, Terraform 1.10 or later, `kubectl`, admin credentials for the environment account (`export AWS_PROFILE=<profile>`).

1. In `lab-gitops`, replace `OWNER` and `ACCOUNT_ID`, then merge by pull request: `grep -n "OWNER\|ACCOUNT_ID" envs/<env>/values/*.yaml`
2. In `lab-infra-platform/envs/<env>/base`: `cp terraform.tfvars.example terraform.tfvars` and set `alert_emails`.
3. Create ECR, buckets and SNS topics: `terraform init -backend-config="bucket=lab-tfstate-platform-<shared-id>" && terraform apply`
4. Confirm the SNS subscription emails in `eu-central-1` and `us-east-1`. Alarms are silent until you do.
5. Build the first images: merge a change to `main` in `lab-backend`, `lab-web`, `lab-cluster`, then merge the `lab-gitops` pull requests they open ([03](03-release-and-promote.md)). For test and prod, use a tag.
6. In `envs/<env>/cluster`: `cp terraform.tfvars.example terraform.tfvars`. Set `kubernetes_version`, `api_allowed_cidrs` (your IP as `/32`), `admin_role_arns`, `argocd_chart_version`, `argocd_apps_chart_version`.
7. Create VPC, EKS, Valkey, Pod Identity roles, Argo CD and the root app: `terraform init -backend-config="bucket=lab-tfstate-platform-<shared-id>" && terraform apply`
8. Connect: `aws eks update-kubeconfig --name lab-<env> --region eu-central-1`
9. Watch Argo CD create the Applications (retries cover start-up ordering): `kubectl -n argocd get applications`
10. Read the values for gitops, in the same directory: `terraform output gitops_values`
11. Read the Kubernetes service IP: `kubectl get svc kubernetes -o jsonpath='{.spec.clusterIP}'`
12. In `lab-gitops/envs/<env>/values`, set `vpcId` and `clusterName` in `apps.yaml`; `vpcCidr` and `kubernetesServiceIp` (`<ip>/32`) in `cluster.yaml`; `vpcCidr` and `ingress.albSecurityGroupId` in `backend.yaml` and `web.yaml`. Replace `vpc-TODO` and `sg-TODO`.
13. Merge the change to `main` by pull request. Argo CD syncs it automatically.
14. Wait for the shared ALB (state `active`): `aws elbv2 describe-load-balancers --names lab-<env>-alb --region eu-central-1 --query 'LoadBalancers[0].State.Code' --output text`
15. Check workloads: `kubectl -n lab get pods`
16. Check the database: `kubectl -n lab get cluster lab-postgres`
17. Apply `edge` (CloudFront, WAF, Lambda, alarms, dashboard): `gh workflow run apply.yml --repo <owner>/lab-infra-platform -f environment=<env> -f stack=edge`
18. Approve the run if the GitHub Environment requires it: `gh run watch --repo <owner>/lab-infra-platform`
19. Get the domain: `aws cloudfront list-distributions --query "DistributionList.Items[?Comment=='lab-<env> dashboard'].DomainName" --output text`
20. Smoke test, health: `curl -s https://<domain>/api/health` returns `{"status":"ok"}`.
21. Smoke test, counters: `curl -s -c /tmp/lab.jar -b /tmp/lab.jar https://<domain>/api/visit` returns `"counters_available":true`.
22. Smoke test, page: `curl -s -o /dev/null -w '%{http_code}\n' https://<domain>/` prints `200`.
23. Smoke test, platform cards (after about 2 minutes): `curl -s https://<domain>/metrics/metrics.json`
24. Open `https://<domain>/` in a browser.

If a step fails, see [06 Troubleshooting](06-troubleshooting.md).
