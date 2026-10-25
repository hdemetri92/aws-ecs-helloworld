# Deployment Workflows

GitHub Actions validates pull requests and deploys changes from `main`. The
workflows use GitHub OIDC for AWS access; do not configure long-lived AWS
access keys in repository secrets.

## Workflow files

- [`terraform-checks.yml`](../.github/workflows/terraform-checks.yml) runs on
  pull requests. It selects application, Terraform, and plan jobs based on
  changed paths. Terraform plans are read-only and posted to a sticky PR
  comment.
- [`deploy.yml`](../.github/workflows/deploy.yml) runs on pushes to `main` and
  manual dispatches from `main`. It plans and applies only the affected
  infrastructure and deployable application changes.
- [`deploy-frontend.yml`](../.github/workflows/deploy-frontend.yml) provides a
  manual-only path to republish the static frontend.

## Change selection

| Change | Checks or deployment |
| --- | --- |
| `terraform/shared/**` (excluding state bootstrap) | Validate and plan shared and production on PR. On `main`, plan/apply shared first, then plan and gate production. |
| `terraform/production/**` or `terraform/modules/**` | Validate and plan production. On `main`, deploy production without applying shared. |
| `python_service/**`, Dockerfile, `.dockerignore`, or runtime dependencies | Run app tests and build the container. On `main`, push an immutable image and deploy it through the production plan. |
| `tests/**` or development-only dependencies | Run app tests and build checks on pull requests; these changes alone do not publish a new application image on `main`. |
| `frontend/**` only | No Terraform plan or ECS image build. On `main`, publish the frontend to S3 and invalidate CloudFront. |
| `terraform/shared/state-bootstrap/**` | Validate the bootstrap root; it is not automatically applied. |

Production consumes outputs from shared Terraform state, so shared changes
also trigger production validation and planning. If infrastructure changes
without an application change, the workflow reuses the image currently active
on ECS instead of rebuilding it. A manual dispatch runs the full deployment
path.

## Production approval and plan artifacts

Production plans are saved as one-day GitHub Actions artifacts. The production
apply job uses the exact saved binary plan rather than creating a new plan
after approval. Configure a GitHub Actions environment named `production`,
restrict it to `main`, and require reviewers to enable the approval gate.
Access to plan artifacts should be limited because Terraform plans can contain
sensitive infrastructure data.

The deploy workflow serializes its runs using a GitHub Actions concurrency
group. Terraform's S3 native lock files also serialize operations against each
state. If the state changes after a plan is created, Terraform rejects the
stale plan; create and review a new plan.

## GitHub configuration

Add these repository Actions variables:

| Variable | Purpose |
| --- | --- |
| `AWS_REGION` | Deployment region |
| `TF_STATE_BUCKET` | S3 bucket created by the state-bootstrap root |
| `AWS_ROLE_ARN` | Deployment role ARN from `terraform -chdir=terraform/shared output -raw github_actions_role_arn` |
| `AWS_PLAN_ROLE_ARN` | Read-only PR plan role ARN from `terraform -chdir=terraform/shared output -raw github_plan_role_arn` |
| `DOMAIN_NAME` | Optional apex domain; leave unset for ALB HTTP mode |
| `ROUTE53_ZONE_ID` | Optional existing Route 53 hosted-zone ID |
| `CREATE_ROUTE53_ZONE` | Set to `true` when shared Terraform creates the zone |
| `GITHUB_OIDC_PROVIDER_ARN` | Optional existing GitHub OIDC provider ARN |
| `ENABLE_DATABASE` | Optional `true` to enable PostgreSQL |
| `DATABASE_MULTI_AZ` | Optional `true` to enable Multi-AZ RDS |
| `DATABASE_DELETION_PROTECTION` | Optional `true` to protect RDS from deletion |

The shared Terraform root configures the deployment and plan roles. The
deployment role trusts the repository's `main` branch for shared planning and
the `production` GitHub environment for protected production operations. The
plan role is intended for same-repository pull requests; fork PRs do not
receive AWS credentials.

## First deployment and manual publishing

The state-bootstrap and shared roots must be initialized before automated
deployments can use the remote states or OIDC roles. Follow the
[Terraform infrastructure guide](./terraform-infrastructure.md) to bootstrap
those prerequisites.

The standalone frontend workflow is for manually republishing existing
frontend assets. It requires the shared bucket and production CloudFront
distribution to exist. Normal frontend-only pushes are published by
`deploy.yml`; infrastructure changes that affect frontend delivery publish
after the protected production apply.
