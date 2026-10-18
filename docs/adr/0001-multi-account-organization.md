# ADR-0001: Multi-account AWS organization

Status: Accepted.

## Context
- Three environments need separate blast radius, state access and cost visibility.
- The total budget is about 100 USD, so extra accounts must not add fixed cost.

## Decision
- One organization with five member accounts: `lab-log-archive` (Security OU), `lab-shared-services` (Infrastructure OU), `lab-dev`, `lab-test`, `lab-prod` (Workloads OU).
- Four SCPs on the root: deny leave-organization, deny unapproved regions (eu-central-1, eu-west-1, us-east-1), deny root user, protect CloudTrail.
- Monthly budget of 100 USD with alerts at 50, 80 and 100 percent of actual spend.

## Consequences
- Each environment has its own ECR, state prefix and CI roles. The state bucket policy limits an account to `envs/<env>/`.
- Accounts are `prevent_destroy`; they cannot be recreated quickly.
- SCPs do not apply to the management account.
- Gap: organization CloudTrail is not implemented, so `lab-log-archive` holds no resources and `lab-protect-cloudtrail` protects nothing yet. GuardDuty, Security Hub and Config are also absent.

## Alternatives considered
- One account with namespaces and tags: weaker isolation.
- AWS Control Tower: not used; the guardrails here are plain Terraform.
