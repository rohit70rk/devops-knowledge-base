# Application Deployment Pattern

This guide explains how to add new containerized applications to the deployment platform and how developers interact with the deployment workflow.

## Overview

A typical application (e.g., a Backend API or a Frontend web app) follows this pattern:
- **Source Code Repository**: Hosted externally (e.g., GitHub, Bitbucket).
- **Deployment Repository**: Hosted on the internal GitLab CE instance (e.g., under a specific `<COMPANY_GROUP>`).
- **CI/CD Build**: The internal GitLab instance handles Docker builds and pushes images to the GitLab Container Registry.
- **Runtime**: Deployed via Docker Compose on the `<APP_VM>`.
- **Routing**: Exposed publicly through the `<PROXY_VM>` via Nginx.

## How to Add a New Application

Follow this checklist to onboard a new service to the infrastructure.

### 1. Repository Setup

1. Keep or create the primary source repository (e.g., GitHub).
2. Create the deployment project in GitLab under the designated group (e.g., `<COMPANY_GROUP>`).
3. Add developer access and configure branch protection rules.

### 2. Docker Configuration

The application repository must include:
- `Dockerfile`
- `.dockerignore`
- `.gitlab-ci.yml`

Ensure Docker images are tagged using commit SHAs for reliable rollbacks:
~~~text
IMAGE_NAME=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
~~~

> **Note for Frontend Apps (e.g. Next.js):**
> Public environment variables must be passed at build time. Ensure the `Dockerfile` uses `ARG` and `ENV` for variables like `NEXT_PUBLIC_API_URL`, and the CI script injects them using `--build-arg`.

### 3. Target Environment Setup (`<APP_VM>`)

1. Add the new service block to the central `docker-compose.yml` file (e.g., at `/opt/apps/<PROJECT_NAME>/docker-compose.yml`).
2. Expose an internal port on the host (e.g., `3001` or `4001`) and configure a health check.
3. Update the root `.env` file to track the application's images:
   ~~~env
   <NEW_APP_PREFIX>_IMAGE=<registry-url>/<company-group>/<app>:<tag>
   <NEW_APP_PREFIX>_PREVIOUS_IMAGE=<registry-url>/<company-group>/<app>:<tag>
   ~~~

### 4. Configure CI/CD Variables

Ensure the following variables are configured in the GitLab project settings:
- `APP_NAME`: Used to resolve the target deployment directory.
- `DEPLOY_BRANCH`: The branch that triggers deployments (e.g., `main`).
- `DEPLOY_IMAGE_VARIABLE`: The `.env` key to update on deployment.
- `DEPLOY_PREVIOUS_IMAGE_VARIABLE`: The `.env` key to store the previous image tag.

(Shared credentials like `SSH_PRIVATE_KEY` and `APP_VM_SSH_HOST_IP` can remain at the group level).

### 5. Configure Routing (`<PROXY_VM>`)

Add an Nginx upstream route pointing to the new `<APP_VM>` internal port.
Verify and reload:
~~~bash
sudo nginx -t
sudo systemctl reload nginx
~~~

### 6. Validate the First Deployment

1. Set the correct `DEPLOY_BRANCH`.
2. Merge your changes.
3. Verify the image appears in the GitLab Container Registry.
4. Verify the `<APP_VM>` successfully pulled the new image and the local health check passes.
5. Verify the application is accessible through the public route.

---

## Developer Onboarding Workflow

### Dual-Remote Setup

Developers typically push their code to the primary source repository for peer review and then to the internal GitLab repository to trigger deployments.

Add remotes to the local repository:
~~~bash
git remote add origin https://github.com/<SOURCE_ORG>/<APP_REPO>.git
git remote add gitlab https://<GITLAB_URL>/<COMPANY_GROUP>/<APP_REPO>.git
~~~

### Normal Development Flow

1. Create a feature branch:
   ~~~bash
   git checkout -b feature/<name>
   ~~~
2. Push to both remotes:
   ~~~bash
   git push origin feature/<name>
   git push gitlab feature/<name>
   ~~~
3. Create a merge request in GitLab targeting the branch configured by `DEPLOY_BRANCH` (e.g., `main`).

> **Rule:** Never push directly to the deployment branch.

### Handling Divergent Histories

If the GitHub and GitLab histories diverge, synchronize them safely using `--force-with-lease`:
~~~bash
git pull gitlab <branch> --rebase
git push gitlab <branch> --force-with-lease
git push origin <branch> --force-with-lease
~~~
