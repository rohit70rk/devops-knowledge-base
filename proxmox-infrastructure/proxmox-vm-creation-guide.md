# 📘 Proxmox VM Creation

## 🎯 Objective
Create a new VM in Proxmox that:
- Matches the architecture & configuration of a reference VM (e.g., VM 100).
- Allows custom resources (CPU/RAM/Disk).
- Does not rely on cloning or templates.

---

## 🧱 PHASE 1: Create Base VM (GUI)

### Step 1: Create New VM
1. Go to: **Datacenter → Node → Create VM**
2. Fill:
   - **Node:** `example-node`
   - **VM ID:** (new ID, e.g., `102`)
   - **Name:** (e.g., `PROXY-VM`)
- Click **Next**

### Step 2: OS Selection
- **Select:**
  - Use CD/DVD disc ISO Image → Ubuntu 24.04 (or your OS) from your local storage.
- **Type:**
  - Linux
  - Version: 6.x - 2.6 Kernel
- Click **Next**

### Step 3: System Configuration
Set exactly like reference VM:
- **Machine:** q35 ✅
- **BIOS:** OVMF (UEFI) ✅
- **Add EFI Disk:** Checked ✅
- **EFI Storage:** `local-lvm` ✅
- **Pre-Enrolled keys:** Checked ✅
- **Qemu Agent:** Checked ✅
- **SCSI Controller:** VirtIO SCSI single ✅
- Click **Next**

### Step 4: Disk Configuration
- **Bus/Device:** SCSI/0 ✅
- **Storage:** `local-lvm` ✅
- **Disk Size:** (as required, e.g., `50GB`) ✅
- **Enable Discard (TRIM):** Checked ✅
- **Enable IO Thread:** Checked ✅
- Click **Next**

### Step 5: CPU Configuration
- **Type:** host ✅
- **Sockets:** `1`
- **Cores:** (as required, e.g., `2`)
- Click **Next**

### Step 6: Memory
- Set RAM (e.g., `4096 MB`)
👉 *Leave ballooning enabled (default)*
- Click **Next**

### Step 7: Network
- **Bridge:** `vmbr1` ✅
- **Model:** VirtIO (paravirtualized) ✅
- **Firewall:** Enabled ✅
- Leave MAC address blank.
- Click **Next → Confirm → Finish**

### Step 8: Pre-Boot Options
Before starting, go to the VM's **Options** tab:
- **Start at boot:** Double-click, check the box, and click **OK**.
- **Boot Order:** `boot: order=scsi0;ide2;net0`
- Ensure **qemu-guest-agent** is Enabled.

---

## 💻 PHASE 2: Install OS

### Step 1: Start VM → Open Console
### Step 2: Basic Setup
- **Language:** Select English (or preferred).
- **Keyboard:** Select English (US) (or preferred).

### Step 3: Choose the type of installation
- **Install Type:** Choose Ubuntu Server (Do not choose minimized unless specifically needed).

### Step 4: Network Configuration (Static IP)
By default, it will fail to auto-configure because your private `vmbr1` network does not have DHCP.
- Arrow up to the network interface (e.g., `enp6s18`) and hit **Enter**.
- Select **Edit IPv4**.
- Change Method from Automatic (DHCP) to **Manual**.
- Fill in the routing details:
  - **Subnet:** `192.168.1.0/24` (Your private network block)
  - **Address:** `192.168.1.x` (Assign a unique IP for this specific VM)
  - **Gateway:** `192.168.1.1` (Your Proxmox host)
  - **Name servers:** `8.8.8.8, 1.1.1.1`
  - **Search Domain:** Do Nothing
- **Create Bond:** Do Nothing
- Select **Save**, then hit **Done**.

### Step 5: Proxy Configuration
- **Proxy Address:** Leave completely blank and hit **Done**.

### Step 6: Mirror Configuration
- **Archive Mirror:** Leave as the default and hit **Done**.

### Step 7: Storage Configuration
1. Ubuntu defaults to only using 50% of the disk for the root partition. You must manually expand it.
   - Check `[X]` **Use an entire disk.**
   - Check `[X]` **Set up this disk as an LVM group.**
   - Uncheck `[ ]` **Encrypt the LVM group with LUKS** (unless required).
   - Hit **Done**.
2. On the File System Summary screen:
   - Arrow down to `ubuntu-lv` under the "USED DEVICES" section and hit **Enter**.
   - Select **Edit**.
   - Change the Size field to the maximum available number shown on the right (e.g., change `23.000G` to `46.000G`).
   - Select **Save**.

Verify "Free Space" at the top now shows less than 1GB. Hit **Done** and confirm the destructive changes.

### Step 8: Profile Setup
- **Your name:** Admin user (e.g., `proxy-node`)
- **Your servers name:** Internal hostname (e.g., `proxy-node`)
- **Pick a username:** Login name (e.g., `proxy-node`)
- **Password:** Set a strong password.
- Hit **Done**.

### Step 9: Ubuntu Pro Configuration
- **Ubuntu Pro:** Select Skip for now and hit **Continue**.

### Step 10: SSH Configuration
- **SSH Setup:**
  - Highlight **Install OpenSSH server**.
  - Press Spacebar to check the box `[X]`.
  - Leave "Allow password authentication" checked.
- Hit **Done**.

### Step 11: Featured Server Snaps
- Leave all boxes unchecked `[ ]` for a clean, minimal install.
- Arrow down and hit **Done**.
- Wait for the installation to finish (it will download security updates).

### Step 12: Reboot Trigger
- When the installation finishes, select **Reboot Now**.
- The console will pause and say: "Please remove the installation medium, then press ENTER".
- **Do not press Enter yet! Move to Phase 3.**

---

## ⚙️ PHASE 3: Post-Creation and OS Install Fixes

### Step 1: Fix Boot Order
1. Go to: **VM → Options → Boot Order → Edit**
2. Set:
   - ✅ `scsi0` (first)
   - ✅ `net0` (second)
   - ❌ Remove `ide2`
- Click **OK**

### Step 2: Remove ISO
1. Go to: **Hardware** tab
2. Select: **CD/DVD Drive (ide2)**
3. Click: ❌ **Remove**

### Step 3: Reboot Now
- Go back to the VM Console.
- Press **ENTER**. The VM will reboot into the newly installed OS.

---

## 🛡️ PHASE 4: Day-1 OS Configuration

Log into your new VM via the Proxmox Console or SSH, and run the following baseline commands:

### 1. Update Packages
Update the package lists and upgrade existing packages:
```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Install the Proxmox Guest Agent
Install, start, and verify the QEMU guest agent, which communicates with the Proxmox host:
```bash
sudo apt install qemu-guest-agent -y
systemctl start qemu-guest-agent.service
sudo systemctl status qemu-guest-agent
```

### 3. Enable Basic Security (UFW Firewall)
Enable UFW and immediately allow SSH access to prevent lockouts:
```bash
sudo ufw allow OpenSSH
sudo ufw enable
```
*(Press 'y' when warned about disrupting SSH connections)*

### 4. Final Reboot
Reboot the VM to apply any kernel updates and ensure services start correctly:
```bash
sudo reboot
```
