# GitLab First App Deploy with CI/CD

## Step-1. Prerequisites You MUST Have (Per App)

### Required Networking Flow
```text
Internet
  ↓
DNS (Cloudflare / Registrar)
  ↓
PROXY-VM (Nginx :80/:443)
  ↓
APP-VM (container : Port)
```

### 1. DNS Record

You need a domain/subdomain for your app pointing to your public IP.
- `app.example.com` → `<your public IP>` → DNS only (⚪)

**Where to configure:**
Cloudflare / GoDaddy / Namecheap (wherever the domain is hosted)

### 2. On PROXY-VM Nginx Configuration

If using UFW, allow HTTP and HTTPS ports:
```bash
# If using UFW -> Allow port 80 & 443:
sudo ufw allow 80
sudo ufw allow 443
```

Open a new Nginx configuration file for your app:
```bash
sudo nano /etc/nginx/sites-available/app.example.com
```

Add the following reverse proxy configuration to forward traffic to the APP-VM:
```nginx
server {
  listen 80;
  server_name app.example.com;

  location / {
    proxy_pass http://192.168.1.10:5000;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

Enable the site configuration and reload Nginx:
```bash
# Enable it:
sudo ln -s /etc/nginx/sites-available/app.example.com /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### 3. Generate SSL Certificate

Create an SSL certificate using your preferred method (e.g., Certbot) and reconfigure the Nginx file to handle HTTPS requests.

Test and reload Nginx to apply changes:
```bash
sudo nginx -t
sudo systemctl reload nginx
```

### 4. Test DNS

Perform these tests to verify routing at each step:
- **Step 1 — Internal check (APP-VM):**
  `curl http://localhost:5000` (Must return HTML or response)
- **Step 2 — From PROXY-VM:**
  `curl http://192.168.1.10:5000` (Confirms network path)
- **Step 3 — From PROXY-VM via domain:**
  `curl http://app.example.com` (Confirms nginx routing)
- **Step 4 — From browser:**
  `http://app.example.com`
- **Step 5 — HTTPS:**
  `https://app.example.com`

### 5. Repo should have a stable dev branch with required files

Your repository must be CI/CD ready, not just code-ready.
- Should have a stable branch for deployment.
- Branch must have specific files (can be empty, edited by the devops team):
  - `Dockerfile`
  - `.dockerignore`
  - `.gitlab-ci.yml`

Directory structure at project root:
```text
project/
├── Dockerfile
├── .dockerignore
├── .gitlab-ci.yml
├── app/
└── . . .
```

## Step-2. Verify GitLab Variables & config.toml

Make sure these variables exist at the Group or Project Level in the GitLab GUI.
- **For Project-level variables:** GitLab → Project → Settings → CI/CD → Variables
- **For Group-level variables:** GitLab → Group → Settings → CI/CD → Variables

Required variables:
- `SSH_PRIVATE_KEY`
- `APP_VM_SSH_USER`
- `APP_VM_SSH_HOST_IP`
- `APP_VM_SSH_KNOWN_HOSTS`
- `SSH_PORT`
- `BASE_DEPLOY_PATH`

Ensure the GitLab Runner container can resolve the internal GitLab domain by editing `config.toml` on the BUILD-VM:
```bash
# Verify in /etc/gitlab-runner/config.toml -> if not available than add it -> BUILD-VM
sudo nano /etc/gitlab-runner/config.toml
```

Add the `extra_hosts` configuration to the runner:
```toml
extra_hosts = ["gitlab-ce.example.com:192.168.1.30", "registry-ce.example.com:192.168.1.30"]
```
*Note: This is required because the Docker executor does NOT use the host's `/etc/hosts` file.*

Restart the GitLab Runner service to apply the networking changes:
```bash
# Restart the GitLab Runner service to apply the networking changes:
sudo systemctl restart gitlab-runner
```

## Step-3. Edit .gitlab-ci.yml, Dockerfile and .dockerignore

