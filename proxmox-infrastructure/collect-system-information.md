# System Information Collection Scripts

This document provides two comprehensive shell script blocks designed to audit the state of your infrastructure. These scripts help in troubleshooting networking, resource allocation, and container runtime health.

## 1. Proxmox Host Information Audit

**Purpose:** Run this on the Proxmox Hypervisor (bare metal) to verify hardware health, Proxmox-specific services (`pve`), virtual bridge configurations, and the status of all hosted VMs/Containers.

```bash
# Gathers Hostname, OS details, and Proxmox VE version
echo "===== PROXMOX HOST INFORMATION =====" && \
echo "===== HOST =====" && hostname && hostnamectl && pveversion -v && echo && \

# Audits CPU architecture and Memory utilization
echo "===== CPU =====" && lscpu && echo && \
echo "===== MEMORY =====" && free -h && echo && \

# Checks physical storage, Proxmox storage pools, and mount points
echo "===== STORAGE =====" && lsblk && pvesm status && df -h && echo && \

# Reviews network interface files and current live routing/bridge states
echo "===== NETWORK CONFIG =====" && cat /etc/network/interfaces && echo && \
echo "===== NETWORK STATE =====" && ip a && ip route && bridge link && echo && \

# Checks for NAT masquerading and active firewall rules
echo "===== NAT RULES =====" && iptables -t nat -L -n -v && echo && \
echo "===== FIREWALL =====" && pve-firewall status && echo && \

# Lists all VMs and LXC Containers along with their specific resource configs
echo "===== VM LIST =====" && qm list && echo && \
echo "===== CONTAINER LIST =====" && pct list && echo && \
echo "===== VM CONFIGS =====" && for vm in $(qm list | awk 'NR>1 {print $1}'); do echo "--- VM $vm ---"; qm config $vm | grep -E "name|cores|memory|net|scsi|cpu|agent"; echo; done && \

# Verifies Bridge status, Cluster health, and required Kernel modules (KVM/Netfilter)
echo "===== BRIDGE INFO =====" && brctl show 2>/dev/null || bridge link && echo && \
echo "===== CLUSTER INFO =====" && (pvecm status || echo "No cluster") && echo && \
echo "===== KERNEL MODULES =====" && lsmod | grep -E "kvm|br_netfilter" && echo && \

# Confirms IPv4 forwarding (needed for NAT) and audits open listening ports
echo "===== SYSCTL =====" && sysctl net.ipv4.ip_forward 2>/dev/null && echo && \
echo "===== OPEN PORTS =====" && ss -tulnp | head -20
```

## VMs INFORMATION

Run this command block inside individual Virtual Machines (like APP-VM or BUILD-VM) to collect their state, networking, container runtime details, and firewall configurations:
```bash
# Identifies the VM and checks current time sync (critical for TLS/Certs)
echo "===== VMs INFORMATION=====" && \
echo "===== HOST =====" && hostname && hostnamectl && uname -a && echo && \
echo "===== TIME =====" && timedatectl && echo && \

# Checks CPU virtualization flags and ensures Swap is disabled for K8s stability
echo "===== CPU =====" && lscpu && nproc && (grep -E "vmx|svm" /proc/cpuinfo | head -1 || echo "No virtualization flag found") && echo && \
echo "===== MEMORY & SWAP =====" && free -h && swapon --show || echo "No active swap" && echo && \

# Audits local storage and networking (MTU, DNS resolution, and local Hosts file)
echo "===== DISK =====" && lsblk -o NAME,SIZE,TYPE,MOUNTPOINT && df -h && echo && \
echo "===== NETWORK =====" && ip -4 addr && ip route && ip link | grep mtu && echo && \
echo "===== DNS =====" && cat /etc/resolv.conf && (nslookup google.com || echo "DNS lookup failed") && echo && \
echo "===== HOSTS =====" && cat /etc/hosts && echo && \

# Tests connectivity to the gateway and internet, and checks for Docker bridge interfaces
echo "===== CONNECTIVITY =====" && (ping -c 2 10.10.10.1 || echo "Gateway unreachable") && (ping -c 2 8.8.8.8 || echo "Internet unreachable") && echo && \
echo "===== DOCKER NETWORKS =====" && (ip a | grep -E "docker|br-" || echo "No docker networks") && echo && \

# Verifies K8s networking requirements: br_netfilter, overlay modules, and sysctl bridges
echo "===== KERNEL MODULES =====" && (lsmod | grep -E "br_netfilter|overlay" || echo "Modules not loaded") && echo && \
echo "===== SYSCTL =====" && (sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward net.ipv4.conf.all.forwarding 2>/dev/null || echo "Sysctl values not set") && echo && \

# Detailed audit of the Container Runtime (Docker/Containerd) and Cgroup version
echo "===== CONTAINER RUNTIME =====" && \
(which docker || echo "docker: not found") && \
(which containerd || echo "containerd: not found") && \
(containerd --version 2>/dev/null || echo "containerd not installed") && \
(systemctl status containerd 2>/dev/null | head -10 || echo "containerd service not found") && \
(ls -l /run/containerd/containerd.sock 2>/dev/null || echo "containerd socket not found") && echo && \
echo "===== CGROUP =====" && (stat -fc %T /sys/fs/cgroup || echo "cgroup check failed") && echo && \

# Final checks for system persistence, identity, package versions, and firewall status
echo "===== SWAP PERSISTENCE =====" && (grep -i swap /etc/fstab || echo "No swap in fstab") && echo && \
echo "===== MACHINE ID =====" && (cat /etc/machine-id || echo "machine-id not found") && echo && \
echo "===== PACKAGES =====" && (dpkg -l | grep -E "kube|docker|containerd" || echo "No related packages installed") && echo && \
echo "===== FIREWALL =====" && (sudo ufw status || echo "ufw not available") && (sudo iptables -L -n | head -20 || echo "iptables access denied") && echo && \
echo "===== PORTS =====" && (ss -tulnp | head -20 || echo "port check failed")
```
