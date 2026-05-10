# Protect GitLab CI/CD Variables

CI/CD variables are the secure bridge that allows:

`GitLab Runner` ➜ `authenticate` ➜ `SSH` ➜ `APP-VM` ➜ `deploy containers`

Go to:
**GitLab → Your Project → Settings → CI/CD → Variables** (project level variable, other project inherits)

or

**GitLab → Your Group → Settings → CI/CD → Variables** (group level variable, all projects can inherit)

## 1. Core Deployment Variables

### 1. SSH_PRIVATE_KEY

Get the Private Key from the BUILD-VM:
```bash
# Get Private Key from BUILD-VM
cat ~/.ssh/id_ed25519
```

It should look like this:
```text
-----BEGIN OPENSSH PRIVATE KEY-----
...
-----END OPENSSH PRIVATE KEY-----
```

Enable the following settings for this variable:
- ✅ **Masked** (unselect if throwing error)
- ✅ **Protected**

Ensure:
- No extra spaces
- No missing lines

### 2. APP_VM_SSH_KNOWN_HOSTS

Generate the host key on the BUILD-VM for the APP-VM IP:
```bash
# Generate the host key on BUILD-VM
ssh-keyscan -H 192.168.1.10
```

Example Output:
```text
192.168.1.10 ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI...
192.168.1.10 ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQ...
192.168.1.10 ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTIt...
```
*Note: Copy and Paste ALL lines into the variable value.*

### 3. APP_VM_SSH_USER

Value: `deployer`

### 4. APP_VM_SSH_HOST_IP

Value: `192.168.1.10`

### 5. SSH_PORT

Value: `22`

### 6. BASE_DEPLOY_PATH

Value: `/opt/apps`

Ensure the directory exists and has the correct permissions on the APP-VM:
```bash
# create folder if not available
sudo mkdir -p /opt/apps

# allow access
sudo chown -R deployer:deployer /opt/apps
```

## You need different security levels for different variables

| Variable | Protected | Masked |
| :--- | :---: | :---: |
| `SSH_PRIVATE_KEY` | ✅ | ✅ |
| `SSH_HOST` | ✅ | ❌ |
| `SSH_USER` | ✅ | ❌ |
| `BASE_DEPLOY_PATH` | ❌ | ❌ |

## Who is allowed to USE variables in pipelines

| Role | Meaning |
| :--- | :--- |
| **No one allowed** | No pipeline can use variables |
| **Owner** | Only group owner can trigger pipelines with variables |
| **Maintainer** | Maintainers + Owner |
| **Developer** | Developers + Maintainers + Owner |
