# ADR-0010: Rebuild per environment from tags, with per-environment ECR

Status: Accepted.

## Context
- Each environment is a separate AWS account with its own ECR and CI role.
- Promoting an image across accounts needs replication or cross-account pull rights.

## Decision
- `app-release.yml` is called per environment: push to `main` builds for dev, tag `vX.Y.Z` for test, manual dispatch with a tag for prod.
- Each run rebuilds the same commit, runs Trivy (fails on CRITICAL before push), pushes to that account's ECR, checks the ECR scan, signs with cosign (keyless) and opens a PR in `lab-gitops` that changes `image.tag`.
- ECR: immutable tags, scan on push, keep the last 20 images.

## Consequences
- Accounts stay isolated; no cross-account registry access.
- The prod image is rebuilt, not the bytes tested in test. Digests can differ if base images move.
- One build per environment per release.
- Gap: signatures are created but nothing in the cluster verifies them.
- Gap: SonarCloud scan and post-deploy smoke test are not written. `OWNER` and `ACCOUNT_ID` are placeholders.

## Alternatives considered
- Build once and promote the digest by ECR replication: stronger provenance, more IAM and setup.
- One shared ECR in `lab-shared-services`: simpler, weaker isolation.
