# Architecture decision records

Each record states the context, the decision and its consequences, including known gaps.

| ADR | Decision | Repos |
|---|---|---|
| [0001](0001-multi-account-organization.md) | Multi-account organization with SCPs and a budget | lab-infra-org |
| [0002](0002-terraform-with-cloudformation-bootstrap.md) | Terraform for infrastructure, CloudFormation only for bootstrap | lab-bootstrap, lab-infra-platform |
| [0003](0003-github-managed-with-terraform.md) | GitHub configured with Terraform | lab-github |
| [0004](0004-in-cluster-postgres-cloudnativepg.md) | In-cluster Postgres via CloudNativePG | lab-gitops, lab-cluster |
| [0005](0005-elasticache-valkey-counters.md) | ElastiCache Valkey for counters and HyperLogLog | lab-infra-platform, lab-backend |
| [0006](0006-single-alb-path-routing.md) | One ALB with in-cluster path routing; Gateway API as migration path | lab-gitops, lab-infra-platform |
| [0007](0007-cloudfront-default-cert-http-origin-waf.md) | CloudFront default certificate, HTTP origin, header-checking WAF | lab-infra-platform |
| [0008](0008-no-route53-no-custom-domain.md) | No Route 53, no custom domain | lab-infra-platform |
| [0009](0009-get-api-visit-for-origin-failover.md) | `GET /api/visit` so CloudFront origin failover could work | lab-backend, lab-infra-platform |
| [0010](0010-rebuild-per-environment-and-ecr.md) | Rebuild per environment from tags; per-environment ECR | lab-github-workflows |
| [0011](0011-helm-in-gitops-sync-waves-and-retries.md) | Helm only in gitops; sync waves relying on retries | lab-gitops, lab-infra-platform |
| [0012](0012-secrets-ssm-and-external-secrets.md) | Secrets via SSM Parameter Store and External Secrets | lab-infra-platform, lab-gitops |
| [0013](0013-vpc-cni-network-policy-standard-mode.md) | VPC CNI network policy left in standard mode | lab-infra-platform, lab-gitops |
| [0014](0014-cnpg-barman-in-tree-vs-plugin.md) | CNPG in-tree Barman backups instead of the Barman Cloud plugin | lab-gitops |
| [0015](0015-argocd-bootstrap-script-and-local-destroy.md) | Argo CD bootstrapped by script, `cluster` stack AWS-only, destroy local and ordered | lab-infra-platform, lab-gitops |

Status values: Accepted, Accepted with a deadline, Superseded. Add new records with the next number; do not edit the decision of an accepted record, supersede it.
