# ADR-0004: In-cluster Postgres with CloudNativePG

Status: Accepted.

## Context
- The app needs a small relational store for daily history and an HLL snapshot.
- Cost matters; the lab also aims to show operating a stateful workload on Kubernetes.

## Decision
- CloudNativePG operator (Argo CD Application) runs `Cluster/lab-postgres`, Postgres 18: 1 instance in dev and test, 2 in prod across zones.
- Storage is an encrypted gp3 volume (`Retain` in prod).
- Base backups daily at 02:00 UTC plus continuous WAL to `lab-<env>-cnpg-backups` (base stack), 7-day retention, via Pod Identity.

## Consequences
- No RDS bill; the operator, upgrades and restores are our work.
- Dev and test have no standby: node loss means downtime until rescheduling.
- The operator is installed by Argo CD from the upstream chart (`lab-gitops/charts/apps/templates/upstream.yaml`); the `lab-postgres` cluster is `lab-gitops/charts/lab-postgres`.
- Backups outlive the cluster. Restore needs a new `serverName` per restore (procedure in `lab-gitops/charts/lab-postgres/README.md`).
- The `Cluster` has `Delete=false`; Argo CD never removes the database.

## Alternatives considered
- RDS or Aurora: managed and Multi-AZ, but a separate service to pay for; pricing not checked here.
