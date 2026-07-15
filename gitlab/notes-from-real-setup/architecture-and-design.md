# Architecture Overview & Deployment Flow

This document outlines the architectural pattern and deployment flow for a self-hosted GitLab CE CI/CD infrastructure deploying containerized applications.

## System Flow

~~~mermaid
flowchart LR
  VCS["Source Code Repository (e.g. GitHub/GitLab)"] --> GitLab["GitLab CE Deployment Repositories"]
  Dev["Developer"] --> VCS
  Dev --> GitLab
  GitLab --> Pipeline["GitLab CI/CD Pipeline"]
  Pipeline --> Runner["GitLab Runner on <BUILD_VM>"]
  Runner --> Registry["GitLab Container Registry"]
  Runner --> AppVM["<APP_VM> via SSH"]
  Registry --> AppVM
  AppVM --> Compose["Docker Compose"]
  Compose --> Frontend["Frontend Container"]
  Compose --> Backend["Backend Container"]
  Nginx["<PROXY_VM> Nginx"] --> Frontend
  Nginx --> Backend
  Users["Users"] --> Nginx
~~~

## Server Roles

| Server | Role | Key Paths / Services |
|---|---|---|
| `<BUILD_VM>` | GitLab CE, registry, runner, Docker image builds | GitLab services, `/etc/gitlab`, `/etc/gitlab-runner/config.toml` |
| `<APP_VM>` | Runs application containers | `/opt/apps/<PROJECT_NAME>`, Docker Compose |
| `<PROXY_VM>` | Public Nginx reverse proxy and SSL termination | `/etc/nginx/sites-available`, `/etc/nginx/sites-enabled` |

## Project Structure Pattern

Group your projects logically in GitLab (e.g. `<COMPANY_GROUP>`):

~~~text
<COMPANY_GROUP>
|-- <PROJECT_NAME>-backend
|-- <PROJECT_NAME>-frontend
|-- infra-templates
~~~

> **Tip:** Use a repository like `infra-templates` to provide shared assets, such as email notification templates or standard CI scripts, used across multiple application pipelines.

## Runtime Layout On Application VM

Organize the application deployment directories cleanly:

~~~text
/opt/apps/<PROJECT_NAME>/
|-- docker-compose.yml
|-- .env
|-- backend.env
|-- frontend.env
~~~

The root `.env` stores image variables used by Compose, for example:

- `<APP_PREFIX>_BACKEND_IMAGE`
- `<APP_PREFIX>_BACKEND_PREVIOUS_IMAGE`
- `<APP_PREFIX>_FRONTEND_IMAGE`
- `<APP_PREFIX>_FRONTEND_PREVIOUS_IMAGE`

## Pipeline To Runtime Flow

~~~mermaid
flowchart TD
  A["Merge or push to DEPLOY_BRANCH"] --> B["Build Docker image"]
  B --> C["Test stage"]
  C --> D["Login to GitLab Registry"]
  D --> E["Push image tagged with commit SHA"]
  E --> F["SSH to <APP_VM>"]
  F --> G["Move current image to previous image variable"]
  G --> H["Set current image to new image"]
  H --> I["docker compose pull"]
  I --> J["docker compose up -d --force-recreate"]
  J --> K["Health check localhost port"]
  K -->|healthy| L["Notify success"]
  K -->|unhealthy| M["Restore previous image"]
  M --> N["Recreate service"]
  N --> O["Notify rollback and fail pipeline"]
~~~

## Image Naming Convention

Use GitLab built-in variables for robust image tagging:

~~~text
IMAGE_NAME=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
~~~

Example:
~~~text
<GITLAB_REGISTRY_URL>/<COMPANY_GROUP>/<PROJECT_NAME>-backend:<commit-sha>
~~~

## Deployment Rules

Tie deployments to a specific branch:

~~~yaml
rules:
  - if: '$CI_COMMIT_BRANCH == $DEPLOY_BRANCH'
~~~

Define `DEPLOY_BRANCH` in the repository's CI/CD variables (e.g., `main` or `production`).

## Deployment Execution Context

When updating the application on the `<APP_VM>`, the deployment script typically relies on Docker Compose:

1. `docker compose pull` fetches the newly pushed SHA tag from the registry.
2. `docker compose up -d --force-recreate` ensures the service is recreated, picking up the new image even when Compose might try to reuse an existing container.

## Health Checks

Always run health checks on the `<APP_VM>` immediately after Compose recreates containers:

| Service | Check Command Example |
|---|---|
| Backend | `curl -f http://localhost:<BACKEND_PORT>` |
| Frontend | `curl -f http://localhost:<FRONTEND_PORT>` |
