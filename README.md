# Hello SaaS on AWS

A portfolio project demonstrating a FastAPI service on Amazon ECS Fargate, a
static frontend on private S3, CloudFront delivery, modular Terraform, and
path-aware GitHub Actions deployments using AWS OIDC.

## Project documentation

- [Architecture overview](./docs/architecture-overview.md): request flow,
  network layout, security boundaries, and design tradeoffs.
- [Application and frontend](./docs/application.md): service routes, frontend
  behavior, local development, tests, and container details.
- [Terraform infrastructure](./docs/terraform-infrastructure.md): Terraform
  roots and modules, state, bootstrap, and domain configuration.
- [Deployment workflows](./docs/deployment-workflows.md): PR checks, path
  filters, deployment sequencing, approvals, and GitHub configuration.

## Quick start

Requires Python 3.12 or newer.

```sh
python -m venv .venv
source .venv/bin/activate
pip install -r requirements-dev.txt
uvicorn python_service.main:app --reload
```

The API is available at `http://localhost:8000/api/hello` and
`http://localhost:8000/health`. Run tests with `pytest`. The static frontend is
deployed separately to S3 and is not served by the local FastAPI process.

```sh
docker build -t hello-saas .
docker run --rm -p 8000:8000 hello-saas
```

## Scope and operating costs

The baseline provisions multi-AZ networking, a public ALB, private ECS tasks,
and one NAT Gateway per AZ. NAT Gateways incur ongoing hourly and data
processing charges. ECS runs at least two tasks. Optional RDS is disabled by
default; the sample application does not connect to it.

Custom DNS and TLS are optional. Without a configured domain, the study
deployment exposes the ALB over HTTP; do not treat that mode as production
internet-facing security. This project is a portfolio demonstration, not a
complete production platform. WAF/rate limiting, alarms, restore testing, and
more tightly scoped deployment permissions remain areas for further work.
