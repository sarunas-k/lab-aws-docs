# Runbooks

| Runbook | Use it to |
|---|---|
| [01 Bootstrap from zero](01-bootstrap-from-zero.md) | Create the organization, accounts, CI roles, state buckets and GitHub configuration |
| [02 Deploy an environment](02-deploy-environment.md) | Apply `base`, `cluster`, `edge` and reach the CloudFront URL |
| [03 Release and promote](03-release-and-promote.md) | Move a change from `main` to dev, test and prod |
| [04 Destroy and cost control](04-destroy-and-cost-control.md) | Stop AWS charges for one environment and confirm nothing is left |
| [05 Database restore](05-database-restore.md) | Restore Postgres from S3 after a cluster rebuild |
| [06 Troubleshooting](06-troubleshooting.md) | Symptom, cause, check, fix |

Order for a new setup: 01, then 02 (dev first), then 03. Use 04 when finished. Use 05 after 04 when the data matters.

## Conventions

- `<env>` is `dev`, `test` or `prod`. Region is `eu-central-1` unless stated.
- Commands run from the repository named in the step. Use credentials for the account named in the step.
- Placeholders to replace before first use: `OWNER` (GitHub user or organization), `ACCOUNT_ID`, `vpc-TODO`, `sg-TODO`.
