# Repository settings

GitHub settings are configured by hand, with the GitHub UI or the `gh` CLI. They are not managed with Terraform ([ADR 0003](adr/0003-github-settings-documented.md)). Replace `<owner>` with your GitHub user or organization. Secret values are never stored in a repository.

Repositories (8): `lab-aws-infra-bootstrap`, `lab-aws-infra-organization`, `lab-aws-infra-environments`, `lab-aws-gitops`, `lab-aws-backend`, `lab-aws-web`, `lab-aws-github-workflows`, `lab-aws-docs`.

## Repository options

Apply to every repository:

```
for r in lab-aws-infra-bootstrap lab-aws-infra-organization lab-aws-infra-environments lab-aws-gitops lab-aws-backend lab-aws-web lab-aws-github-workflows lab-aws-docs; do
  gh repo edit <owner>/$r --visibility public --accept-visibility-change-consequences \
    --enable-squash-merge --squash-merge-commit-message pr-title-description --enable-merge-commit=false --enable-rebase-merge=false \
    --delete-branch-on-merge --enable-auto-merge --enable-wiki=false --enable-projects=false \
    --enable-secret-scanning --enable-secret-scanning-push-protection
  gh api --method PUT repos/<owner>/$r/vulnerability-alerts
done
```

## Branch ruleset on `main`

New branch ruleset `protect-main`, enforcement active, target the default branch. Rules for every repository: require a pull request with 1 approval, code owner review, dismiss stale approvals on push, and resolved conversations; require linear history; block force pushes and deletions. Require branches to be up to date before merging.

| Repository | Required status checks | Admin bypass |
|---|---|---|
| `lab-aws-infra-bootstrap` | `cfn / cfn-lint` | no |
| `lab-aws-infra-organization` | `terraform / terraform-validate` | no |
| `lab-aws-infra-environments` | `ci-passed` | yes |
| `lab-aws-backend`, `lab-aws-web` | `test`, `image / image` | yes |
| `lab-aws-gitops`, `lab-aws-github-workflows`, `lab-aws-docs` | none | yes |

Admin bypass: add the repository admin role as a bypass actor, mode always. `lab-aws-infra-bootstrap` and `lab-aws-infra-organization` have no bypass actor.

## Tag ruleset

Tag ruleset `protect-release-tags` on every repository, enforcement active, pattern `v*`: restrict updates, restrict deletions, block force pushes.

## Environments

The OIDC trust policies in `lab-aws-infra-bootstrap` (`stage1/seed.yaml`, `stage2/member-account.yaml`) match `sub = repo:<owner>/<repo>:environment:<env>`. The Environment names are part of the AWS trust and must not be renamed.

| Repository | Environment (reviewer required) |
|---|---|
| `lab-aws-infra-organization` | `org-apply` (yes) |
| `lab-aws-infra-environments` | `dev` (no), `test` (yes), `prod` (yes) |
| `lab-aws-backend`, `lab-aws-web` | `dev` (no), `test` (no), `prod` (yes) |

Create one with yourself as reviewer and limit deployments to `main` and `v*` tags:

```
uid=$(gh api user --jq .id)
echo "{\"reviewers\":[{\"type\":\"User\",\"id\":$uid}],\"deployment_branch_policy\":{\"protected_branches\":false,\"custom_branch_policies\":true}}" | gh api --method PUT repos/<owner>/lab-aws-backend/environments/prod --input -
echo '{"name":"main","type":"branch"}' | gh api --method POST repos/<owner>/lab-aws-backend/environments/prod/deployment-branch-policies --input -
echo '{"name":"v*","type":"tag"}' | gh api --method POST repos/<owner>/lab-aws-backend/environments/prod/deployment-branch-policies --input -
```

Without a reviewer, send only `deployment_branch_policy`. With one maintainer, leave "prevent self-review" off.

## Variables and secrets

| Name | Kind | Repositories |
|---|---|---|
| `GITOPS_TOKEN` | secret | `lab-aws-backend`, `lab-aws-web` |
| `DEV_ACCOUNT_ID`, `TEST_ACCOUNT_ID`, `PROD_ACCOUNT_ID` | variable | `lab-aws-backend`, `lab-aws-web`, `lab-aws-infra-environments` |
| `PLATFORM_STATE_BUCKET`, `ALERT_EMAIL`, `KUBERNETES_VERSION`, `API_ALLOWED_CIDRS`, `ADMIN_ROLE_ARNS` | variable | `lab-aws-infra-environments` |

Example: `gh variable set DEV_ACCOUNT_ID --repo <owner>/lab-aws-backend --body <account-id>` and `gh secret set GITOPS_TOKEN --repo <owner>/lab-aws-backend`. `GITOPS_TOKEN` is a fine-grained token with contents and pull-requests write on `lab-aws-gitops`. `API_ALLOWED_CIDRS` and `ADMIN_ROLE_ARNS` are JSON lists.
