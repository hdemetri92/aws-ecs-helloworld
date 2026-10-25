# Terraform Infrastructure

Terraform is split into reusable modules, a shared root, a production root,
and a separate state-bootstrap root:

```text
terraform/
├── modules/
│   ├── network/
│   ├── ecs-service/
│   └── database/
├── shared/
│   └── state-bootstrap/
└── production/
```

## Roots and modules

### Shared root

[`terraform/shared`](../terraform/shared) owns account/environment resources
consumed by deployments: the GitHub OIDC roles, immutable ECR repository,
private versioned frontend bucket, optional Route 53 hosted zone, and
DNS-validated ACM certificates. CloudFront's viewer certificate is requested
in `us-east-1`; the ALB certificate is requested in the configured AWS region.

### Production root

[`terraform/production`](../terraform/production) reads shared resource
identifiers through Terraform remote state. It provisions the production VPC,
ECS/Fargate service, ALB, CloudFront distribution and aliases when a domain is
configured, and optional RDS.

### Reusable modules

- `network`: VPC, three subnet tiers per AZ, internet gateway, same-AZ NAT
  Gateways, and explicit route tables/associations.
- `ecs-service`: ALB, security groups, Fargate cluster/service, separate task
  and execution roles, CloudWatch logs, health checks, and CPU autoscaling.
- `database`: optional private PostgreSQL RDS, isolated subnet group, and
  application-task-only database ingress.

The production root composes all three modules. Shared changes affect
production planning because the production root consumes outputs from shared
state.

## State and locking

The state-bootstrap root uses local state because it creates the S3 bucket
needed for remote state. Run it once, keep its local state file safe, and do
not commit that file.

Shared and production use independent S3 state keys:

- `hello-saas/shared/terraform.tfstate`
- `hello-saas/production/terraform.tfstate`

The state bucket uses encryption, versioning, public-access blocking, and a
TLS-only bucket policy. Both roots use S3 native lock files
(`use_lockfile=true`). The state bucket is outside these Terraform states and
is protected by `prevent_destroy`.

## One-time state bootstrap

Prerequisites: Terraform 1.10+, AWS CLI credentials with permissions for the
initial setup, and a globally unique S3 bucket name. Validation and tests do
not create AWS resources.

```sh
export AWS_REGION=us-east-2
export TF_STATE_BUCKET=terraform-state-bucket-helloworld
export GITHUB_REPOSITORY=hdemetri92/aws-ecs-helloworld

terraform -chdir=terraform/shared/state-bootstrap init
terraform -chdir=terraform/shared/state-bootstrap apply \
  -var="aws_region=$AWS_REGION" \
  -var="terraform_state_bucket=$TF_STATE_BUCKET"
```

Initialize and apply the shared root using its remote state:

```sh
terraform -chdir=terraform/shared init \
  -backend-config="bucket=$TF_STATE_BUCKET" \
  -backend-config="key=hello-saas/shared/terraform.tfstate" \
  -backend-config="region=$AWS_REGION" \
  -backend-config="encrypt=true" \
  -backend-config="use_lockfile=true"

terraform -chdir=terraform/shared apply \
  -var="aws_region=$AWS_REGION" \
  -var="github_repository=$GITHUB_REPOSITORY" \
  -var="terraform_state_bucket=$TF_STATE_BUCKET"
```

Configure the GitHub Actions variables and `production` environment described
in the [deployment workflows guide](./deployment-workflows.md) after the
shared outputs exist.

## Optional GoDaddy domain setup

Terraform can create the Route 53 hosted zone, ACM certificates, validation
records, and CloudFront configuration. GoDaddy remains the registrar, so
nameserver delegation is a manual step.

First create only the hosted zone and obtain its nameservers:

```sh
terraform -chdir=terraform/shared apply -target=aws_route53_zone.main \
  -var="aws_region=$AWS_REGION" \
  -var="github_repository=$GITHUB_REPOSITORY" \
  -var="terraform_state_bucket=$TF_STATE_BUCKET" \
  -var="domain_name=datumintellectus.com" \
  -var="create_route53_zone=true"

terraform -chdir=terraform/shared output route53_name_servers
```

Set the domain nameservers in GoDaddy to the Route 53 nameservers. Before
changing delegation, preserve and recreate any DNS records you still need
(such as email MX, SPF, DKIM, or verification TXT records) in Route 53. After
delegation propagates, run the normal shared apply with the same domain values
so Terraform can create certificates and validation records.

The production root then creates CloudFront aliases for the apex, `www`, and
`app` hostnames. Do not change registrar nameservers until the Route 53 hosted
zone exists and its nameservers have been recorded.

## Migration caution

The former layout used one Terraform root under `infra/` and a single state
key. Terraform cannot automatically split an existing state into the new
shared and production states. If the old configuration was applied, back up
the state and explicitly map resources to the new roots before running the
new deployment workflow or destroying either stack.
