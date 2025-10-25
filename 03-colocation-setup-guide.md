# Colocation Datacenter Setup Guide

## Overview
This guide covers the setup of 3× HP DL380 Gen11 servers in a colocation datacenter with provider-managed network infrastructure.

## Pre-Deployment Planning

### Questions to Ask Your Colocation Provider

#### Rack & Power
1. **What rack space is included?**
   - You need: 6U minimum (3× 2U servers)
   - Recommended: 1/2 rack (21U) or 1/4 rack (10U) for future growth

2. **What power is included?**
   - You need: ~2 kW sustained (3 servers × ~500W each + overhead)
   - Ask for: 2× 15A 120V circuits (A+B feeds) OR 1× 16A 230V circuit
   - Confirm: Power is metered or flat-rate?

3. **Are PDUs provided?**
   - Ideal: Dual PDUs (A+B power) for redundancy
   - Each server has 2 PSUs - connect to different PDUs

4. **What's the price per amp or per kWh?**
   - Typical: $20-50/amp/month or $0.10-0.15/kWh

#### Network
5. **How many network ports are included?**
   - You need: 6× 10 Gbps ports (2 per server) + 3× 1 Gbps (for iLO)
   - Ask if cross-connects are free or charged per port

6. **What network topology do you provide?**
   - Top-of-rack (ToR) switch with access to their backbone?
   - Can you create VLANs?
   - What's the uplink speed (1G, 10G, 100G)?

