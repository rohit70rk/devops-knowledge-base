# Centralized, reusable email notification system across all projects

**Architecture:**
```text
GitLab Group
 ↓
Shared CI/CD Template
 ↓
Shared Email Script
 ↓
All Projects Inherit It
```

## Step 1 — SMTP Setup for GitLab Level Notification

### 1. Open GitLab Config → On BUILD-VM

Open the main configuration file for GitLab:
```bash
sudo nano /etc/gitlab/gitlab.rb
```

### 2. Add SMTP Config

Append or modify the SMTP settings to enable global notifications through a central email provider:
```ruby
gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "mail.example.com"
gitlab_rails['smtp_port'] = 587
gitlab_rails['smtp_user_name'] = "devops@example-node.tech"
gitlab_rails['smtp_password'] = "YOUR_PASSWORD"
gitlab_rails['smtp_domain'] = "example-node.tech"
gitlab_rails['smtp_authentication'] = "login"
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['smtp_tls'] = false

gitlab_rails['gitlab_email_from'] = "devops@example-node.tech"
gitlab_rails['gitlab_email_display_name'] = "Banao GitLab"
gitlab_rails['gitlab_email_reply_to'] = "devops@example-node.tech"
```

### 3. Apply Configuration

Reconfigure and restart GitLab to apply the SMTP changes:
```bash
sudo gitlab-ctl reconfigure
sudo gitlab-ctl restart
```

### 4. Test SMTP (VERY IMPORTANT)

Launch the GitLab Rails console to send a test email:
```bash
sudo gitlab-rails console

# Then run the below command
Notify.test_email('johndoe@example.com', 'SMTP Test', 'Hello from GitLab').deliver_now

# To exit type
exit
```

---

**→ Disable if you don't want to receive email for all CI/CD pipelines from GUI**

- 🔧 **STEP 1 — Open Notification Settings:** Go to 👉 GitLab → Top Right Avatar → Preferences
- 🔧 **STEP 2 — Open Notifications Tab:** Click 👉 Notifications
- 🔧 **STEP 3 — Change Global Level:** Set Notification level → Custom
- 🔧 **STEP 4 — Disable ONLY Pipeline Emails:** Scroll down and UNSELECT these options:
  - ❌ Pipeline failed
  - ❌ Pipeline fixed
  - ❌ Pipeline status changed
- 🔧 **STEP 5 — Save Changes:** Click 👉 Save changes

## STEP 2 — Add Group-Level Variables

Navigate to your group's CI/CD variables section:

👉 **GitLab → Group → Settings → CI/CD → Variables**

Add these as default fallbacks for ALL projects:
```text
EMAIL_TO=devops@example-node.tech
SMTP_PASSWORD=example_password_123
```

## STEP 3 — Create Shared CI Template - in that group

Create a new repository named `infra-templates/`.
On the right side panel, click ➕ **New file** and create `email-notification.yml`.

Provide the centralized template content:
```yaml
# Centralized Email Notification Template
# ------------------------------------------------

.email_notify:
  image: alpine:3.20

  before_script:
    - apk add --no-cache msmtp ca-certificates tzdata openssh-client

    - |
      mkdir -p ~/.ssh
      echo "$APP_VM_SSH_KNOWN_HOSTS" > ~/.ssh/known_hosts
      echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_ed25519
      chmod 600 ~/.ssh/id_ed25519

    - |
      cat > ~/.msmtprc <<EOF
      defaults
      auth                    on
      tls                     on
      tls_trust_file /etc/ssl/certs/ca-certificates.crt
      logfile                 ~/.msmtp.log

      account default
      host mail.example.com
      port 587
      from devops@example-node.tech
      user devops@example-node.tech
      password $SMTP_PASSWORD
      EOF

    - chmod 600 ~/.msmtprc

  script:
    - |
      set -e

      PROJECT=$CI_PROJECT_NAME
      BRANCH=$CI_COMMIT_REF_NAME
      COMMIT=$CI_COMMIT_SHORT_SHA
      PIPELINE_URL=$CI_PIPELINE_URL
      EMAIL=$EMAIL_TO
      TIME=$(TZ='Asia/Kolkata' date '+%d-%m-%Y %I:%M:%S %p IST')

      FAILED_STAGE=$(echo "${FAILED_STAGE:-UNKNOWN}" | tr '[:lower:]' '[:upper:]')

      IMAGE=$(ssh -i ~/.ssh/id_ed25519 -p $SSH_PORT \
        $APP_VM_SSH_USER@$APP_VM_SSH_HOST_IP \
        "cat $BASE_DEPLOY_PATH/$APP_NAME/current_image.txt 2>/dev/null" \
        || echo "UNKNOWN")

      STATUS=${STATUS:-PIPELINE_FAILED}

      if [ "$STATUS" = "SUCCESS" ]; then
        SUBJECT="[SUCCESS ✅ DEPLOY] $PROJECT"
        BODY="Deployment Successful ✅

         Project : $PROJECT
         Branch   : $BRANCH
         Commit   : $COMMIT
         Time     : $TIME

         Deployed Image : $IMAGE
         Pipeline           : $PIPELINE_URL"

      elif [ "$STATUS" = "ROLLBACK" ]; then
        SUBJECT="[FAILED ⚠️ ROLLBACK OK] $PROJECT"
        BODY="Deployment Failed ❌
          Rollback Successful 🔁

         Project : $PROJECT
         Branch   : $BRANCH
         Commit   : $COMMIT
         Time     : $TIME

         System has been restored to previous working version.

         Deployed Image : $IMAGE
         Pipeline           : $PIPELINE_URL"

      elif [ "$STATUS" = "PIPELINE_FAILED" ]; then
        SUBJECT="[FAILED    ❌ $FAILED_STAGE] $PROJECT"
        BODY="$FAILED_STAGE Stage Failed   ❌
         Project : $PROJECT
         Branch   : $BRANCH
         Commit   : $COMMIT
         Stage    : $FAILED_STAGE
         Time        : $TIME

         Failure occurred in the $FAILED_STAGE stage before deployment.
         No changes were applied to the live system.

         Deployed Image : $IMAGE
         Pipeline               : $PIPELINE_URL"

      else
         SUBJECT="[CRITICAL 🚨 DEPLOY] $PROJECT"
         BODY="DEPLOY Stage Failed ❌
           Rollback Failed 🚨

          Project : $PROJECT
          Branch      : $BRANCH
          Commit      : $COMMIT
          Time        : $TIME

          System may be DOWN. Immediate action required.

          Deployed Image : $IMAGE
          Pipeline               : $PIPELINE_URL"
      fi

      BODY=$(printf "%s" "$BODY" | sed 's/^[ \t]*//')

      printf "From: Example-Org CI/CD <devops@example-node.tech>\nTo: %s\nSubject: %s\nMIME-Version: 1.0\nContent-Type: text/html; charset=utf-8\n\n<div style=\"font-family: ui-monospace, 'Courier New', Consolas, monospace; background-color: #f8f9fa; border: 1px solid #dadce0; border-radius: 6px; padding: 20px; white-space: pre-wrap; font-size: 14px; color: #000000; line-height: 1.5;\">\n%s\n</div>\n" "$EMAIL" "$SUBJECT" "$BODY" | msmtp -t
```

