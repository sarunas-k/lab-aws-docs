# 01 Bootstrap from zero

Result: organization, five accounts, CI roles, state buckets, GitHub repositories and environments.
Needs: AWS CLI v2, Terraform 1.10 or later, `gh`, admin login to the management account, MFA on root. Replace `<...>` values.

1. Check the account plan in the Billing console: it must allow AWS Organizations and creating member accounts.
2. Replace `OWNER` with your GitHub user or organization: `grep -rln OWNER --include='*.yml' --include='*.tf' --exclude-dir=.git --exclude-dir=.terraform --exclude-dir=lab-docs .`
3. Stage 1, in `lab-bootstrap`. Creates bucket `lab-tfstate-org-<mgmt-id>` and roles `lab-github-org-plan`, `lab-github-org-apply`. Add `CreateOidcProvider=false` if the provider exists.
   ```
   aws cloudformation deploy --stack-name lab-bootstrap-stage1 --template-file stage1/seed.yaml \
     --capabilities CAPABILITY_NAMED_IAM --parameter-overrides GitHubOwner=<owner>
   ```
4. Create the organization once: `aws organizations create-organization --feature-set ALL`
5. In `lab-infra-org`, fill `budget_email`, `owner`, `email_local_part`, `email_domain`: `cp terraform.tfvars.example terraform.tfvars`
6. Initialise: `terraform init -backend-config="bucket=lab-tfstate-org-<mgmt-id>"`
7. Import the organization: `terraform import aws_organizations_organization.this <organization-id>`
8. Create OUs, accounts, SCPs and budget `lab-monthly` (100 USD, alerts at 50, 80, 100 percent): `terraform apply`
9. Read account and OU ids: `terraform output account_ids; terraform output ou_ids`
10. In `lab-github`: `cp terraform.tfvars.example terraform.tfvars` (`github_owner`, `reviewer`), `export GITHUB_TOKEN=<fine-grained token>` (repository administration, environments, contents), then `terraform init -backend-config="bucket=lab-tfstate-org-<mgmt-id>"`
11. Import the existing repositories: `for r in lab-bootstrap lab-infra-org lab-github lab-infra-platform lab-gitops lab-backend lab-web lab-cluster lab-github-workflows lab-docs; do terraform import "github_repository.this[\"$r\"]" "$r"; done`
12. Create rulesets and environments (`org-apply`, `dev`, `test`, `prod`): `terraform apply`
13. Stage 2, in `lab-bootstrap`. CI roles in dev, test, prod:
    ```
    aws cloudformation create-stack-set --stack-set-name lab-member-account \
      --template-body file://stage2/member-account.yaml --permission-model SERVICE_MANAGED \
      --auto-deployment Enabled=true,RetainStacksOnAccountRemoval=true --capabilities CAPABILITY_NAMED_IAM \
      --parameters ParameterKey=GitHubOwner,ParameterValue=<owner> ParameterKey=DevAccountId,ParameterValue=<id> \
      ParameterKey=TestAccountId,ParameterValue=<id> ParameterKey=ProdAccountId,ParameterValue=<id> \
      ParameterKey=SharedServicesAccountId,ParameterValue=<shared-id>
    aws cloudformation create-stack-instances --stack-set-name lab-member-account \
      --deployment-targets OrganizationalUnitIds=<workloads-ou-id> --regions eu-central-1
    ```
14. Stage 2, platform state bucket `lab-tfstate-platform-<shared-id>`:
    ```
    aws cloudformation create-stack-set --stack-set-name lab-shared-state \
      --template-body file://stage2/shared-state.yaml --permission-model SERVICE_MANAGED \
      --auto-deployment Enabled=false --parameters ParameterKey=OrganizationId,ParameterValue=<o-xxxx> \
      ParameterKey=DevAccountId,ParameterValue=<id> ParameterKey=TestAccountId,ParameterValue=<id> ParameterKey=ProdAccountId,ParameterValue=<id>
    aws cloudformation create-stack-instances --stack-set-name lab-shared-state \
      --deployment-targets OrganizationalUnitIds=<infrastructure-ou-id> --regions eu-central-1
    ```
15. Check both: `aws cloudformation list-stack-instances --stack-set-name lab-member-account` (repeat for `lab-shared-state`)
16. Set variables on `lab-infra-platform`: `gh variable set DEV_ACCOUNT_ID --repo <owner>/lab-infra-platform --body <id>`. Repeat for `TEST_ACCOUNT_ID`, `PROD_ACCOUNT_ID`, `PLATFORM_STATE_BUCKET` (`lab-tfstate-platform-<shared-id>`), `ALERT_EMAIL`, `KUBERNETES_VERSION`, `API_ALLOWED_CIDRS`, `ADMIN_ROLE_ARNS` (the last two as JSON lists), and the three `*_ACCOUNT_ID` on `lab-backend`, `lab-web`, `lab-cluster`.
17. Set the secret on those three repositories: `gh secret set GITOPS_TOKEN --repo <owner>/lab-backend` (contents and pull-requests write on `lab-gitops`).
18. Continue with [02 Deploy an environment](02-deploy-environment.md).