A comprehensive `.gitlab-ci.yml` file to handle building, testing, pushing, and deploying your application:
```yaml
stages:
  - build
  - test
  - push
  - deploy

variables:
  IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

# ------------------------------------------------
# Build
# ------------------------------------------------
build:
  stage: build
  script:
    - docker build --provenance=false -t $IMAGE .

# ------------------------------------------------
# Test (BUILD-VM)
# ------------------------------------------------
test:
  stage: test
  script:
    - apk add --no-cache curl
    - docker rm -f test-container || true
    - docker run -d --name test-container $IMAGE

    - sleep 10

    - TEST_IP=$(docker inspect -f '{{ .NetworkSettings.IPAddress }}' test-container)
    - echo "Testing against internal container IP -> $TEST_IP:5000"
    - curl -f http://$TEST_IP:5000

    - docker rm -f test-container

# ------------------------------------------------
# Push (BUILD-VM Cleanup after successful push)
# ------------------------------------------------
push:
  stage: push
  script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login $CI_REGISTRY -u $CI_REGISTRY_USER --password-stdin
    - docker push $IMAGE
    - docker rmi $IMAGE || true

# ------------------------------------------------
# Deploy (APP-VM)
# ------------------------------------------------
deploy:
  stage: deploy
  only:
    - dev-test
  before_script:
    - mkdir -p ~/.ssh
    - echo "$APP_VM_SSH_KNOWN_HOSTS" > ~/.ssh/known_hosts
    - echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_ed25519
    - chmod 600 ~/.ssh/id_ed25519
  script:
    - |
      ssh -i ~/.ssh/id_ed25519 -p $SSH_PORT $APP_VM_SSH_USER@$APP_VM_SSH_HOST_IP "
            set -e

            cd \"$BASE_DEPLOY_PATH/johndoe-flask-portfolio\"

            CURRENT=current_image.txt
            PREVIOUS=previous_image.txt

            echo '📦 Saving current state...'
            [ -f \$CURRENT ] && cp \$CURRENT \$PREVIOUS

            echo '🚀 Deploying...'
            export IMAGE=$IMAGE
            docker compose pull
            docker compose up -d --force-recreate

            echo '🧪 Health check...'
            sleep 10

            if curl -f http://localhost:5000; then
              echo $IMAGE > \$CURRENT
              echo '    ✅ Deploy success'
              # Targeted APP-VM Cleanup (Keep Current & Previous)
              echo '    🧹 Cleaning up old images...'
              PREV_IMG=\$(cat \$PREVIOUS 2>/dev/null || echo \"\")

              for img in \$(docker images --format '{{.Repository}}:{{.Tag}}' | grep \"$CI_REGISTRY_IMAGE\" || true); do
                   if [ \"\$img\" != \"$IMAGE\" ] && [ \"\$img\" != \"\$PREV_IMG\" ]; then
                    echo \"   🗑️ Removing obsolete image: \$img\"
                    docker rmi \"\$img\" || true
                   fi
              done

            else
              echo '    ❌ Deploy failed → rollback'
              if [ -f \$PREVIOUS ]; then
                   export IMAGE=\$(cat \$PREVIOUS)
                   docker compose up -d --force-recreate
                   echo '   🔁 Rollback done'
              else
                echo '🚨 No rollback data'
              fi

              exit 1
            fi
        "
```

## Step-4. Prepare APP-VM

SSH into the APP-VM through the BUILD-VM to configure the deployment environment:
```bash
# SSH into APP-VM through BUILD-VM:
ssh deployer@192.168.1.10
```

Create the application directory if it is not available:
```bash
# Create app directory if not available
mkdir -p /opt/apps/johndoe-flask-portfolio
cd /opt/apps/johndoe-flask-portfolio
```

Create and define the `docker-compose.yml` file to handle container orchestration:
```bash
# Create docker-compose.yml
nano docker-compose.yml
```

Paste the following configuration:
```yaml
services:
  web:
    container_name: flask-portfolio
    image: ${IMAGE}
    ports:
      - "5000:5000"
    env_file:
      - .env
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:5000 || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s
    restart: unless-stopped
```

Create the environment file to store runtime variables securely:
```bash
# Add app level .ENV here
nano /opt/apps/johndoe-flask-portfolio/.env
# Paste .env variables
# Save and exit
```