7. **IP addressing - public or private?**
   - Option A: They assign you public IPs (ask for /28 or /29 block)
   - Option B: Private IPs with NAT (you'll need VPN)
   - Do they charge per IP?

8. **Do you provide VPN access to management VLAN?**
   - Critical for accessing iLO remotely
   - Or do you allow direct public IP on iLO? (less secure)

#### Access & Support
9. **What remote hands services are available?**
   - Cost: ~$100-200/hour (for cable swaps, reboots)
   - Response time: 15 minutes, 1 hour, 4 hours?

10. **Can I access the datacenter for physical work?**
    - Badge access 24/7 or business hours only?
    - ID requirements?
    - Need to schedule visits?

11. **How do I ship equipment to the facility?**
    - Shipping address
    - Receiving hours
    - Who signs for deliveries?

12. **What monitoring and alerts do you provide?**
    - Power alerts
    - Temperature monitoring
    - Network port status

---

## Network Design (Using Colo-Provided Switches)

### VLAN Strategy

Since the colocation provides switch ports, work with them to configure these VLANs:

#### VLAN 10: Management (iLO/BMC)
- **Purpose**: Remote server management (iLO 6 interfaces)
- **Subnet**: 192.168.10.0/24 (or provider-assigned)
- **IPs**:
  - 192.168.10.11 - k8s-host-01 iLO
  - 192.168.10.12 - k8s-host-02 iLO
  - 192.168.10.13 - k8s-host-03 iLO
- **Access**: VPN only (do NOT expose to internet)
- **Ports**: 3× 1 Gbps connections

#### VLAN 20: Hypervisor Management
- **Purpose**: Proxmox/ESXi web UI and SSH access
- **Subnet**: 10.10.20.0/24
- **IPs**:
  - 10.10.20.11 - k8s-host-01 management
  - 10.10.20.12 - k8s-host-02 management
  - 10.10.20.13 - k8s-host-03 management
- **Access**: VPN or bastion host
- **VLAN tagging**: Tagged on server 10G NICs

#### VLAN 30: Kubernetes Cluster Network
- **Purpose**: Kubernetes node-to-node communication
- **Subnet**: 10.10.30.0/24
- **IPs**:
  - 10.10.30.10 - k8s-master-01
  - 10.10.30.21-24 - k8s-app-worker-01 to 04
  - 10.10.30.31-33 - k8s-pg-worker-01 to 03
- **MTU**: 9000 (jumbo frames for better performance)
- **VLAN tagging**: Tagged on server 10G NICs

#### VLAN 40: Application/Public Access (Optional)
- **Purpose**: Public-facing load balancer / ingress
- **Subnet**: Provider public IP block or NAT'd private
- **Access**: Internet-facing
- **VLAN tagging**: Tagged on server 10G NICs

### Network Port Assignment

**Per Server** (HP DL380 Gen11):

| Port | Interface | VLAN | Speed | Purpose |
|------|-----------|------|-------|---------|
| iLO | Dedicated 1G | VLAN 10 | 1 Gbps | Remote management |
| NIC1 | Embedded SFP28 #1 | VLAN 20, 30, 40 (tagged) | 10 Gbps | Primary data path |
| NIC2 | Embedded SFP28 #2 | VLAN 20, 30, 40 (tagged) | 10 Gbps | Secondary data path (bond) |

**Bond/Team Configuration** (in hypervisor):
- NIC1 + NIC2 bonded (LACP 802.3ad or active-backup)
- Provides 10 Gbps with failover
- All VM traffic goes through bond

### IP Allocation Table

| Hostname | iLO (VLAN 10) | Hypervisor (VLAN 20) | Kubernetes (VLAN 30) | Notes |
|----------|---------------|----------------------|----------------------|-------|
| k8s-host-01 | 192.168.10.11 | 10.10.20.11 | N/A | Physical host |
| k8s-host-02 | 192.168.10.12 | 10.10.20.12 | N/A | Physical host |
| k8s-host-03 | 192.168.10.13 | 10.10.20.13 | N/A | Physical host |
| k8s-master-01 | N/A | N/A | 10.10.30.10 | VM on host-01 |
| k8s-app-worker-01 | N/A | N/A | 10.10.30.21 | VM on host-01 |
| k8s-app-worker-02 | N/A | N/A | 10.10.30.22 | VM on host-01 |
| k8s-app-worker-03 | N/A | N/A | 10.10.30.23 | VM on host-02 |
| k8s-app-worker-04 | N/A | N/A | 10.10.30.24 | VM on host-03 |
| k8s-pg-worker-01 | N/A | N/A | 10.10.30.31 | VM on host-02 |
| k8s-pg-worker-02 | N/A | N/A | 10.10.30.32 | VM on host-02 |
| k8s-pg-worker-03 | N/A | N/A | 10.10.30.33 | VM on host-03 |

### DNS Configuration

**Option A: Internal DNS** (recommended)
- Run CoreDNS or dnsmasq on a VM or container
- Forward external queries to 1.1.1.1 or 8.8.8.8
- Manage /etc/hosts or DNS zone files

**Option B: External DNS**
- Use Cloudflare, Route53, or other managed DNS
- Only for public-facing services
- Internal services use /etc/hosts

---

## Physical Installation Steps

### 1. Pre-Shipment Preparation (at your location)

If you receive servers before shipping to colo:

1. **Unbox and inspect all 3 servers**
   - Check for shipping damage
   - Verify all components present

2. **Power on each server**
   - Connect to temporary monitor/keyboard
   - Or use iLO via DHCP (find IP from router)

3. **Configure iLO 6**
```bash
# Access iLO web interface (default: https://192.168.1.x)
# Login: Administrator / password on pull-out tag

# Settings to configure:
- Set static IP (or reserve DHCP)
- Change admin password (strong password!)
- Enable SSH (if needed)
- Enable Serial-Over-LAN (SOL)
- Update iLO firmware (from support.hpe.com)
- Document MAC address
```

4. **Update firmware**
   - Download HP Service Pack for ProLiant (SPP)
   - Create bootable USB or mount via iLO virtual media
   - Run Smart Update Manager (SUM)
   - Update: BIOS, iLO, NIC firmware, RAID controller

5. **Configure BIOS settings** (via iLO remote console):
```
Power Management:
  - HP Power Profile: Maximum Performance
  - Minimum Processor Idle Power Core C-State: No C-states
  - Energy/Performance Bias: Maximum Performance

Virtualization:
  - Intel VT: Enabled
  - Intel VT-d: Enabled
  - SR-IOV: Enabled

Boot Options:
  - Boot Mode: UEFI
  - Secure Boot: Disabled (for Linux hypervisors)
  - Boot Order: Hard Drive first
```

6. **Create RAID array for boot disks**
   - Access Smart Storage Administrator (Ctrl+R during boot)
   - Create RAID 1 with 2× 480 GB SATA SSDs
   - Label: "Boot Volume"
   - Initialize volume

7. **Test remote installation**
   - Mount Ubuntu or Rocky Linux ISO via iLO virtual media
   - Boot from virtual CD/DVD
   - Verify installation proceeds
   - Don't complete installation yet - just test

8. **Document everything**
   - iLO IP, MAC, password
   - Server MAC addresses (for NIC1, NIC2)
   - Serial numbers
   - Firmware versions
   - Take photos of BIOS settings

### 2. Coordinate with Colocation Provider

**2 weeks before shipment:**
- [ ] Confirm rack space is ready (1/4 rack or 1/2 rack)
- [ ] Confirm power circuits installed (2× 15A or 1× 16A)
- [ ] Request VLAN configuration (VLANs 10, 20, 30, 40)
- [ ] Request IP addresses (management and cluster subnets)
- [ ] Set up VPN access to management VLAN
- [ ] Provide MAC addresses for switch port assignment

**1 week before:**
- [ ] Schedule delivery date/time
- [ ] Provide shipping tracking number
- [ ] Send racking instructions (if using remote hands)
- [ ] Send cable layout diagram

**Racking instructions to send**:
```
Equipment: 3× HP DL380 Gen11 servers (2U each)

Rack layout (top to bottom):
  U19-20: k8s-host-01 (labeled)
  U17-18: k8s-host-02 (labeled)
  U15-16: k8s-host-03 (labeled)
  U14: Cable management (1U blank panel)

Cabling (per server):
  - 2× power cables to separate PDUs (A+B feeds)
  - 1× Cat6 from iLO port to management switch (VLAN 10)
  - 2× SFP+ cables from server NICs to data switch (VLANs 20/30/40)

Power on sequence:
  1. Connect all cables
  2. Power on k8s-host-03 (bottom)
  3. Wait 5 minutes
  4. Power on k8s-host-02 (middle)
  5. Wait 5 minutes
  6. Power on k8s-host-01 (top)

Notify when complete: your-email@example.com
```

### 3. Post-Installation Remote Setup

Once servers are racked and powered:

1. **Verify iLO access**
```bash
# From your workstation (via VPN)
ping 192.168.10.11
ping 192.168.10.12
ping 192.168.10.13

# Access web interfaces
https://192.168.10.11  # k8s-host-01
https://192.168.10.12  # k8s-host-02
https://192.168.10.13  # k8s-host-03
```

2. **Check hardware health**
   - iLO → System Information → Health Status
   - Verify: All fans, PSUs, temps OK
   - Check: All drives detected

3. **Verify network connectivity**
   - iLO → Network → Ping Test
   - Ping gateway, DNS servers

4. **Check NIC link status**
   - iLO → System Information → Network Adapters
   - Verify: Both 10G NICs show "Link Up"
   - Note: Speed (10 Gbps), VLAN tags

---

## Hypervisor Installation (Remote via iLO)

You'll install the hypervisor remotely using iLO virtual media. See next document (04-hypervisor-installation.md) for details.

**High-level steps:**
1. Download Proxmox VE ISO (or ESXi ISO)
2. Mount ISO via iLO virtual media
3. Boot server from virtual CD
4. Follow installation wizard via iLO remote console
5. Configure network (VLANs, bonds)
6. Repeat for all 3 servers

---

## Colocation Best Practices

### Security

1. **iLO Security**
   - Use strong passwords (20+ chars, password manager)
   - Enable two-factor authentication (iLO Advanced)
   - Disable unused services (IPMI v1, SNMP v1/v2c)
   - Only allow access via VPN

2. **Network Security**
   - Never expose iLO to public internet
   - Use VPN for all management access
   - Implement firewall rules at hypervisor level
   - Enable fail2ban on hypervisor and VMs

3. **Physical Security**
   - Use locking rack (if colo allows)
   - Set BIOS password (prevents unauthorized changes)
   - Enable Secure Boot (if using signed OS)

### Monitoring

1. **Hardware Monitoring**
   - Use HP's Agentless Management Service (AMS)
   - Or install HP MCP (Management Component Pack) in hypervisor
   - Send SNMP traps to monitoring system
   - Set up email alerts from iLO

2. **Power Monitoring**
   - Track power usage via iLO → Power Meter
   - Optimize workload if exceeding power budget

3. **Temperature Monitoring**
   - iLO reports inlet/exhaust temps
   - Alert if > 30°C (colo AC may be failing)

### Backups

1. **Configuration Backups**
   - Export iLO config regularly (iLO → Maintenance → Backup)
   - Export hypervisor config (Proxmox backup feature)
   - Store off-site (Git repo, S3, etc.)

2. **VM Backups**
   - Use Proxmox Backup Server or Velero (Kubernetes)
   - Store backups off-site (S3, Backblaze B2)
   - Test restores monthly

### Disaster Recovery

**If server fails:**
1. Check iLO logs (iLO → Logs → Integrated Management Log)
2. Contact colo for remote hands (check/reseat RAM, drives)
3. If unrecoverable, ship replacement server
4. Restore VMs from backup
5. Kubernetes should self-heal (pods reschedule to other nodes)

**If network fails:**
1. Check iLO (if iLO still accessible)
2. Contact colo NOC (Network Operations Center)
3. Verify with remote hands (check cables, switch ports)

**If power fails:**
1. Colo should have N+1 redundant power (rare failure)
2. Servers will auto-power-on when power restored (set in BIOS)
3. Monitor Kubernetes pods for recovery

---

## Ongoing Maintenance

### Monthly
- [ ] Check iLO hardware health logs
- [ ] Review power consumption trends
- [ ] Check firmware update availability
- [ ] Test iLO remote console access

### Quarterly
- [ ] Update hypervisor (Proxmox/ESXi patches)
- [ ] Update Kubernetes (minor versions)
- [ ] Test disaster recovery plan (restore a VM from backup)

### Annually
- [ ] Update server firmware (BIOS, iLO, NIC, RAID)
- [ ] Review colocation contract (price increases?)
- [ ] Review resource utilization (need more servers?)

---

## Cost Breakdown (Typical Colocation)

### One-Time Costs
| Item | Cost |
|------|------|
| Setup fee (rack, IP assignment) | $200-500 |
| Shipping to datacenter (3 servers) | $200-400 |
| Remote hands for racking (2 hours) | $200-400 |
| **Total one-time** | **~$600-1,300** |

### Monthly Recurring Costs
| Item | Monthly Cost |
|------|--------------|
| 1/2 rack space (21U) | $250-400 |
| Power (2 kW @ $50/amp or metered) | $150-300 |
| Bandwidth (10 Gbps port, 20 TB transfer) | $100-200 |
| IP addresses (8 IPs) | $20-50 |
| **Total monthly** | **~$520-950** |

**Annual cost**: $6,240 - $11,400/year for colocation

---

## Troubleshooting Common Issues

### Issue: Cannot access iLO after racking

**Possible causes:**
- Cable not connected to correct VLAN/port
- iLO configured with wrong IP
- VPN not routing to management VLAN

**Fix:**
- Ask colo to check cable (remote hands)
- Boot server with monitor, press F8 for iLO config, reconfigure
- Verify VPN route: `ip route get 192.168.10.11`

### Issue: 10G NICs not showing link

**Possible causes:**
- SFP+ cable not seated properly
- Wrong transceiver type (SR vs LR)
- Switch port not configured

**Fix:**
- Ask colo to reseat cables
- Verify switch port shows "up" (ask NOC)
- Check NIC firmware version (update if old)

### Issue: High server temperatures

**Possible causes:**
- Datacenter AC failure
- Blocked airflow (front/rear obstructions)
- Failed fan

**Fix:**
- Check iLO temps (iLO → System Information → Temperatures)
- Contact colo if datacenter ambient > 25°C
- Check fan status in iLO (should auto-adjust speed)

---

## Next Steps

After colocation setup complete:
1. Install hypervisor (see 04-hypervisor-installation.md)
2. Configure networking and storage (see 05-network-storage-setup.md)
3. Deploy Kubernetes (see 06-kubernetes-installation.md)
4. Set up PostgreSQL with CNPG (see 07-postgresql-cnpg-setup.md)
