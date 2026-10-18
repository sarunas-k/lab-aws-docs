# ADR-0011: Helm only in lab-aws-gitops; sync waves backed by retries

Status: Accepted.

## Context
- Terraform and Argo CD must not both own in-cluster resources.
- Argo CD syncs child Applications independently, and has no default health check for `Application`.

## Decision
- `scripts/bootstrap-argocd.sh` installs Argo CD and a root app-of-apps (`charts/apps`) once per cluster; Terraform does not (ADR 0015). Pod Identity roles for add-ons stay in Terraform.
- All in-cluster resources are Helm charts in `lab-aws-gitops`: upstream add-ons with pinned versions, and four own charts.
- Order: wave 0 add-ons, 1 `lab-platform-config`, 2 `lab-postgres`, 3 `lab-backend` and `lab-web`.
- The bootstrap script adds an `Application` health check so waves can gate.
- Every Application also retries: 10 attempts, 10 s doubling to a 5 min cap (about 20 minutes), with `SkipDryRunOnMissingResource`.

## Consequences
- Retries are the fallback for missing CRDs, webhooks or the `lab` namespace on a cold cluster.
- Argo CD does not retry the same commit after the budget is spent; sync by hand.
- All environments track `main`, and the charts are shared, so a chart change reaches every environment. Pinning prod to a tag is not done.
- Apps use the `default` AppProject.

## Alternatives considered
- Terraform `helm_release` for add-ons: simple ordering, but upgrades go through Terraform state.
