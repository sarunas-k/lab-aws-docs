# ADR-0003: GitHub configured with Terraform

Status: Accepted.

## Context
- Branch protection and deployment approvals are what gate the AWS pipelines.
- Click-configured settings drift and cannot be reviewed.

## Decision
- `lab-github` manages repositories, a `protect-main` ruleset, a `protect-release-tags` ruleset (`v*`) and GitHub Environments.
- `lab-bootstrap`, `lab-infra-org` and `lab-github` are `strict`: no bypass.
- Other repos let repository admins bypass, so a solo maintainer can merge.
- Secret values (`GITOPS_TOKEN`, tokens) are set in the GitHub UI and never enter Terraform state.

## Consequences
- Repositories already exist and must be imported before the first plan.
- The first apply uses a personal token; later changes go through the PR pipeline.
- `prevent_self_review` is off while there is one reviewer.
- Required check names must match the reusable-workflow job names (`<caller job> / <job>`). A mismatch blocks merges.
- Repositories are public.

## Alternatives considered
- Manual settings in the UI: unreviewed, not reproducible.
