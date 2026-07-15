# Installation and Setup Guide

This guide documents the generic procedure for building the CI/CD and deployment platform from scratch.

## 1. Prepare DNS / Hosts

Ensure DNS records or `/etc/hosts` entries are mapped correctly before beginning.
Required hostnames:

| Hostname | Target |
|---|---|
| `<GITLAB_URL>` | GitLab CE on `<BUILD_VM>` through `<PROXY_VM>` |
| `<REGISTRY_URL>` | GitLab Container Registry through `<PROXY_VM>` |
| `<APP_URL>` | Application via `<APP_VM>` through `<PROXY_VM>` |

## 2. Prepare Build Server (`<BUILD_VM>`)

Install Docker and GitLab CE.

1. Verify installation:
   ~~~bash
   docker --version
   sudo gitlab-ctl status
   sudo gitlab-rake gitlab:check
   ~~~

2. Verify GitLab configuration (`/etc/gitlab/gitlab.rb`):
   ~~~bash
   sudo grep -E "external_url|registry_external_url|trusted_proxies|nginx|registry" /etc/gitlab/gitlab.rb
   ~~~

3. Apply config changes if needed:
   ~~~bash
   sudo gitlab-ctl reconfigure
   sudo gitlab-ctl restart
   ~~~

4. In the GitLab UI, create the required `<COMPANY_GROUP>` and corresponding application projects.

## 3. Configure Container Registry

The registry stores images built by CI. It is typically hosted on the `<BUILD_VM>` alongside GitLab.

1. Ensure `registry_external_url` is set correctly in `/etc/gitlab/gitlab.rb`.
2. Reconfigure GitLab and check status.
3. Test manual login from the `<BUILD_VM>`:
   ~~~bash
   docker login <REGISTRY_URL>
   ~~~

## 4. Install GitLab Runner

Register a Docker executor runner on `<BUILD_VM>` to run pipelines.

Required runner configuration (`/etc/gitlab-runner/config.toml`):
~~~toml
executor = "docker"
concurrent = 3 # Adjust based on CPU
pull_policy = "if-not-present"
volumes = ["/var/run/docker.sock:/var/run/docker.sock", "/cache"]
~~~

> **Important:** Sharing the host Docker socket (`/var/run/docker.sock`) allows build and push jobs to share images natively. However, CI jobs have high control over the `<BUILD_VM>`, so restrict the runner to trusted projects only.

Verify runner:
~~~bash
sudo gitlab-runner status
sudo gitlab-runner verify
sudo -u gitlab-runner docker info
~~~
If Docker permission fails:
~~~bash
sudo usermod -aG docker gitlab-runner
sudo systemctl restart gitlab-runner
~~~

## 5. Prepare Application Server (`<APP_VM>`)

1. Create a dedicated deploy user and application directory:
   ~~~bash
   sudo useradd -m -s /bin/bash deployer
   sudo usermod -aG docker deployer
   sudo mkdir -p /opt/apps/<PROJECT_NAME>
   sudo chown -R deployer:deployer /opt/apps/<PROJECT_NAME>
   ~~~

2. Setup the `.env` file at `/opt/apps/<PROJECT_NAME>/.env` containing variables matching the registry tags pushed by CI.

3. Verify Docker Compose deployment manually before relying on pipelines:
   ~~~bash
   cd /opt/apps/<PROJECT_NAME>
   docker compose pull
   docker compose up -d --force-recreate
   docker compose ps
   ~~~

## 6. Configure Nginx Proxy Server (`<PROXY_VM>`)

The proxy exposes GitLab, the Registry, and the Applications securely to the public.

Example application proxy shape:
~~~nginx
server {
    listen 80;
    server_name <APP_URL>;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name <APP_URL>;

    location / {
        proxy_pass http://<APP_VM_IP>:<FRONTEND_PORT>;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /api/ {
        proxy_pass http://<APP_VM_IP>:<BACKEND_PORT>;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
~~~

Verify configuration:
~~~bash
sudo nginx -t
sudo systemctl reload nginx
~~~

## 7. First Deployment Validation

1. Configure required CI/CD variables in GitLab (e.g., `DEPLOY_BRANCH`, SSH keys, app IPs).
2. Push pipeline files to the deployment repository.
3. Trigger a merge to the designated `$DEPLOY_BRANCH`.
4. Monitor the pipeline stages: `build -> test -> push -> deploy -> notify`.
5. Verify on `<APP_VM>`:
   ~~~bash
   curl -f http://localhost:<FRONTEND_PORT>
   curl -f http://localhost:<BACKEND_PORT>
   ~~~
