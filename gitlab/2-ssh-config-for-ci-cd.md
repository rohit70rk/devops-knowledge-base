# SSH Hardening to deploy app on app-node in docker container for GitLab ci/cd

## 1. Create Deployment User → APP-VM

Login as the root user, then create a new user named `deployer` without a usable password login:
```bash
# Create the user without a usable password login
sudo adduser --disabled-password deployer
```

Add the new user to the `docker` group so it can manage containers:
```bash
# Add the user to the docker group so it can manage containers
sudo usermod -aG docker deployer
```
*Note: If there are no permission errors, the user is successfully added.*

Switch your terminal session to act as this new user and verify permissions:
```bash
# Switch your terminal session to act as this new user
sudo su - deployer

# Verify new user permission
newgrp docker
docker ps
```

## 2. Generate SSH Key → BUILD-VM

We generate the SSH key here because this is the machine that will deploy. Keep everything default or empty during creation:
```bash
# We generate the key here because this is the machine that will deploy, keep everything default or empty
ssh-keygen -t ed25519 -C "gitlab-deployer-key"
```
*Note: Do not enter a passphrase. Automation requires passwordless key authentication.*

Print out the contents of the public key (default location is `~/.ssh/id_ed25519.pub`) so you can copy it for the next step:
```bash
# File location -> default (~/.ssh/id_ed25519)
cat ~/.ssh/id_ed25519.pub
```

## 3. Install the "Lock" (Public Key) → APP-VM

Create the SSH directory for the deployer user:
```bash
# Create the SSH directory
mkdir -p /home/deployer/.ssh
```

Open the `authorized_keys` file and paste the Public Key generated on the BUILD-VM:
```bash
# Open the authorized_keys file
nano /home/deployer/.ssh/authorized_keys
# Paste the Public Key -> Which is created on the BUILD-VM
```

Set the correct ownership and permissions for the SSH directory and files to ensure SSH key authentication works:
```bash
# Without correct ownership, SSH key authentication WILL fail silently
chown -R deployer:deployer /home/deployer/.ssh

# Lock down the file
chmod 700 /home/deployer/.ssh
chmod 600 /home/deployer/.ssh/authorized_keys
```

## 4. TEST AGAIN → BUILD-VM

Test the SSH connection from the BUILD-VM to the APP-VM to ensure passwordless access is working:
```bash
# Test ssh
ssh deployer@192.168.1.10
```

## 5. Configuring Password Login for app-node Only → APP-VM

Open the SSH configuration file as an admin user on the app-node:
```bash
# Open the SSH configuration file as admin user -> app-node
sudo nano /etc/ssh/sshd_config
```

Configure SSH settings to enhance security and allow specific exceptions:
```sshconfig
# Find the PasswordAuthentication directive and set it to no
PasswordAuthentication no

# Enable key authentication
PubkeyAuthentication yes

# Disable root login by default
PermitRootLogin no

# Scroll to the very bottom of the file. You must use a Match block to create an exception
Match User app-node
     PasswordAuthentication yes
```

Save and exit the file (Ctrl+O, Enter, Ctrl+X), then restart the SSH service to apply the changes:
```bash
# Restart the SSH service to apply the changes
sudo systemctl restart ssh
```
