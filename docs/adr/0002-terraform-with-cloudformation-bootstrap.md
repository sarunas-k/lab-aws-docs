# ADR-0002: Terraform for infrastructure, CloudFormation only for bootstrap

Status: Accepted.

## Context
- Terraform needs a state bucket and CI credentials before it can run.
- Every workload account needs a GitHub OIDC provider and CI roles before any pipeline can apply.

## Decision
- `lab-bootstrap` stage 1 (CloudFormation): state bucket, OIDC provider and the org plan/apply roles in the management account.
- Stage 2 (service-managed StackSets): per-account OIDC provider and CI roles, and the platform state bucket.
- Everything else is Terraform, with an S3 backend and `use_lockfile` (no DynamoDB table).

## Consequences
- Two tools, but CloudFormation is limited to resources Terraform cannot create for itself (state bucket, OIDC providers, CI roles).
- Stage 1 and the StackSet commands are manual CLI steps with an admin login.
- Org and platform apply roles are broad (platform: AdministratorAccess per single-purpose account). Controls: OIDC subject limited to a GitHub Environment, approval gate, SCPs.
- Plan roles are read-only and trusted only from `pull_request` runs.
- `cfn-ci.yml` runs cfn-lint and Checkov on templates.

## Alternatives considered
- CloudFormation for everything: rejected; Terraform is used for the large stacks.
- Manual console bootstrap: not repeatable, nothing to review.
