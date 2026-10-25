# Application and Frontend

The application consists of a static frontend and a small FastAPI backend.
They are deployed independently: CloudFront serves static files from a private
S3 bucket, while ECS Fargate runs the containerized Python service.

## Python service

The service entry point is
[`python_service/main.py`](../python_service/main.py). It exposes:

| Route | Purpose |
| --- | --- |
| `GET /api/hello` | Returns a hello-world JSON response |
| `GET /health` | ECS/ALB health check |

FastAPI runs under Uvicorn in the container. It does not serve the frontend
and does not currently connect to the optional PostgreSQL database.

## Static frontend

The frontend lives in [`frontend/`](../frontend). `index.html` and
`styles.css` are uploaded to a private S3 bucket. CloudFront serves the static
content and forwards `/api/*` and `/health` requests to the ALB, which routes
them to the FastAPI service. The page's button calls the same-origin
`/api/hello` endpoint.

## Local development

Requires Python 3.12 or newer:

```sh
python -m venv .venv
source .venv/bin/activate
pip install -r requirements-dev.txt
uvicorn python_service.main:app --reload
```

Visit `http://localhost:8000/docs` for the generated API documentation.
The frontend is separate from the local FastAPI server; use a static file
server or the deployed CloudFront URL to view it with the API.

## Tests and container

Tests live in [`tests/`](../tests) and cover the hello endpoint, health check,
and separation of frontend assets from the API. Run them with:

```sh
pytest
```

The root [`Dockerfile`](../Dockerfile) installs runtime dependencies, copies
`python_service/`, and starts Uvicorn as a non-root user:

```sh
docker build -t hello-saas .
docker run --rm -p 8000:8000 hello-saas
```

GitHub Actions runs the tests and builds the container when the Python service,
tests, Dockerfile, or Python dependencies change. See the
[deployment workflows guide](./deployment-workflows.md) for the deployment
sequence and path filters.

## Current scope

This is a hello-world service, not a complete SaaS application. It has no
authentication, persistence, or application-level database integration. The
optional RDS resource is an infrastructure example only.