Note that the following two files will be created automatically on the first CI/CD pipeline run to track deployments:
```bash
# This two file also get create on first ci/cd pipeline run
cat /opt/apps/johndoe-flask-portfolio/current_image.txt  # Store current image
cat /opt/apps/johndoe-flask-portfolio/previous_image.txt # Store previous image
```

## Step-5. Create Empty GitLab Repository

Before adding GitLab as a remote, create an empty repository inside your GitLab group.

### 1. Create New Project

Go to:

GitLab → Your Group → New Project → Create Blank Project

Fill:
- Project name → `ci-cd-test`
- Visibility → Private (recommended)

⚠️ Keep these unchecked:
- Initialize repository with README
- Add .gitignore
- Add License

Click:

✅ Create project

### 2. Copy Repository URL

Example:
```text
https://gitlab-ce.example.com/example-org/ci-cd-test.git
```

### 3. Update These Values

| Replace | With |
|---|---|
| `gitlab-ce.example.com` | Your GitLab domain |
| `example-org` | Your GitLab group name |
| `ci-cd-test` | Your repository name |


## Step-6. Connect GitHub → GitLab

### 📌 GitHub → GitLab Sync Decision
| Option | Reason Not Selected |
| :--- | :--- |
| **Pull Mirroring** | Not available in GitLab CE (requires Premium/Ultimate) |
| **GitHub Webhook → GitLab** | Triggers pipeline only, does not sync repo; adds complexity |
| **GitHub Actions Sync** | Extra CI layer, requires tokens, unnecessary complexity |

✅ **Selected Approach**
| Approach | Why Selected |
| :--- | :--- |
| **Dual Remote Push** | Simple, reliable, instant pipeline trigger, no extra config or dependencies |

**Workflow Flow:**

`Local Dev` → `Push → GitHub (backup/source)` → `Push → GitLab (CI/CD trigger)` → `Pipeline runs` → `Deploy to APP-VM`


1. **Get GitLab Repo URL**

   👉 Copy HTTPS URL → `https://gitlab-ce.example.com/example-org/ci-cd-test.git`

2. **Add GitLab as Remote**
```bash
git remote add gitlab https://gitlab-ce.example.com/example-org/ci-cd-test.git
```

3. **Verify Remotes**
```bash
git remote -v
```
Expected output:
```text
gitlab https://gitlab-ce.example.com/example-org/ci-cd-test.git (fetch)
gitlab https://gitlab-ce.example.com/example-org/ci-cd-test.git (push)
origin git@github.com:johndoe/Portfolio.git (fetch)
origin git@github.com:johndoe/Portfolio.git (push)
```

4. **Push to GitLab**
```bash
git push -u gitlab dev-test
```
*Note: Git will prompt for your GitLab username and a Personal Access Token (PAT) as the password.*

**How to Create GitLab PAT (Do this once):**
- Go to: 👉 GitLab → Profile → Access Tokens
- Create token with: 
  - ✅ `read_repository`
  - ✅ `write_repository`
  - ✅ `api` (optional but safe)
- 👉 Copy the token to share with the team or to use

To avoid entering the token every time, configure credential helper:
```bash
# Avoids entering token every time
git config credential.helper store
```

5. **Daily Workflow (Super Simple)**
Whenever you push changes, update both remotes:
```bash
git push -u origin dev-test   # GitHub
git push -u gitlab dev-test   # GitLab (triggers CI)
```

## Step-7. MANUAL CI PIPELINE TRIGGER

- Go to: **GitLab → Project → Build → Pipelines**
- Click: **New pipeline**
- Select branch: `dev-test`
- Click: **Run pipeline**

✅ **Result**
Pipeline starts immediately in the following sequence: `build → test → push → deploy`

## Step-8. MANUAL ROLLBACK COMMAND

To manually rollback to the previous version from the APP-VM:
```bash
cd /opt/apps/johndoe-flask-portfolio

# Check available versions
cat current_image.txt
cat previous_image.txt

# Rollback
export IMAGE=$(cat previous_image.txt) && docker compose up -d --force-recreate
```

## Step-9. Watch Pipeline

Monitor the execution of your CI/CD flow from the web UI:
**Project → Build → Pipelines**

**Expected flow:**
Build ✅ → Test ✅ → Push ✅ → Deploy ✅
