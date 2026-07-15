# CI/CD Pipeline Architecture and Strategy

This document details the CI/CD pipeline structure, environment variables, rollback strategy, and common troubleshooting steps for deploying containerized applications.

## Pipeline Structure

The pipeline applies to both frontend and backend projects, running through the following stages:

~~~yaml
stages:
  - build
  - test
  - push
  - deploy
  - notify
~~~

### Trigger Conditions

Deploy jobs are restricted to run only on a specific branch:

~~~yaml
rules:
  - if: '$CI_COMMIT_BRANCH == $DEPLOY_BRANCH'
~~~

### 1. Build Stage

**Backend Example:**
~~~bash
docker build --provenance=false -t "$IMAGE_NAME" .
~~~

**Frontend Example (with build arguments):**
~~~bash
docker build \
  --provenance=false \
  --build-arg NEXT_PUBLIC_API_URL="$NEXT_PUBLIC_API_URL" \
  --build-arg NEXT_PUBLIC_GOOGLE_CLIENT_ID="$NEXT_PUBLIC_GOOGLE_CLIENT_ID" \
  -t "$IMAGE_NAME" .
~~~
> **Tip:** `--provenance=false` prevents Docker from attaching SBOM metadata which can sometimes cause compatibility issues with self-hosted registries.

### 2. Test Stage

If using a runner with the host Docker socket (`/var/run/docker.sock`), avoid using fixed host port mapping during tests, as it will cause port conflicts on the build server. Use container networking or randomized host ports.

### 3. Push Stage

Logs in to the registry and pushes the tagged image:
~~~bash
echo "$CI_REGISTRY_PASSWORD" | docker login "$CI_REGISTRY" -u "$CI_REGISTRY_USER" --password-stdin
docker push "$IMAGE_NAME"
~~~

### 4. Deploy Stage

The core deployment logic executing on the runner:
1. Write the SSH private key and known hosts.
2. SSH into the `<APP_VM>`.
3. Update the `CURRENT_IMAGE` and `PREVIOUS_IMAGE` variables in the deployment `.env` file.
4. Execute `docker compose pull`.
5. Execute `docker compose up -d --force-recreate`.
6. Run a health check (e.g., `curl -f http://localhost:<PORT>`).
7. Roll back automatically if the health check fails.

### 5. Notify Stage

An example shared notification template using GitLab `include`:
~~~yaml
include:
  - project: "<COMPANY_GROUP>/infra-templates"
    ref: "main"
    file: "/email-notification.yml"
~~~

## Required CI/CD Variables

Configure these at the Group or Project level.

| Variable | Purpose | Example |
|---|---|---|
| `DEPLOY_BRANCH` | Branch allowed to deploy | `main` |
| `APP_NAME` | Target directory name | `my-app` |
| `DEPLOY_IMAGE_VARIABLE` | Key in `.env` for current image | `<APP_PREFIX>_BACKEND_IMAGE` |
| `DEPLOY_PREVIOUS_IMAGE_VARIABLE`| Key in `.env` for previous image| `<APP_PREFIX>_BACKEND_PREVIOUS_IMAGE` |
| `SSH_PRIVATE_KEY` | Key for `<APP_VM>` SSH access | (secret) |
| `APP_VM_SSH_HOST_IP` | `<APP_VM>` IP address | `192.168.1.10` |
| `APP_VM_SSH_USER` | Deploy SSH user | `deployer` |
| `APP_VM_SSH_KNOWN_HOSTS`| Host key pinning | (from `ssh-keyscan`) |
| `SSH_PORT` | SSH port | `22` |
| `BASE_DEPLOY_PATH` | Base deploy directory | `/opt/apps` |

## Rollback Strategy

The application uses a **single-previous-image** rollback model.

### Automated Rollback

If the post-deployment health check fails during the pipeline:
1. Read the previous image tag from the `.env` file.
2. Replace the current image variable with the previous one.
3. Run `docker compose up -d --force-recreate`.
4. Fail the pipeline (since the new release failed).
5. Send a rollback notification.

### Manual Rollback

If you need to roll back manually via SSH on the `<APP_VM>`:
~~~bash
cd /opt/apps/<PROJECT_NAME>

# Extract previous image
PREVIOUS=$(grep '^<APP_PREFIX>_BACKEND_PREVIOUS_IMAGE=' .env | cut -d '=' -f2-)

# Overwrite current with previous
sed -i "s|^<APP_PREFIX>_BACKEND_IMAGE=.*|<APP_PREFIX>_BACKEND_IMAGE=$PREVIOUS|" .env

# Recreate service
docker compose pull
docker compose up -d --force-recreate

# Verify
curl -f http://localhost:<PORT>
~~~

## Common Troubleshooting

### Missing Build-Time Variables (e.g., Next.js)
**Cause:** Variables like `NEXT_PUBLIC_*` must be available during `docker build`, not just at runtime.
**Fix:** Pass them via `--build-arg` in the `docker build` command and define them as `ARG` and `ENV` in the `Dockerfile`.

### Docker Test Port Conflicts
**Cause:** The GitLab runner uses the host Docker socket, meaning exposed test ports bind directly to the `<BUILD_VM>`.
**Fix:** Use random ports or Docker networks for tests instead of fixed port mapping like `-p 3000:3000`.

### GitLab Include Access Denied
**Cause:** The user triggering the pipeline does not have access to the included repository (e.g., `infra-templates`).
**Fix:** Grant the user access to the shared repository.

### Missing Protected Variables
**Cause:** Protected CI/CD variables only expose themselves to protected branches or tags.
**Fix:** Ensure `$DEPLOY_BRANCH` is marked as a Protected Branch in the repository settings.

### Compose Starts Old Image
**Fix:** Ensure you pull and force recreate:
~~~bash
docker compose pull
docker compose up -d --force-recreate
~~~
