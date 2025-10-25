# Hypervisor Installation Guide

## Overview

This guide covers remote installation of hypervisors on HP DL380 Gen11 servers located in a colocation datacenter, using iLO 6 virtual media.

**Hypervisor Options:**
- **Proxmox VE 8.x** (recommended - free, open-source)
- **VMware ESXi 8.0** (requires paid licenses for vCenter)

---

## Proxmox VE Installation (Recommended)

### Why Proxmox?

**Advantages:**
- ✅ **Free** - no licensing costs
- ✅ **Full-featured** - clustering, HA, live migration included
- ✅ **Modern web UI** - manage VMs, storage, networking
- ✅ **KVM/QEMU** - excellent Linux VM performance
- ✅ **Built-in clustering** - 3-node cluster support (like vCenter)
- ✅ **Active community** - extensive documentation
- ✅ **No CPU/RAM limits** - unlike ESXi free
- ✅ **API-driven** - Terraform/Ansible support

**Disadvantages:**
- Debian-based (some prefer RHEL ecosystem)
- Less common than VMware in enterprise

### Proxmox Installation Steps

#### 1. Download Proxmox VE ISO

From your workstation:

```bash
# Download latest Proxmox VE ISO (8.2 as of writing)
cd ~/Downloads
wget https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso/proxmox-ve-8-2-iso-installer

# Or download via browser:
# https://www.proxmox.com/en/downloads

# Verify checksum
sha256sum proxmox-ve_8.2-1.iso
# Compare with checksum on website
```

#### 2. Access iLO and Mount ISO

**For k8s-host-01** (repeat for host-02 and host-03):

```bash
# Connect via VPN first
# Then access iLO web interface
https://192.168.10.11
```

**In iLO web interface:**

1. **Login** with admin credentials
2. **Virtual Media** → **CD/DVD-ROM**
3. **Scripted Media URL**: Click "Local ISO"
4. **Browse** and select `proxmox-ve_8.2-1.iso` from your computer
5. **Mount** (wait for upload - may take 5-10 minutes)
6. **Power** → **Momentary Press** (soft power on)
7. **Remote Console** → **HTML5 Console** (or Java if preferred)

#### 3. Boot from Virtual CD

In the remote console:

1. Server boots, press **F11** for **Boot Menu** (if not auto-booting from CD)
2. Select **Virtual Removable Media** or **Virtual CD/DVD**
3. Proxmox installer boots (grub menu appears)

#### 4. Proxmox Installation Wizard

**Installer screens:**

**Screen 1: Welcome**
- Select **Install Proxmox VE (Graphical)**
- Press Enter

**Screen 2: EULA**
- Read and click **I agree**

**Screen 3: Target Harddisk**
- **Target Harddisk**: Select RAID 1 volume (2× 480 GB SSDs)
  - Should show as "HPE Logical Volume" or similar
