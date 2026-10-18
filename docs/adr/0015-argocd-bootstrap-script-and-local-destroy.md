# ADR-0015: Argo CD bootstrap script, AWS-only cluster stack, local ordered destroy

Status: Accepted.

## Context
- The Terraform `helm` and `kubernetes` providers installed Argo CD, so the `cluster` stack needed the Kubernetes API.
- The API is limited to `api_allowed_cidrs`, so a hosted CI runner could neither apply nor destroy `cluster`.
- Destroying `edge` after the ALB was gone fails, because `edge` reads the ALB with `data "aws_lb"`.
- Volume cleanup looked for the tag `kubernetes.io/cluster/lab-<env>=owned`, which the EBS CSI driver does not set by default.

## Decision
- `cluster` is AWS-only: VPC, EKS, Valkey, Pod Identity roles. `apply.yml` can apply it.
- `scripts/bootstrap-argocd.sh <env> <owner>` installs Argo CD and the root app-of-apps once per cluster; afterwards Argo CD syncs the platform from `lab-aws-gitops` and is upgraded by rerunning the script.
- Destroy is local only (`scripts/destroy.sh <env> <region>`), in the order `edge`, `scripts/teardown.sh`, `cluster`. The environment name is typed to confirm. There is no destroy workflow.
- The gp3 StorageClass sets `tagSpecification_1: "lab-cluster=lab-<env>"`; `teardown.sh` finds volumes by that tag.
- The `cost-guard` workflow runs every 6 hours and fails when a cluster or NAT gateway is running.

## Consequences
- CI never needs Kubernetes API access.
- A destroy is a deliberate local action; the cost-guard failure notification is the reminder.
- Argo CD upgrades: edit the chart version in the bootstrap script and rerun it.
- Volumes created before the tag was added are not found by `teardown.sh`.
- In `prod` the volumes stay (`Retain`) and are deleted by hand.

Alternative: a self-hosted runner inside the VPC keeps CI destroy but adds a runner to operate.
