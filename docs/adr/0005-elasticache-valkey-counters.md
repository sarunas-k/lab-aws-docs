# ADR-0005: ElastiCache Valkey for counters and HyperLogLog

Status: Accepted.

## Context
- Every page load increments counters and records a visitor id.
- Writing each visit to Postgres adds per-request database load for a counter.

## Decision
- Valkey 8.0 on one `cache.t4g.micro` node: `INCR` for `visits:*`, `PFADD`/`PFCOUNT` (HyperLogLog, about 1 percent error) for `uniques:*`.
- TLS and an AUTH token; no replica, no Multi-AZ, no snapshots.
- The nightly `lab-aggregator` (00:05 UTC) writes the previous day to `daily_stats` and a raw snapshot of `uniques:total` to `hll_snapshots`.
- The backend restores `uniques:total` at startup if the key is missing.

## Consequences
- Node loss resets today's counters. The all-time unique count loses at most one day.
- Gap: `visits:total` is not snapshotted in the code, so the all-time visit total resets after a cache loss.
- Daily keys expire after 3 days; history lives only in Postgres.
- The AUTH token is stored in Terraform state (see ADR-0012).
- If Valkey is down, `/api/visit` returns 503 with history only.

## Alternatives considered
- Postgres-only counters: simpler, but a write per request.
- Multi-AZ cache with replica: higher cost, not needed for a lab.
