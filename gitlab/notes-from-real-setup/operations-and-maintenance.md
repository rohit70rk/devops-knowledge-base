# Operations and Maintenance

This document covers daily checks, operational runbooks, and backup/recovery procedures for the CI/CD platform and hosted applications.

## Daily Checks

| Check | Command / Location |
|---|---|
| GitLab health | `sudo gitlab-ctl status` on `<BUILD_VM>` |
| Runner online | `sudo gitlab-runner status` or GitLab Runner UI |
| Registry reachable | `docker login <REGISTRY_URL>` |
| App containers | `docker compose ps` in `/opt/apps/<PROJECT_NAME>` on `<APP_VM>` |
| Public application | `curl -I https://<APP_URL>` |
| Disk space | `df -h` on `<BUILD_VM>` and `<APP_VM>` |

### Validating the Runtime Environment

To verify an application is running correctly from the `<APP_VM>`:
~~~bash
cd /opt/apps/<PROJECT_NAME>
docker compose ps
curl -f http://localhost:<INTERNAL_PORT>
cat .env
~~~

## Operational Rules

- **Do not edit** the application `.env` files manually except during emergency deployment recovery or rollback.
- **Do not delete** registry tags unless you are certain they are not needed for a rollback.
- **Do not remove** shared templates (like `infra-templates`) access from users who need to deploy.
- **Do not change** the `DEPLOY_BRANCH` CI variable without restoring it immediately after testing.
- When Git histories diverge, use `--force-with-lease` rather than an unsafe force push.

---

## Operational Runbooks

### 1. Switch Deployment Branch Temporarily

To test a deployment from a non-standard branch:
1. Navigate to: `Project -> Settings -> CI/CD -> Variables -> DEPLOY_BRANCH`
2. Update it to the temporary branch name.
3. Protect the temporary branch if it requires access to protected variables.
4. Run the test deployment.
5. Immediately restore the original value (e.g., `DEPLOY_BRANCH=main`).

### 2. Recover From Failed Deployment

1. Check the failed job log in GitLab.
2. Determine if the automated rollback successfully executed.
3. Inspect the `<APP_VM>` state:
   ~~~bash
   cd /opt/apps/<PROJECT_NAME>
   docker compose ps
   docker compose logs --tail=200
   cat .env
   ~~~
4. If automated rollback failed, run a manual rollback by restoring the previous image tag in `.env` and recreating the container (see `ci-cd-pipelines.md`).

### 3. Recover the GitLab Runner

If CI jobs are stuck pending:
~~~bash
sudo gitlab-runner status
sudo gitlab-runner verify
sudo systemctl restart gitlab-runner
sudo -u gitlab-runner docker info
~~~
If Docker permission issues are observed:
~~~bash
sudo usermod -aG docker gitlab-runner
sudo systemctl restart gitlab-runner
~~~

### 4. Recover SSH Deployment Access

If the CI pipeline fails during the deployment stage due to SSH issues:
~~~bash
# From a secure operator machine
ssh -p <SSH_PORT> <DEPLOY_USER>@<APP_VM_IP>
~~~
If the host key changed (e.g., the `<APP_VM>` was rebuilt):
~~~bash
ssh-keyscan -p <SSH_PORT> <APP_VM_IP>
~~~
Update the `APP_VM_SSH_KNOWN_HOSTS` CI/CD variable with the new output.

---

## Backup and Recovery

### What Needs Backing Up

| Area | Component |
|---|---|
| **GitLab** | Standard GitLab backup (repositories, DB, uploads) |
| **GitLab Config** | `/etc/gitlab/gitlab.rb` and GitLab secrets JSON |
| **Registry** | The directory defined by the registry storage path |
| **Runner** | `/etc/gitlab-runner/config.toml` |
| **Application** | `/opt/apps/<PROJECT_NAME>` (Docker Compose files and `.env`) |
| **Proxy** | Nginx site files and SSL certificates/renewal configs |

### Performing Backups

**GitLab and Secrets (`<BUILD_VM>`)**:
~~~bash
sudo gitlab-backup create
sudo cp /etc/gitlab/gitlab.rb /secure-backup-location/gitlab.rb
sudo cp /etc/gitlab/gitlab-secrets.json /secure-backup-location/gitlab-secrets.json
~~~

**Application (`<APP_VM>`)**:
~~~bash
cd /opt/apps
sudo tar -czf /secure-backup-location/app-vm-backup.tgz <PROJECT_NAME>
~~~

**Proxy (`<PROXY_VM>`)**:
~~~bash
sudo tar -czf /secure-backup-location/nginx-backup.tgz /etc/nginx/sites-available /etc/nginx/sites-enabled
~~~

### Recovery Order (Disaster Recovery)

1. Restore DNS and `<PROXY_VM>` routing.
2. Restore GitLab CE and the Container Registry on the `<BUILD_VM>`.
3. Restore the GitLab Runner configuration.
4. Restore the application compose files on the `<APP_VM>`.
5. Validate that the `<APP_VM>` can successfully pull images from the registry.
6. Start applications: `docker compose up -d --force-recreate`.
7. Validate public URLs.
