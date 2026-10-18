# ADR-0016: Aggregator in the backend repository

Status: Accepted.

## Context
- The nightly aggregator reads Valkey keys written by the API and writes the Postgres tables the API reads (`daily_stats`, `hll_snapshots`).
- It lived in its own repository with its own image, so a change to the key layout or schema needed two repositories and two image tags.
- Both use the same libraries (`valkey`, `psycopg`).

## Decision
- The aggregator is the package `aggregator/` in `lab-backend`, with integration tests in `tests/test_aggregator.py`.
- The image is the same as the API; the CronJob overrides the command: `python -m aggregator.job`.
- `lab-gitops` deploys the CronJob from `charts/lab-backend`, using the same image repository and tag as the Deployment. The Postgres cluster stays in `charts/lab-postgres`.
- CI in `lab-backend` runs Valkey and Postgres service containers for the integration tests.

## Consequences
- The data contract and the code that uses it change in one pull request and ship under one tag.
- One image, one release pipeline and one repository fewer (`lab-cluster` is removed).
- The API image carries the aggregator code, and an API release redeploys the CronJob image too.
- The CronJob pods keep the label `app.kubernetes.io/name: lab-aggregator`, so NetworkPolicies still select them separately from the API.

## Alternatives considered
- Separate repository and image: independent release cadence, which the lab does not need.