## STEP 4 — Use Template in ALL Projects

### 1. Per-Project Customization (Very Important)

Define custom variables at the Project level under **Project → CI/CD Variables**:
```text
EMAIL_TO=johndoe@example.com
APP_NAME=johndoe-flask-portfolio
```

### 2. In each project update gitlab-ci.yml:

Include the centralized template and configure stages to inherit from `.track_stage`:
```yaml
include:
  - project: "example-org/infra-templates"
    ref: "main"
    file: "/email-notification.yml"

stages:
  - build
  - test
  - push
  - deploy
  - notify

variables:
  IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

.track_stage:
  after_script:
    - |
      touch stage.env
      if [ "$CI_JOB_STATUS" = "failed" ]; then
            echo "FAILED_STAGE=$CI_JOB_STAGE" >> stage.env
      fi
  artifacts:
    when: always
    reports:
      dotenv: stage.env

# ------------------------------------------------
# Build
# ------------------------------------------------
build:
  extends: .track_stage
  stage: build
  script:
    - docker build --provenance=false -t $IMAGE .

# ------------------------------------------------
# Test (BUILD-VM)
# ------------------------------------------------
test:
  extends: .track_stage
  stage: test
  script:
    - apk add --no-cache curl
    - docker rm -f test-container || true
    - docker run -d --name test-container $IMAGE
    - sleep 10
    - |
      trap "docker rm -f test-container || true" EXIT

      sleep 10

      TEST_IP=$(docker inspect -f '{{ .NetworkSettings.IPAddress }}' test-container)
      echo "Testing against internal container IP -> $TEST_IP:5000"

      for i in 1 2 3; do
       curl -f http://$TEST_IP:5000 && exit 0
       sleep 3
      done

      echo "App failed"
      exit 1

# ------------------------------------------------
# Push (BUILD-VM Cleanup after successful push)
# ------------------------------------------------
push:
  extends: .track_stage
  stage: push
  script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login $CI_REGISTRY -u $CI_REGISTRY_USER --password-stdin
    - docker push $IMAGE
    - |
      IMAGES=$(docker images "$CI_REGISTRY_IMAGE" --format '{{.Repository}}:{{.Tag}}' | grep -v "$IMAGE" || true)
      [ -n "$IMAGES" ] && docker rmi $IMAGES || true

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
      STATUS=$(ssh -i ~/.ssh/id_ed25519 -p $SSH_PORT $APP_VM_SSH_USER@$APP_VM_SSH_HOST_IP "
           set -e

           cd \"$BASE_DEPLOY_PATH/$APP_NAME\"

           CURRENT=current_image.txt
           PREVIOUS=previous_image.txt

           echo '  📦 Saving current state...'
           [ -f \$CURRENT ] && cp \$CURRENT \$PREVIOUS

           echo '  🚀 Deploying...'
           export IMAGE=$IMAGE
           docker compose pull
           docker compose up -d --force-recreate

           echo '  🧪 Health check...'
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

             echo SUCCESS

           else
             echo '    ❌ Deploy failed → rollback'
                 if [ -f \$PREVIOUS ]; then
                      export IMAGE=\$(cat \$PREVIOUS)
                      docker compose up -d --force-recreate
                      echo '   🔁 Rollback done'
                      echo ROLLBACK
                 else
                      echo '   🚨 No rollback data'
                      echo FAILED
                 fi
            fi
            " | tail -n 1)

    - echo "STATUS=$STATUS" >> deploy.env

    - |
      if [ "$STATUS" != "SUCCESS" ]; then
           exit 1
      fi
  after_script:
    - |
      if [ "$CI_JOB_STATUS" = "failed" ]; then
            if ! grep -q "FAILED_STAGE=" deploy.env 2>/dev/null; then
                 echo "FAILED_STAGE=$CI_JOB_STAGE" >> deploy.env
            fi
      fi
  artifacts:
    when: always
    reports:
      dotenv: deploy.env

# ------------------------------------------------
# Email Notifications
# ------------------------------------------------
email:
  extends: .email_notify
  stage: notify
  needs:
    - job: build
      optional: true
    - job: test
      optional: true
    - job: push
      optional: true
    - job: deploy
      optional: true
  when: always
```
