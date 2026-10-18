# Security

## Account model

One AWS account per environment (`lab-infra-org`), so the blast radius is the account. Accounts have `prevent_destroy`. OUs: Security (`lab-log-archive`, no resources defined), Infrastructure (`lab-shared-services`, platform state bucket), Workloads (`lab-dev`, `lab-test`, `lab-prod`).

SCPs from `lab-infra-org/scps.tf`, attached to the root. They never apply to the management account.

| Policy | Effect |
|---|---|
| `lab-deny-leave-org` | Deny `organizations:LeaveOrganization`. |
| `lab-deny-unapproved-regions` | Deny all outside `eu-central-1`, `eu-west-1`, `us-east-1`, except listed global services (IAM, CloudFront, WAF, STS, Budgets, Organizations...). |
| `lab-deny-root-user` | Deny all actions for `arn:aws:iam::*:root`. |
| `lab-protect-cloudtrail` | Deny `cloudtrail:StopLogging` and `cloudtrail:DeleteTrail`. |

## CI identity: GitHub OIDC

No AWS keys are stored in GitHub. Roles come from `lab-bootstrap` and are assumed only when the token `sub` (`repo:<owner>/...`) and `aud = sts.amazonaws.com` match.

| Role | Account | Trusted `sub` suffix | Permissions |
|---|---|---|---|
| `lab-github-org-plan` | management | `lab-infra-org:pull_request` | ReadOnlyAccess, state read, lock file |
| `lab-github-org-apply` | management | `lab-infra-org:environment:org-apply` | Organizations, SSO, CloudTrail, Budgets, Account, state bucket |
| `lab-github-platform-plan` | dev, test, prod | `lab-infra-platform:pull_request` | ReadOnlyAccess, state read and lock under `envs/<env>/` |
| `lab-github-platform-apply` | dev, test, prod | `lab-infra-platform:environment:<env>` | AdministratorAccess, 2 h session |
| `lab-github-ecr-push` | dev, test, prod | `lab-backend`, `lab-web`, `lab-cluster`: `environment:<env>` | ECR push to `lab-*` |
| `lab-github-assets-sync` | dev, test, prod | `lab-web:environment:<env>` | Read, write, delete on `lab-<env>-assets/images/*` |

State isolation: the shared bucket policy admits each account only under `envs/<env>/`, for `lab-github-platform-*` roles in the organization.

Repository controls (`lab-github`):
- `protect-main`: pull request, 1 approval, code owner review, resolved threads, linear history, strict checks, no force-push. No admin bypass on `lab-bootstrap`, `lab-infra-org`, `lab-github`. `v*` tags are immutable.
- Environments accept deployments from `main` and `v*` only. Secret scanning with push protection and Dependabot alerts are on.

## Secrets
| Secret | Stored in | Delivered by |
|---|---|---|
| Valkey AUTH token | SSM SecureString `/lab/<env>/valkey/auth-token` | External Secrets (Pod Identity, scoped to `/lab/<env>/*`, `lab` namespace only) |
| Postgres app password | Secret `lab-postgres-app` (CloudNativePG) | `secretKeyRef` |
| `X-Origin-Verify` value | Terraform `random_password` | CloudFront header, regional WAF rule |
| `GITOPS_TOKEN` | GitHub secret | `app-release.yml` |

The Valkey token and origin header value are also in Terraform state (private, encrypted bucket). Kubernetes Secrets use EKS KMS envelope encryption.

## Supply chain (`lab-github-workflows`)
| Stage | Control |
|---|---|
| Pull request | hadolint, build, Trivy fails on CRITICAL (unfixed ignored); fmt, validate, tflint, Checkov, cfn-lint |
| Release | Trivy before push, ECR scan after push (fails on CRITICAL), immutable tags, keyless cosign signature by digest |
| Deploy | The bot opens a pull request changing one `tag:` line; branch rules apply |

## Network and pod hardening
- Edge: CloudFront WAF (2000 requests per IP per 5 minutes, AWS common rules). ALB security group admits the CloudFront prefix list only; regional WAF blocks requests without `X-Origin-Verify`.
- Nodes: IMDSv2 only, hop limit 1, encrypted root volumes. EKS API public access limited to `api_allowed_cidrs`.
- Namespace `lab`: Pod Security `restricted` enforced; default-deny ingress and egress, DNS rule, per-workload allow rules.
- Application pods (backend, web, aggregator): non-root, seccomp `RuntimeDefault`, read-only root filesystem, capabilities dropped, no privilege escalation, no service account token. Postgres pods are managed by CloudNativePG and need API access.
- Valkey: nodes-only security group, TLS, encryption at rest, AUTH token. Buckets: private, versioned, TLS-only.

## Known gaps and next steps

| Gap | Next step |
|---|---|
| No organization CloudTrail, GuardDuty, Security Hub or Config; `lab-log-archive` is empty | Add a trail and log-archive bucket, then the detection services |
| `lab-github-platform-apply` has AdministratorAccess; cosign signatures are not verified in the cluster | Scope the policy per stack; add an admission policy |
| Actions pinned to tags, reusable workflows to `@main` | Pin to commit SHAs and release tags |
| No SonarCloud scan; CloudFront to ALB is HTTP | Add the scan; HTTPS needs a domain and certificate |
| No network policy outside `lab`; `lab` ingress admits the whole VPC CIDR; Postgres egress admits HTTPS to any non-VPC address | Per-namespace policies, ALB subnet CIDRs, S3 gateway endpoint with FQDN-capable CNI |
| `require_code_owner_review` is on but no `CODEOWNERS` file exists; platform `ci.yml` sets no plan role (TODO) | Add `CODEOWNERS`; set per-environment `plan-role-arn` |
| Argo CD uses `server.insecure`, no SSO; no secret rotation automation | Keep Argo CD unexposed; rotate by apply and pod restart |
| Disaster recovery: not built | See [architecture](architecture.md) |
