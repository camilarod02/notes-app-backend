# Notes API (Backend)

### Run the APIIIIIIIIIIIIIIIII

```bash
cd test-backend
python -m venv .venv

# Windows
.venv\Scripts\activate

# macOS / Linux
source .venv/bin/activate

pip install -r requirements.txt
cp .env.example .env
uvicorn app.main:app --reload
```

Copy `.env.example` to `.env` and set `CORS_ORIGINS` to the frontend URLs that may call the API (comma-separated, no spaces required).

- API: http://127.0.0.1:8000
- Interactive docs: http://127.0.0.1:8000/docs


## Testssssss

```bash
pip install -r requirements.txt
pytest
```

Verbose output:

```bash
pytest -v
```

## Docker

Build and run:

```bash
docker build -t notes-api .
docker run -p 8000:8000 notes-api
```

## Deployment (GitHub Actions + Elastic Beanstalk)

Pushes and pull requests to `main` run tests. A successful **push** to `main` deploys a zip bundle to Elastic Beanstalk.

### AWS setup (one-time)

1. Create an Elastic Beanstalk **application** and **environment**.
2. Platform: **Docker running on 64bit Amazon Linux 2**.
3. Create an IAM user (or role) with permissions to deploy to Elastic Beanstalk and upload to S3.

### GitHub configuration

**Secrets** (Settings → Secrets and variables → Actions):

| Secret | Description |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | IAM access key |
| `AWS_SECRET_ACCESS_KEY` | IAM secret key |

**Variables**:

| Variable | Example | Description |
|----------|---------|-------------|
| `AWS_REGION` | `us-east-1` | AWS region |
| `EB_APPLICATION_NAME` | `notes-api` | EB application name |
| `EB_ENVIRONMENT_NAME` | `notes-api-prod` | EB environment name |

### Pipeline overview

1. **test** — install dependencies, run `pytest`
2. **deploy** (main only) — zip `Dockerfile`, `app/`, `Dockerrun.aws.json`, `.ebextensions`, etc., then deploy with [beanstalk-deploy](https://github.com/einaregilsson/beanstalk-deploy)

### Elastic Beanstalk config files

These files are included in the deployment zip. They tell Elastic Beanstalk how to run the Docker container and how to check that the app is healthy.

#### `Dockerrun.aws.json`

Elastic Beanstalk does not read `Dockerfile` alone on the Docker platform — it also needs a **Dockerrun** manifest. This project uses **version 1** (single-container).

```json
{
  "AWSEBDockerrunVersion": "1",
  "Ports": [
    {
      "ContainerPort": 8000
    }
  ]
}
```

| Field | Meaning |
|-------|---------|
| `AWSEBDockerrunVersion` | `1` = one Docker container per instance |
| `Ports[].ContainerPort` | Port **inside** the container where Uvicorn listens (`8000`, matching the `Dockerfile` and `EXPOSE 8000`) |

Elastic Beanstalk builds the image from `Dockerfile`, runs the container, and routes public HTTP traffic (port 80 on the load balancer) to this container port.

#### `.ebextensions/healthcheck.config`

Files under `.ebextensions/` are applied when the environment is created or updated. This one configures the **load balancer health check** for the default process:

```yaml
option_settings:
  aws:elasticbeanstalk:environment:process:default:
    HealthCheckPath: /health
    Protocol: HTTP
```

| Setting | Meaning |
|---------|---------|
| `HealthCheckPath` | URL path the load balancer calls to decide if the instance is healthy |
| `Protocol` | Use HTTP (not HTTPS) for the check |

That path must exist on the API. The app exposes `GET /health`, which returns `{"status": "ok"}`. If the check fails (wrong path, app crash, or container not listening on port 8000), Elastic Beanstalk marks the instance unhealthy and can roll back the deployment.

**Why both files?** `Dockerrun.aws.json` defines *how the container runs* (which port). `healthcheck.config` defines *how AWS knows the app is working* (which URL to probe). They work together: traffic hits port 8000 in the container, and `/health` confirms the FastAPI process is responding.

### CORS on Elastic Beanstalk

`.env` is not deployed to EB (see `.dockerignore`). In the EB console, open your environment → **Configuration** → **Software** → **Environment properties** and set:

| Property | Example |
|----------|---------|
| `CORS_ORIGINS` | `http://notes-app-frontend-test.s3-website-us-east-1.amazonaws.com` |

Use a comma-separated list for multiple origins (e.g. local dev + S3 website URL). Redeploy or apply the configuration change for it to take effect.