- **Filesystem**: `ext4` (default, reliable)
  - Alternative: `XFS` (better for large files)
  - Alternative: `ZFS RAID1` (if you want software RAID features, but we're using hardware RAID)
- Click **Options** to verify:
  - `hdsize`: should be ~446 GB (480 GB minus overhead)
- Click **Next**

**Screen 4: Location and Time Zone**
- **Country**: Select your country (affects mirrors, timezone)
- **Time zone**: Select your timezone
- **Keyboard Layout**: Default (us) or select your layout
- Click **Next**

**Screen 5: Administration Password and Email**
- **Password**: Strong password (20+ chars, use password manager)
- **Confirm**: Re-enter password
- **Email**: your-email@example.com (for system alerts)
- Click **Next**

**Screen 6: Network Configuration**

⚠️ **IMPORTANT** - this is for management network (VLAN 20):

```
Management Interface: enp1s0f0  (first 10G NIC)
Hostname (FQDN): k8s-host-01.yourdomain.com
IP Address: 10.10.20.11
Netmask: 255.255.255.0  (or /24)
Gateway: 10.10.20.1  (ask colocation provider)
DNS Server: 1.1.1.1  (or your internal DNS)
```

**Notes:**
- Use **first NIC only** for now (we'll configure bonding later)
- Hostname **must include domain** (e.g., `.local`, `.yourdomain.com`)
- Gateway is usually first IP in subnet (.1)

Click **Next**

**Screen 7: Summary**
- Review all settings
- Click **Install**
- Installation takes 5-10 minutes

**Screen 8: Installation Complete**
- Click **Reboot**
- Remove virtual CD (iLO → Virtual Media → Unmount)

#### 5. Post-Install Access

After reboot, Proxmox displays:

```
Welcome to Proxmox Virtual Environment!

  Web Interface: https://10.10.20.11:8006/

  Login: root
```

**Access web UI** from your workstation (via VPN):

```bash
https://10.10.20.11:8006
```

- **Username**: `root`
- **Password**: (password you set during install)
- **Realm**: `Linux PAM standard authentication`

You'll see a warning about no subscription - this is fine for free use.

#### 6. Post-Installation Configuration

**In Proxmox web UI:**

##### 6.1 Disable Enterprise Repository (Optional for Free Use)

```bash
# SSH to Proxmox host
ssh root@10.10.20.11

# Disable enterprise repo (requires subscription)
echo "#deb https://enterprise.proxmox.com/debian/pve bookworm pve-enterprise" > /etc/apt/sources.list.d/pve-enterprise.list

# Enable no-subscription repo
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list

# Update packages
apt update
apt dist-upgrade -y
```

##### 6.2 Configure Network Bonding (LACP)

**Create bond with both 10G NICs:**

```bash
# Edit network config
nano /etc/network/interfaces
```

**Replace** the auto-generated config with:

```bash
auto lo
iface lo inet loopback

# Physical interfaces (no IP, used in bond)
auto enp1s0f0
iface enp1s0f0 inet manual

auto enp1s0f1
iface enp1s0f1 inet manual

# Bond interface (LACP 802.3ad)
auto bond0
iface bond0 inet manual
    bond-slaves enp1s0f0 enp1s0f1
    bond-mode 802.3ad
    bond-miimon 100
    bond-downdelay 200
    bond-updelay 200
    bond-lacp-rate 1
    bond-xmit-hash-policy layer3+4

# Bridge for VMs (on top of bond)
auto vmbr0
iface vmbr0 inet static
    address 10.10.20.11/24
    gateway 10.10.20.1
    bridge-ports bond0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 20 30 40

# MTU for jumbo frames
    mtu 9000
```

**Notes:**
- `bond0` aggregates both 10G NICs (LACP)
- `vmbr0` is Linux bridge for VMs
- `bridge-vlan-aware yes` allows VLAN tagging on VMs
- `bridge-vids 20 30 40` allows VLANs 20, 30, 40
- Requires LACP configured on datacenter switch (ask provider)

**Apply changes:**

```bash
# Test config syntax
ifup --no-act vmbr0

# Apply (will disconnect briefly)
systemctl restart networking

# Or reboot to be safe
reboot
```

**Verify** after reboot:

```bash
# Check bond status
cat /proc/net/bonding/bond0
# Should show: "Bonding Mode: IEEE 802.3ad Dynamic link aggregation"
# Should show: "MII Status: up" for both slaves

# Check bridge
ip addr show vmbr0
# Should show: 10.10.20.11/24

# Test connectivity
ping 10.10.20.1
ping 1.1.1.1
```

##### 6.3 Configure Storage

**Add NVMe drives as separate datastores:**

```bash
# List available disks
lsblk

# Expected output:
# sda: 446G (RAID 1 boot disk, already used by Proxmox)
# nvme0n1: 1.8T (first NVMe SSD)
# nvme1n1: 1.8T (second NVMe SSD)

# Create LVM volume group on each NVMe
pvcreate /dev/nvme0n1
vgcreate nvme-vg-01 /dev/nvme0n1

pvcreate /dev/nvme1n1
vgcreate nvme-vg-02 /dev/nvme1n1

# Create thin pool (allows overprovisioning, snapshots)
lvcreate -L 1.8T -n data nvme-vg-01
lvcreate -L 1.8T -n data nvme-vg-02
```

**In Proxmox web UI:**

1. **Datacenter** → **Storage** → **Add** → **LVM-Thin**
2. **ID**: `nvme-local-01`
3. **Volume Group**: `nvme-vg-01`
4. **Content**: ☑ Disk image, ☑ Container
5. **Nodes**: Select `k8s-host-01`
6. **Add**

Repeat for `nvme-vg-02`.

**Or via CLI:**

```bash
pvesm add lvmthin nvme-local-01 --vgname nvme-vg-01 --thinpool data --content images,rootdir
pvesm add lvmthin nvme-local-02 --vgname nvme-vg-02 --thinpool data --content images,rootdir
```

##### 6.4 Update Proxmox and Firmware

```bash
# Update Proxmox packages
apt update
apt dist-upgrade -y

# Install HP monitoring tools (optional)
apt install hp-health hponcfg -y

# Check HP firmware via iLO (web UI has firmware update tool)
```

#### 7. Repeat for Other Servers

- Install Proxmox on **k8s-host-02** (192.168.10.12, 10.10.20.12)
- Install Proxmox on **k8s-host-03** (192.168.10.13, 10.10.20.13)
- Use same steps, adjust IPs and hostnames

---

### Proxmox Clustering (Optional but Recommended)

**After all 3 servers installed**, create a cluster for centralized management:

**On k8s-host-01:**

```bash
# Create cluster
pvecm create k8s-cluster

# Check status
pvecm status
# Should show: 1 node
```

**On k8s-host-02:**

```bash
# Join cluster (use host-01's IP)
pvecm add 10.10.20.11

# Enter root password for host-01 when prompted
```

**On k8s-host-03:**

```bash
# Join cluster
pvecm add 10.10.20.11
```

**Verify cluster**:

```bash
# On any node
pvecm status
# Should show: 3 nodes

# In web UI:
# Datacenter → Cluster → Now shows all 3 nodes
# Can manage all servers from one interface!
```

**Benefits of clustering:**
- Manage all 3 servers from one web UI
- Live migration of VMs between hosts
- High availability (VMs auto-restart on failure)
- Shared configuration

---

## VMware ESXi Installation (Alternative)

### Why ESXi?

**Advantages:**
- ✅ Industry standard
- ✅ Mature, widely used
- ✅ Excellent Windows VM support
- ✅ Enterprise support available

**Disadvantages:**
- ❌ **Expensive** - vCenter Standard ~$6,000, Enterprise Plus ~$12,000
- ❌ **Free version limitations**:
  - Max 8 vCPUs per VM (not enough for our 16-core workers!)
  - No vCenter (can't manage all hosts centrally)
  - No API access
  - No vMotion

⚠️ **Recommendation**: Only use ESXi if you have paid licenses (Standard or Enterprise Plus).

### ESXi Installation Steps (Brief)

#### 1. Download ESXi

- Requires VMware account (free to register)
- Download from: https://customerconnect.vmware.com/
- **Version**: ESXi 8.0 U2 (or latest)
- **File**: `VMware-ESXi-8.0U2-<build>-x86_64.iso`

#### 2. Install via iLO Virtual Media

1. Mount ESXi ISO via iLO (same as Proxmox)
2. Boot from virtual CD
3. ESXi installer boots (yellow/black screen)
4. Press **Enter** to start installation
5. Accept EULA (**F11**)
6. Select boot disk (RAID 1 array, ~446 GB)
7. Select keyboard layout
8. Set root password
9. **Confirm install** (F11)
10. Installation takes ~5 minutes
11. Reboot, remove virtual CD

#### 3. Configure Management Network

**After reboot**, ESXi shows:

```
Press F2 to customize system/view logs
```

**Via iLO remote console:**

1. Press **F2**
2. Login with root password
3. **Configure Management Network** → **IPv4 Configuration**
   - [ ] Use DHCP
   - [×] Set static IPv4 address
   - IP: `10.10.20.11`
   - Subnet: `255.255.255.0`
   - Gateway: `10.10.20.1`
4. **DNS Configuration**
   - Primary: `1.1.1.1`
   - Hostname: `k8s-host-01.yourdomain.com`
5. **Esc** to save
6. Restart management network: **Y**

#### 4. Access ESXi Web UI

```bash
https://10.10.20.11
```

- Username: `root`
- Password: (set during install)

#### 5. Configure Networking (vSwitch with LACP)

**In ESXi web UI:**

1. **Networking** → **Virtual switches** → **vSwitch0**
2. **Add uplink** → Select second 10G NIC (`vmnic1`)
3. **Teaming and failover**:
   - **Load balancing**: Route based on IP hash
   - **Active uplinks**: vmnic0, vmnic1
4. **Save**

**Note**: ESXi requires Enterprise Plus license for proper LACP (policy-based teaming). Without it, use active-passive or IP hash.

#### 6. Create Port Groups (VLANs)

1. **Networking** → **Port groups** → **Add port group**
2. **Name**: `VLAN-20-Management`
   - **VLAN ID**: `20`
   - **Virtual switch**: `vSwitch0`
3. Repeat for:
   - `VLAN-30-K8s` (VLAN 30)
   - `VLAN-40-Public` (VLAN 40)

#### 7. Configure Storage (NVMe Datastores)

1. **Storage** → **Datastores** → **New datastore**
2. **Type**: VMFS
3. **Name**: `nvme-datastore-01`
4. **Device**: Select first NVMe drive (`naa.xxx`)
5. **VMFS version**: VMFS 6
6. **Finish**

Repeat for second NVMe drive (`nvme-datastore-02`).

#### 8. Licensing (if using paid ESXi)

1. **Manage** → **Licensing** → **Assign license**
2. Enter license key (Standard or Enterprise Plus)
3. **Check license** (verify features enabled)

---

## Creating VM Templates

After hypervisor installation, create a VM template for Kubernetes nodes.

### Proxmox: Ubuntu 22.04 LTS Template

```bash
# SSH to Proxmox host
ssh root@10.10.20.11

# Download Ubuntu cloud image
cd /var/lib/vz/template/iso
wget https://cloud-images.ubuntu.com/releases/22.04/release/ubuntu-22.04-server-cloudimg-amd64.img

# Create VM (ID 9000 = template convention)
qm create 9000 --name ubuntu-22.04-template --memory 4096 --cores 2 --net0 virtio,bridge=vmbr0,tag=30

# Import cloud image as disk
qm importdisk 9000 ubuntu-22.04-server-cloudimg-amd64.img nvme-local-01
qm set 9000 --scsihw virtio-scsi-pci --scsi0 nvme-local-01:vm-9000-disk-0

# Add cloud-init drive
qm set 9000 --ide2 nvme-local-01:cloudinit

# Set boot disk
qm set 9000 --boot c --bootdisk scsi0

# Set serial console
qm set 9000 --serial0 socket --vga serial0

# Enable agent
qm set 9000 --agent enabled=1

# Convert to template
qm template 9000
```

**To create VM from template:**

```bash
# Clone template (full clone)
qm clone 9000 101 --name k8s-master-01 --full

# Resize disk (template is small, expand to 100 GB)
qm resize 101 scsi0 +90G

# Set CPU and RAM
qm set 101 --cores 8 --memory 16384

# Set static IP via cloud-init
qm set 101 --ipconfig0 ip=10.10.30.10/24,gw=10.10.30.1

# Start VM
qm start 101
```

### ESXi: Ubuntu 22.04 LTS Template

1. **Download Ubuntu Server ISO**: https://ubuntu.com/download/server
2. **Upload to ESXi**: Datastore browser → Upload
3. **Create VM**:
   - Name: `ubuntu-22.04-template`
   - Guest OS: Ubuntu Linux (64-bit)
   - CPU: 2 cores
   - RAM: 4 GB
   - Disk: 100 GB (thin provisioned)
   - Network: VLAN-30-K8s
4. **Install Ubuntu** (mount ISO, follow wizard)
5. **Power off**, right-click → **Convert to Template**

**To deploy from template:**
1. Right-click template → **New VM from This Template**
2. Customize CPU, RAM, disk, IP
3. Power on

---

## Next Steps

After hypervisor and templates ready:

1. **Create 8 VMs** from template (1 master + 7 workers)
2. **Configure networking** (VLANs, static IPs)
3. **Install Kubernetes** (see 06-kubernetes-installation.md)
4. **Deploy CNPG** for PostgreSQL (see 07-postgresql-cnpg-setup.md)

See next document for Kubernetes installation.
