# ADR-0014: CNPG in-tree Barman backups instead of the Barman Cloud plugin

Status: Accepted with a deadline. Must be revisited before upgrading the operator past 1.30.

## Context
- `barmanObjectStore` in the `Cluster` spec is deprecated since CNPG 1.26. Removal is planned for 1.31.0.
- The pinned operator is 1.30.1 (chart 0.29.1).
- The plugin needs cert-manager and its own Application, neither of which is in the lab.

## Decision
- Use the in-tree `barmanObjectStore` with `ScheduledBackup` method `barmanObjectStore`.
- Pin the operator version in `envs/<env>/values/apps.yaml`.
- Use the `system` image flavour, which includes the Barman tools.

## Consequences
- Fewer components today and one less certificate dependency.
- Upgrading `cloudnativePg` to a chart with operator 1.31 breaks backups unless the plugin migration is done first.
- Migration touches `Cluster`, `ScheduledBackup`, the restore values and adds cert-manager.
- Nothing but review prevents a chart version bump past 1.30.

## Alternatives considered
- Plugin now: the supported path, but it adds cert-manager and a second Application before the first backup works.
