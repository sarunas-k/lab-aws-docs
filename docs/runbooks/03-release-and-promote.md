# 03 Release and promote

| Trigger (in `lab-aws-backend`, `lab-aws-web`) | Target | Image tag |
|---|---|---|
| Push to `main` | dev | 12-character commit SHA |
| Tag `vX.Y.Z` | test | the tag |
| Manual dispatch with a tag | prod | the tag |

Each target rebuilds the same commit in its own account and ECR. The `release` job runs in the GitHub Environment of the target and waits for approval where the Environment has a required reviewer (`prod` in the app repositories, see [repository settings](../repository-settings.md)). Nothing writes to a cluster directly: the bot opens a pull request in `lab-aws-gitops`.

1. Open a pull request in the app repository. Required checks: `test` and `image / image`.
   ```
   gh pr create --repo <owner>/lab-aws-backend
   ```
2. Merge it. The `release` workflow starts for dev: build, Trivy (fails on CRITICAL), push to ECR, ECR scan, cosign signature.
   ```
   gh run watch --repo <owner>/lab-aws-backend
   ```
3. Find the bot pull request `Bump lab-backend dev to <sha>` in `lab-aws-gitops`. It changes one `tag:` line in `envs/dev/values/backend.yaml`.
   ```
   gh pr list --repo <owner>/lab-aws-gitops
   ```
4. Review and merge it. Argo CD syncs dev automatically.
   ```
   gh pr merge <number> --repo <owner>/lab-aws-gitops --squash
   ```
5. Check the rollout: `kubectl -n lab rollout status deployment/lab-backend`. Use `deployment/lab-web` for the web app.
6. Check the page through the dev CloudFront URL: `curl -s https://<domain>/api/health`
7. Promote to test: create a release tag on `main`.
   ```
   gh release create v1.0.0 --repo <owner>/lab-aws-backend --target main --generate-notes
   ```
8. Approve the `test` environment if asked, then merge the `Bump lab-backend test to v1.0.0` pull request in `lab-aws-gitops`.
9. Promote to prod: dispatch from the tag, so the same commit is built. Approve the `prod` environment when asked.
   ```
   gh workflow run release.yml --repo <owner>/lab-aws-backend --ref v1.0.0 -f tag=v1.0.0
   ```
10. Merge the `Bump lab-backend prod to v1.0.0` pull request in `lab-aws-gitops`.
11. Roll back: revert the bump pull request in `lab-aws-gitops` and merge it. Argo CD returns to the previous tag. ECR tags are immutable, so never reuse a tag.

Notes:
- `lab-aws-web` also syncs `src/images` to `s3://lab-dev-assets/images` on pushes to `main`.
- `lab-backend` also runs the visit aggregator CronJob from the same image, so one bump updates both.
- Prod rebuilds from the tag; the bytes can differ from test if base images changed.
- The bot opens a pull request only. If the tag is already current, no pull request is opened.
