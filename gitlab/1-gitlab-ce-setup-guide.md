# GitLab CE Setup Guide

## Step 1: Prerequisites
*Reference: [GitLab official documentation](https://docs.gitlab.com/install/package/ubuntu/?tab=Community+Edition)*

Enable SSH and open firewall ports:

**PROXY-VM:**
- 80, 443 (public)

**BUILD-VM:**
- 22 (public if needed)
- 80, 5050 (internal only)

Enable and start the SSH service on the BUILD-VM:
```bash
sudo systemctl enable --now ssh
```

Set up your DNS: `https://gitlab-ce.example.com` and `https://gitlab-ce-registry.example.com`

### Only if you are using Proxy-Node
**On the PROXY-VM (Nginx):**

Configure Nginx to listen on port 443 with your SSL certificates, and `proxy_pass` the traffic to the internal IP of the BUILD-VM. Crucially, must forward the headers so GitLab knows the original request was secure and knows the real IP of the user.

Open the Nginx configuration for GitLab:
```bash
nano /etc/nginx/sites-available/gitlab
```

Add the following configuration to handle HTTP to HTTPS redirection, the GitLab UI, and the Docker Registry:
```nginx
server {
  listen 80;
  server_name gitlab-ce.example.com gitlab-ce-registry.example.com;

  return 301 https://$host$request_uri;
}

# GitLab UI
server {
  listen 443 ssl;
  server_name gitlab-ce.example.com;

  client_max_body_size 0;

  ssl_certificate /path/to/your/fullchain.pem;
  ssl_certificate_key /path/to/your/privkey.pem;

  location / {
    proxy_pass http://192.168.1.50:80;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header X-Forwarded-Ssl on;
    proxy_set_header X-Forwarded-Host $host;

    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

    proxy_read_timeout 300;
    proxy_buffering off;
  }
}

# Registry
server {
  listen 443 ssl;
  server_name gitlab-ce-registry.example.com;

  client_max_body_size 2G;

  ssl_certificate /path/to/your/fullchain.pem;
  ssl_certificate_key /path/to/your/privkey.pem;

  location / {
    proxy_pass http://192.168.1.50:5050;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header X-Forwarded-Ssl on;
    proxy_set_header X-Forwarded-Host $host;

    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

    proxy_read_timeout 300;
    proxy_buffering off;
  }
}
```

Enable the Nginx site, test the configuration, and restart Nginx:
```bash
# Generate SSL Certificate Manually (if not already done)
sudo ln -s /etc/nginx/sites-available/gitlab /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

## Step 2: Install Dependencies

Update the package list and install required dependencies for GitLab CE:
```bash
sudo apt-get update
sudo apt-get install -y curl openssh-server ca-certificates tzdata perl
```
*Note: If prompted to configure Postfix for email, select "Internet Site" or "No configuration" if you plan to use an external SMTP provider later.*

## Step 3: Add the GitLab CE Repository and Install

Download and execute the GitLab CE repository setup script:
```bash
curl --location "https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh" | sudo bash
```

Install GitLab CE with the specified external URL:
```bash
sudo EXTERNAL_URL="http://gitlab-ce.example.com" apt-get install gitlab-ce
```

## Step 4: Apply the Proxy and Registry configurations
*Reference: [GitLab official documentation](https://docs.gitlab.com/administration/packages/container_registry/#configure-container-registry-under-an-existing-gitlab-domain)*

Open the GitLab configuration file to apply settings for the proxy and registry:
```bash
sudo nano /etc/gitlab/gitlab.rb
```

Append or find them to add/edit these configurations in `gitlab.rb`:
```ruby
# Disable internal SSL (handled by PROXY-VM)
letsencrypt['enable'] = false

# External URLs (what users and runners access)
external_url 'https://gitlab-ce.example.com'
registry_external_url 'https://gitlab-ce-registry.example.com'

# Enable Container Registry
gitlab_rails['registry_enabled'] = true
gitlab_rails['registry_api_url'] = "http://127.0.0.1:5000"

# Internal GitLab Nginx (HTTP only - behind proxy)
nginx['listen_port'] = 80
nginx['listen_https'] = false

# Internal Registry Nginx (HTTP only - behind proxy)
registry_nginx['listen_port'] = 5050
registry_nginx['listen_https'] = false

# Trust PROXY-VM (real client IP handling)
gitlab_rails['trusted_proxies'] = ['192.168.1.30']
nginx['real_ip_trusted_addresses'] = ['192.168.1.30']
nginx['real_ip_header'] = 'X-Real-IP'

# Default project settings
gitlab_rails['gitlab_default_projects_features_container_registry'] = true

# Artifacts configuration
gitlab_rails['artifacts_enabled'] = true
gitlab_rails['artifacts_expire_in'] = '7 days'

# SSH configuration
gitlab_rails['gitlab_shell_ssh_port'] = 22
```

Reconfigure GitLab to apply the changes:
```bash
sudo gitlab-ctl reconfigure
```

Verify the overall status and specifically the registry component:
```bash
# Verify overall status
sudo gitlab-ctl status

# Verify registry status
sudo gitlab-ctl status | grep registry
```

Check the available memory to ensure the system is not starved of RAM after installation. Consider restarting the VM if RAM is full:
```bash
# Verify the free space and Restart the vm to remove/clear the vm RAM filled while installation
free -h
```

## Step 5: Initial sign-in

After GitLab is installed, go to the URL (`https://gitlab-ce.example.com`) you set up and use the following credentials to sign in:
- **Username:** `root`
- **Password:** See `/etc/gitlab/initial_root_password`

Retrieve the initial root password:
```bash
sudo cat /etc/gitlab/initial_root_password
```
*Note: After signing in, change your password, email address and configure important settings.*

Before attempting a GitLab CE Registry login, update the hosts file to route internal requests:
```bash
sudo nano /etc/hosts
```

Add this line to the file (right below the `127.0.0.1 localhost` line) to enable internal routing:
```text
192.168.1.30 gitlab-ce.example.com gitlab-ce-registry.example.com
```

Verify the GitLab Registry by logging in via Docker:
```bash
docker login gitlab-ce-registry.example.com
```

## Step 6: Install & Configure GitLab Runner (BUILD-VM)
*Reference: [GitLab official documentation](https://docs.gitlab.com/runner/install/linux-repository/?tab=Debian%2FUbuntu%2FMint)*

Install the Runner directly on the OS and wire it to your existing Docker engine to compile your images.

### 1. Add the Runner repository and install:

Download the repository script for GitLab Runner:
```bash
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" -o script.deb.sh
```

Verify the script contents and run it:
```bash
# Verify
less script.deb.sh

# Run the script
sudo bash script.deb.sh
```

Install the GitLab Runner package:
```bash
sudo apt update
sudo apt install gitlab-runner -y
```

Verify the installation and check the service status:
```bash
gitlab-runner --version

# Check service status
sudo systemctl status gitlab-runner
```

Start and enable the runner service, add it to the Docker group, and restart it:
```bash
# Start & enable if needed
sudo systemctl enable gitlab-runner

sudo usermod -aG docker gitlab-runner

sudo systemctl restart gitlab-runner
```

### 2. Create the Runner in the GitLab UI
*Reference: [GitLab official documentation](https://docs.gitlab.com/runner/register/)*
- Open `https://gitlab-ce.example.com` as administrator.
- Go to **Admin Area > CI/CD > Runners**.
- Click **New instance runner**.
- Apply any relevant tags (e.g., `docker`, `build-vm`), check **Run untagged jobs**, and click **Create runner** or config according to requirement.
- Copy the generated Runner Authentication Token (it usually begins with `glrt-`).

### 3. Register the Runner via CLI

Register the runner with the GitLab instance using your copied token:
```bash
sudo gitlab-runner register \
  --url "https://gitlab-ce.example.com" \
  --token "<YOUR_glrt_TOKEN>"
```

You will be prompted to configure the runner:
- Enter the GitLab instance URL (for example, `https://gitlab.com/`): press enter for default
- Enter a name for the runner. This is stored only in the local config.toml file:
- Enter an executor: Type `docker`.
- Enter the default Docker image: Type `docker:24.0.5` (or `ubuntu:24.04` depending on your baseline preference).

### 4. Optimize Configuration (Best Practices)

Open the runner's global configuration file:
```bash
sudo nano /etc/gitlab-runner/config.toml
```

Locate and make the following specific adjustments:
```toml
# The concurrent value is a global limit that dictates the maximum number of CI/CD jobs the GitLab
# Runner will execute simultaneously across all registered projects.

concurrent = 3

# Find [runners.docker] section.

[runners.docker]
   tls_verify = false
   image = "docker:24.0.5"
   privileged = false
   disable_entrypoint_overwrite = false
   oom_kill_disable = false
   disable_cache = false
   pull_policy = "if-not-present"
   volumes = ["/var/run/docker.sock:/var/run/docker.sock", "/cache"]
   shm_size = 0
```

Save the file and restart the runner to apply the changes:
```bash
# Save the file and restart the runner to apply the changes:
sudo systemctl restart gitlab-runner
```

Test Docker connectivity for the GitLab Runner user:
```bash
# If that command returns the Docker system information without a permission error, your CI/CD powerhouse is 100% ready.
sudo -u gitlab-runner docker info
```
