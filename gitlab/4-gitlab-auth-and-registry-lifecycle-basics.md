# Secure authentication for APP-VM and Full registry lifecycle (system-level design)

## Prerequisites

Check the `/etc/hosts` file on the app-node to ensure correct DNS resolution:
```bash
app-node@app-node:~$ cat /etc/hosts
```

It should look something like this:
```text
127.0.0.1 localhost
127.0.1.1 app-node
192.168.1.30 gitlab-ce.example.com
192.168.1.30 gitlab-ce-registry.example.com

# The following lines are desirable for IPv6 capable hosts
::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

Verify DNS Resolution from the app-node:
```bash
# Verify DNS Resolution
ping gitlab-ce-registry.example.com
```

Example output:
```text
PING gitlab-ce-registry.example.com (192.168.1.30) 56(84) bytes of data.
...
```

## STEP 1 — Create Deploy Token

Create a deploy token either at the Group level (valid for multiple projects) or at the Project level.
- **Valid for Multiple project**: Group → Settings → Repository → Deploy Tokens
- **Valid Only One project**: Project → Settings → Repository → Deploy Tokens

| Field | Value |
| :--- | :--- |
| **Name** | `app-vm-registry-pull` |
| **Expiration** | (optional) |
| **Username** | auto-generated |
| **Scopes** | ✅ `read_registry` **ONLY** |

👉 **Copy username & password immediately (you will NOT see again)**

## STEP 2 — Authenticate APP-VM

SSH into the APP-VM as the `deployer` user:
```bash
# SSH into APP-VM
ssh deployer@192.168.1.10
```

Logout from any existing Docker registry sessions to ensure a clean state:
```bash
# Logout admin/root user if you login already
docker logout gitlab-ce-registry.example.com
```

Verify you are acting as the correct user before logging in:
```bash
# Verify User
whoami
```

Log in to the GitLab registry using the deploy token:
```bash
# Run Docker Login
docker login gitlab-ce-registry.example.com
```

When prompted, enter:
- **Username** → deploy token username
- **Password** → deploy token password

**Expected Output**:
```text
Login Succeeded
```

Verify that Docker has stored the credentials securely:
```bash
# Verify Credential Storage
cat ~/.docker/config.json
```

You should see an output like this:
```json
{
    "auths": {
        "gitlab-ce-registry.example.com": {
            "auth": "xxxxx"
        }
    }
}
```
*Note: This file is what allows the CI process to ssh into APP-VM and run `docker pull` WITHOUT prompting.*

## STEP 3 — HARD VALIDATION

### 1. SSH into BUILD-VM:

Pull a base image from Docker Hub to test:
```bash
# Pull a base image
docker pull nginx
```

Tag the image for your custom GitLab registry (replace with your actual project path):
```bash
# Replace with your actual project path:
docker tag nginx gitlab-ce-registry.example.com/example-org/registry-test:manual-test
```

Push the tagged image to your GitLab registry:
```bash
# Push it
docker push gitlab-ce-registry.example.com/example-org/registry-test:manual-test
```

### 2. Now go to APP-VM through BUILD-VM:

SSH into the APP-VM from the BUILD-VM:
```bash
ssh deployer@192.168.1.10
```

Pull the image you just pushed:
```bash
# Pull the image
docker pull gitlab-ce-registry.example.com/example-org/registry-test:manual-test
```

**Expected Result**:
```text
Status: Downloaded newer image for gitlab-ce-registry.example.com/example-org/registry-test:manual-test
```

Confirm that the image exists locally on the APP-VM:
```bash
# Confirms image actually exists locally
docker images | grep manual-test
```

## STEP 4 — Registry CI/CD Governance

### 1. Image Tagging Strategy

You must implement this pattern for tagging container images:
```bash
# Every pipeline build produces:
$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
```

Example output:
```text
registry-ce.example.com/devops/myapp:3f2a9c1
```

### 2. Registry Cleanup Policy

Navigate to **Project → Settings → Packages & Registries → Container Registry**.

| Setting | Value |
| :--- | :--- |
| **Keep last** | `10–20` |
| **Remove untagged** | ✅ |
| **Cleanup schedule** | `daily` |

**What this does**:
- **Keeps**: SHA images
- **Removes**: dangling and broken builds

👉 **Ensure**: Cleanup policy is actually ENABLED + scheduled.

GitLab requires:
- toggle **ON**
- schedule set (**daily**)

*Otherwise: config exists but does nothing.*

### 3. CI Registry Authentication Model (BUILD SIDE)

GitLab automatically provides these CI/CD variables:
```bash
$CI_REGISTRY
$CI_REGISTRY_USER
$CI_REGISTRY_PASSWORD
```

Use these variables to log in to the Docker registry within the CI pipeline:
```bash
# Login in CI
docker login $CI_REGISTRY \
  -u $CI_REGISTRY_USER \
  -p $CI_REGISTRY_PASSWORD
```

### APP-VM (Deploy side) - 👉 Deploy Token (`read_registry` ONLY)
**🔥 Production Rule**

| Access | Method |
| :--- | :--- |
| **Push** | CI only |
| **Pull** | deploy token |
| **Admin** | NEVER in automation |
