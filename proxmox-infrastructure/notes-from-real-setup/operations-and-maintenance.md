# Operations & Maintenance Runbook

This document details the standard operational procedures, access management, and maintenance tasks for the Proxmox infrastructure and its VMs.

## Access & Credentials Management

### Network Isolation
All VMs reside on a private subnet (e.g., `10.10.10.0/24`) and are not directly exposed to the internet. 

Accessing a VM requires one of the following:
1. **VPN Connection** to the internal network.
2. **Proxmox Web UI Console** (NoVNC / Xterm.js).
3. **SSH Jump Host** routing through the Proxmox host.

### SSH Jump Host Configuration Example
To connect to internal VMs via the Proxmox host, add the following to your local `~/.ssh/config`:

```text
Host pve
    HostName <PUBLIC_IP>
    User root
    Port <CUSTOM_SSH_PORT>

Host app-vm
    HostName <APP_VM_IP>
    User <APP_USER>
    ProxyJump pve

Host db-vm
    HostName <DB_VM_IP>
    User <DB_USER>
    ProxyJump pve
```
*Usage:* `ssh app-vm` will now securely tunnel through the Proxmox host.

### Service Accounts matrix
Use dedicated, unprivileged accounts for automated tasks:
- **CI/CD Deployment:** `deployer` user on the `<APP_VM>`, accessible via SSH keys restricted to the `<BUILD_VM>`.
- **External CI (e.g., GitHub Actions):** `github-deploy` user on the `<APP_VM>`, accessible via SSH keys stored in external secrets.

---

## Maintenance Runbook

### 1. Restarting Virtual Machines
Use the `qm` CLI on the Proxmox host for VM lifecycle management:

```bash
# Graceful shutdown:
qm shutdown <VMID>
# Start VM:
qm start <VMID>
# Hard stop (force):
qm stop <VMID>
```

### 2. Restarting Application Containers
All applications are containerized. Connect to the respective VM and use Docker Compose:

```bash
ssh <USER>@<VM_IP>
cd /path/to/app/directory/
sudo docker compose restart <service-name>

# To view logs:
sudo docker compose logs -f --tail=100 <service-name>

# Full recreate (if updating environment variables or images):
sudo docker compose down && sudo docker compose up -d
```

### 3. Database Backup Procedures
Regular database backups are critical, especially if automated schedules are not yet configured.

**PostgreSQL Backup Example (`<DB_VM>`):**
```bash
sudo docker exec <postgres-container> pg_dump -U <username> <dbname> > ~/databases/<db-folder>/backup-$(date +%Y%m%d).dump
```

**MongoDB Backup Example (`<DB_VM>`):**
```bash
sudo docker exec <mongo-container> mongodump --out /data/backup
sudo docker cp <mongo-container>:/data/backup ~/databases/<db-folder>/backup-$(date +%Y%m%d)
```

### 4. Disk Space Management
Monitoring disk space is crucial for thin-provisioned VMs and small proxy instances.

**Check all VMs from the Proxmox host:**
```bash
for vm in 100 101 102 103 104; do
  echo "=== VM $vm ==="
  qm guest exec $vm -- df -h /
done
```

**Clean up Docker (`<APP_VM>`, `<BUILD_VM>`):**
```bash
# Remove dangling images, stopped containers, and unused volumes
sudo docker system prune -a --volumes 
```

**Clean up System Logs (`<PROXY_VM>`):**
```bash
sudo journalctl --vacuum-size=50M
sudo apt clean && sudo apt autoremove -y
```

### 5. Managing Proxy Configurations (`<PROXY_VM>`)

**Nginx (Web Traffic):**
```bash
sudo nginx -t                          # Test configuration syntax
sudo systemctl reload nginx            # Zero-downtime reload
sudo tail -f /var/log/nginx/error.log  # Tail error logs
```

**HAProxy (Database / TCP Traffic):**
```bash
haproxy -c -f /etc/haproxy/haproxy.cfg # Test configuration syntax
sudo systemctl restart haproxy         # Restart service
```

---

## Common Troubleshooting

### Troubleshooting Application Flow (e.g. Connection Refused)
If an application is unreachable, trace the path from the outside in:

1. **Check the Proxmox Host NAT:** Verify iptables is routing the port.
   ```bash
   iptables -t nat -L PREROUTING -n -v | grep <PORT>
   ```
2. **Check the Proxy (`<PROXY_VM>`):** Is Nginx/HAProxy running and listening?
   ```bash
   sudo systemctl status nginx
   ```
3. **Check the Application (`<APP_VM>` or `<DB_VM>`):** Is the container actually running and healthy?
   ```bash
   sudo docker ps | grep <app-name>
   sudo docker logs <container_id> --tail=50
   ```

### SSL Certificate Expiry (Let's Encrypt / Certbot)
If using DNS challenge validation for wildcard or internal domains, automated renewals may fail unless hooked into a DNS API. 
To manually renew on the `<PROXY_VM>`:
```bash
sudo certbot certonly --manual --preferred-challenges dns -d <domain>
# Follow the prompts to add the required TXT records to your DNS provider.
sudo systemctl reload nginx
```
