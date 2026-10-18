# 05 Database restore

Use after the `cluster` stack was destroyed and recreated. Backups survive because the bucket `lab-<env>-cnpg-backups` is in the `base` stack. Full detail: `lab-gitops/charts/lab-cluster/README.md`.

| Setting (`lab-gitops/envs/<env>/values/cluster.yaml`) | Fresh install | First restore | Second restore |
|---|---|---|---|
| `bootstrap.mode` | `initdb` | `recovery` | `recovery` |
| `bootstrap.recovery.sourceServerName` | `lab-postgres` | `lab-postgres` | `lab-postgres-r1` |
| `backup.serverName` | `lab-postgres` | `lab-postgres-r1` | `lab-postgres-r2` |

In `recovery` mode, `backup.serverName` must differ from `sourceServerName`, or the chart refuses to render. The restored cluster writes WAL to a new folder because CNPG refuses a folder that holds another cluster's WAL ("Expected empty archive").

Do this before step 8 of [02](02-deploy-environment.md), i.e. before Argo CD is bootstrapped. Argo CD syncs `lab-cluster` within minutes of the bootstrap.

1. Confirm backups exist: `aws s3 ls s3://lab-<env>-cnpg-backups/<sourceServerName>/ --recursive | head`. The folder is the `sourceServerName` from the table (`lab-postgres`, or `lab-postgres-r1` for the second restore).
2. Edit `envs/<env>/values/cluster.yaml` as in the table (first restore column).
3. Merge to `main` by pull request.
4. Create the cluster stack and bootstrap Argo CD: [02](02-deploy-environment.md), steps 6 to 8.
5. Watch the restore: `kubectl -n lab get pods -w`
6. Check status: `kubectl -n lab get cluster lab-postgres` shows `Cluster in healthy state`.
7. Check a new backup is written to the new folder: `kubectl -n lab get backup`
8. Check data: `kubectl -n lab exec lab-postgres-1 -c postgres -- psql -U postgres -d lab -c 'select count(*) from daily_stats'`
9. Restart the backend so it reads the new `lab-postgres-app` Secret: `kubectl -n lab rollout restart deployment/lab-backend`
10. Leave the settings in place. `bootstrap` applies only when the Cluster is first created, and `backup.serverName` must keep pointing at the folder in use.

Started empty by mistake (Cluster in `initdb`, WAL archiving fails with "Expected empty archive"):

11. Delete the Cluster and PVCs: `kubectl -n lab delete cluster lab-postgres; kubectl -n lab delete pvc -l cnpg.io/cluster=lab-postgres`
12. Set the recovery values, merge, and let Argo CD recreate the Cluster. The Cluster has `Delete=false`, so Argo CD never removes it for you.

Notes:
- The app user password is not restored. CNPG creates a new Secret.
- Point-in-time recovery: add `recoveryTarget` to the `recovery` block in `templates/cluster.yaml`.
- Never use `recovery` on an environment with no backups: the Cluster never becomes ready.
- Valkey counters are not in this backup. The backend restores `uniques:total` from the `hll_snapshots` table at start; `visits:total` restarts at zero.
