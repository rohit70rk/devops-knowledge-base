# Proxmox Host Configuration

This document covers the essential configuration required at the Proxmox host level to support the network architecture and VM deployments.

## Storage Configuration

A standard deployment uses LVM-Thin for VM disks to allow over-provisioning and snapshots.

### Typical Block Device Layout
```
/dev/sda (SSD/NVMe)
├── sda1    BIOS boot
├── sda2    /boot/efi
└── sda3    LVM (pve)
    ├── pve-swap    [SWAP]
    ├── pve-root    / (root filesystem)
    └── pve-data    LVM thin pool (VM storage)
```

### Proxmox Storage Pools
- `local` (Directory): Stores ISO images and container templates.
- `local-lvm` (LVM-Thin): Stores all VM disks (`vm-XXX-disk-1`).

## Network Interfaces (`/etc/network/interfaces`)

The host must define the public internet-facing bridge (`vmbr0`) and the private internal bridge (`vmbr1`).

### Example Configuration

```text
auto lo
iface lo inet loopback

# Physical Interface
auto nic0
iface nic0 inet manual

# Public Bridge
auto vmbr0
iface vmbr0 inet static
        address <PUBLIC_IP>/<CIDR>
        gateway <PUBLIC_GATEWAY>
        bridge-ports nic0
        bridge-stp off
        bridge-fd 0
        # NAT masquerade rule for outbound internet from private subnet
        post-up   iptables -t nat -A POSTROUTING -s <PRIVATE_SUBNET>/24 -o vmbr0 -j MASQUERADE
        post-down iptables -t nat -D POSTROUTING -s <PRIVATE_SUBNET>/24 -o vmbr0 -j MASQUERADE

# Private Bridge (Internal only)
auto vmbr1
iface vmbr1 inet static
        address <PRIVATE_GATEWAY_IP>/24
        bridge-ports none
        bridge-stp off
        bridge-fd 0
        # Ensure IP forwarding is enabled
        post-up echo 1 > /proc/sys/net/ipv4/ip_forward

source /etc/network/interfaces.d/*
```

## IP Forwarding

To allow the Proxmox host to act as a router for the internal VMs, IPv4 forwarding must be enabled. The `post-up` hook on `vmbr1` handles this dynamically, but it should also be set persistently:

```text
# /etc/sysctl.d/99-routing.conf
net.ipv4.ip_forward = 1
```

## Security & SSH

### SSH Configuration
- Consider changing the SSH port from `22` to a non-standard port (e.g., `5920`) in `/etc/ssh/sshd_config` to reduce log spam from automated scanners.
- Disable password authentication; require SSH keys.
- Store authorized keys in `/root/.ssh/authorized_keys`.

### Open Ports on Host
Ensure the following ports are accessible (subject to firewall rules):
- `8006/tcp`: pveproxy (Proxmox Web UI)
- `3128/tcp`: spiceproxy (for VM console access)
- `<CUSTOM_SSH_PORT>/tcp`: SSH

## Automated Port Forwarding Script

To maintain complex iptables DNAT rules (including MTU clamping), use an idempotent shell script that can be rerun safely without creating duplicate rules.

### `portforward.sh` Example

```bash
#!/bin/bash

# Define internal targets
PROXY_VM="<PROXY_VM_IP>"

# ---------- HELPER FUNCTIONS TO PREVENT DUPLICATES ----------
add_nat_rule() {
    iptables -t nat -C "$@" 2>/dev/null || iptables -t nat -A "$@"
}
add_forward_rule() {
    iptables -C "$@" 2>/dev/null || iptables -A "$@"
}
add_input_rule() {
    iptables -C INPUT "$@" 2>/dev/null || iptables -I INPUT "$@"
}
add_mangle_rule() {
    iptables -t mangle -C "$@" 2>/dev/null || iptables -t mangle -A "$@"
}

# ---------- NETWORK FIXES (MTU CLAMPING & ICMP) ----------
add_input_rule -p icmp -j ACCEPT
add_mangle_rule PREROUTING -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --set-mss 1360
add_mangle_rule OUTPUT -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --set-mss 1360
add_mangle_rule FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
add_mangle_rule POSTROUTING -p tcp --tcp-flags SYN,RST SYN -o vmbr0 -j TCPMSS --clamp-mss-to-pmtu

# ---------- WEB TRAFFIC (NGINX ON PROXY VM) ----------
add_nat_rule PREROUTING -i vmbr0 -p tcp --dport 80 -j DNAT --to $PROXY_VM:80
add_forward_rule FORWARD -p tcp -d $PROXY_VM --dport 80 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT

add_nat_rule PREROUTING -i vmbr0 -p tcp --dport 443 -j DNAT --to $PROXY_VM:443
add_forward_rule FORWARD -p tcp -d $PROXY_VM --dport 443 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT

# ---------- DATABASE PORT FORWARDING (HAPROXY ON PROXY VM) ----------
# Example: Exposing Postgres ports
for PORT in 5432 5433 5434; do
    add_nat_rule PREROUTING -i vmbr0 -p tcp --dport $PORT -j DNAT --to $PROXY_VM:$PORT
    add_forward_rule FORWARD -p tcp -d $PROXY_VM --dport $PORT -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT
done
```

### Persistence
After running the script, persist the rules so they survive a reboot:
```bash
bash /root/portforward.sh
netfilter-persistent save
```
The rules are written to `/etc/iptables/rules.v4` and loaded automatically by the `netfilter-persistent` service at startup.
