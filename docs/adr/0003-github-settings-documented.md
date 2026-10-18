# ADR-0003: GitHub settings documented, not managed with Terraform

Status: Accepted. Replaces the earlier decision to manage GitHub with a Terraform repository (`lab-github`).

## Context
- Branch protection and deployment approvals gate the AWS pipelines.
- Terraform for GitHub needed a personal token, imports of existing repositories and its own state, for eight repositories and a handful of settings.
- The OIDC trust policies in `lab-aws-infra-bootstrap` match GitHub Environment names, so those names must stay stable.

## Decision
- GitHub settings are configured by hand in the UI or with the `gh` CLI. [Repository settings](../repository-settings.md) lists every repository, ruleset, Environment, required check, secret and variable.
- `lab-aws-infra-bootstrap` and `lab-aws-infra-organization` have no admin bypass on `main`. Other repositories let admins bypass, so a solo maintainer can merge.
- Secret values are set in the GitHub UI and are not stored in code.
- The `lab-github` repository is removed.

## Consequences
- One repository and one state file fewer to operate.
- Settings can drift and a change is not reviewed as a pull request. The page is the reference to compare against.
- Required check names must match the reusable-workflow job names (`<caller job> / <job>`). A mismatch blocks merges.
- Environment names are part of the AWS trust and must not be renamed.

## Alternatives considered
- Terraform GitHub provider: reviewable and reproducible, but more to maintain than the settings justify here.
