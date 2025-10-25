# HP ProLiant DL380 Gen11 Server Specifications

## Architecture Summary
- **3× HP DL380 Gen11 servers** in colocation datacenter
- **8 Kubernetes VMs**: 1 master + 4 app workers + 3 PostgreSQL workers
- **Form factor**: 2U rack servers (6U total rack space)
- **Remote management**: iLO 6 (included)

## Server Configuration Details

### Server 1: "k8s-host-01" (Master + 2 App Workers)

**Chassis**: HP DL380 Gen11 (2U)

**Processors**:
- **Option A**: 2× Intel Xeon Gold 6430 (32 cores, 2.1 GHz, 270W TDP)
  - Total: 64 cores / 128 threads
  - Good balance of price/performance

- **Option B**: 2× Intel Xeon Silver 4410Y (12 cores, 2.0 GHz, 155W TDP)
  - Total: 24 cores / 48 threads
  - Budget option, sufficient for this server's load

**Memory**:
- **Configuration**: 12× 16GB DDR5-4800 ECC RDIMM
- **Total**: 192 GB
- **Slots used**: 12 of 32 available
- **Upgrade path**: Can add up to 512 GB later

**Storage**:
- **Boot/Hypervisor**:
  - 2× 480 GB SATA SSD (HP part P43903-B21) in RAID 1
  - Connected to embedded Smart Array controller

- **VM Datastore**:
  - 2× 1.92 TB NVMe SSD (HP part P50859-B21)
  - Direct attach, no RAID
  - Total: ~3.8 TB for VMs

**Network**:
- **Embedded**: HPE Ethernet 10/25Gb 2-port 640FLR-SFP28 Adapter
  - 2× 10/25 Gbps SFP28 ports (use as 10G)
  - Use both ports (bond/team for HA)

- **Management**: iLO 6 dedicated 1G port

**Power**:
- 2× 800W Platinum redundant PSUs (included)

**Workload**:
- k8s-master-01 (8 vCPU, 16 GB)
- k8s-app-worker-01 (16 vCPU, 64 GB)
- k8s-app-worker-02 (16 vCPU, 64 GB)
- **Total allocation**: 40 vCPU, 144 GB

---

### Server 2: "k8s-host-02" (2 App + 2 PostgreSQL Workers)

**Chassis**: HP DL380 Gen11 (2U)

**Processors**:
- **Recommended**: 2× Intel Xeon Gold 6438Y+ (32 cores, 2.0 GHz, 205W TDP)
  - Total: 64 cores / 128 threads
  - Higher core count for heavy workload

- **Alternative**: 2× Intel Xeon Gold 6430 (32 cores, 2.1 GHz, 270W TDP)
  - Same as Server 1

**Memory**:
- **Configuration**: 16× 16GB DDR5-4800 ECC RDIMM
- **Total**: 256 GB
- **Slots used**: 16 of 32 available
- **Layout**: Populate all channels evenly for max bandwidth

**Storage**:
- **Boot/Hypervisor**:
  - 2× 480 GB SATA SSD in RAID 1

- **VM Datastore** (Heavy I/O for PostgreSQL):
  - 4× 1.92 TB NVMe SSD (HP part P50859-B21)
  - Or 4× 3.84 TB NVMe if budget allows
  - No RAID, direct VM access
  - Total: ~7.6 TB for VMs

**Network**:
- Same as Server 1: Embedded 10/25Gb 2-port SFP28

**Power**:
- 2× 800W Platinum redundant PSUs

**Workload**:
- k8s-app-worker-03 (16 vCPU, 64 GB)
- k8s-app-worker-04 (16 vCPU, 64 GB)
- k8s-pg-worker-01 (16 vCPU, 64 GB)
- k8s-pg-worker-02 (16 vCPU, 64 GB)
- **Total allocation**: 64 vCPU, 256 GB

---

### Server 3: "k8s-host-03" (1 App + 1 PostgreSQL Worker)

**Chassis**: HP DL380 Gen11 (2U)

**Processors**:
- **Option A**: 2× Intel Xeon Silver 4410Y (12 cores, 2.0 GHz, 155W TDP)
  - Total: 24 cores / 48 threads
  - Budget option

- **Option B**: 2× Intel Xeon Gold 5420+ (28 cores, 2.0 GHz, 225W TDP)
  - Total: 56 cores / 112 threads
  - More headroom

**Memory**:
- **Configuration**: 12× 16GB DDR5-4800 ECC RDIMM
- **Total**: 192 GB

**Storage**:
- **Boot/Hypervisor**:
  - 2× 480 GB SATA SSD in RAID 1

- **VM Datastore**:
  - 2× 1.92 TB NVMe SSD
  - Total: ~3.8 TB for VMs

**Network**:
- Same as Server 1: Embedded 10/25Gb 2-port SFP28

**Power**:
- 2× 800W Platinum redundant PSUs

**Workload**:
- k8s-app-worker-05 (16 vCPU, 64 GB)
- k8s-pg-worker-03 (16 vCPU, 64 GB)
- **Total allocation**: 32 vCPU, 128 GB

---

## Resource Allocation Summary

| Server | Physical CPU | Physical RAM | NVMe Storage | VMs | vCPU Used | RAM Used |
|--------|--------------|--------------|--------------|-----|-----------|----------|
| Host-01 | 64 cores (or 48) | 192 GB | 2× 1.92 TB | 3 | 40 | 144 GB |
| Host-02 | 64 cores | 256 GB | 4× 1.92 TB | 4 | 64 | 256 GB |
| Host-03 | 48 cores (or 56) | 192 GB | 2× 1.92 TB | 2 | 32 | 128 GB |
| **Total** | **176 cores** | **640 GB** | **16 TB** | **9** | **136** | **528 GB** |

**Overcommit ratios**:
- **CPU**: ~1.3:1 (safe for production workloads)
- **RAM**: 1.2:1 (minimal overcommit)

---

## HP Part Numbers for Ordering

### Common Components (All Servers)

**Chassis**:
- P52994-B21: HP DL380 Gen11 2U chassis

**RAID Controller**:
- P55249-B21: HPE Smart Array E208i-p SR Gen11 (embedded, supports RAID 0/1/5)
  - Sufficient for boot disk RAID 1
  - Or use embedded S100i in AHCI mode

**Boot SSDs** (per server):
- P43903-B21: HP 480GB SATA 6G SSD (2× per server)
  - Or P47142-B21: HP 960GB SATA if more space needed

**NVMe SSDs**:
- P50859-B21: HP 1.92TB NVMe Gen4 SSD (mainstream performance)
- P47926-B21: HP 3.84TB NVMe Gen4 SSD (if more capacity needed)
- P50867-B21: HP 1.92TB NVMe Gen4 RI (Read Intensive, for read-heavy workloads)

**Power Supplies**:
- Included: 2× 800W Flex Slot Platinum PSUs (standard)
- Upgrade: P56431-B21 (1600W if needed)

**iLO License** (optional but recommended):
- BD505A: iLO Advanced license (1-server)
  - Features: remote console, virtual media, video record
  - Consider "iLO Advanced Premium Security Edition" for TPM/encryption

**SFP+ Transceivers** (if needed):
- 455883-B21: HP 10GbE SFP+ SR transceiver (for fiber)
- J9150D: HP X242 10G SFP+ SFP+ 3m DAC cable (for direct attach to switch)

---

## CPU Selection Guidance

### Budget Build (~$25,000 total for 3 servers)
- **Server 1**: 2× Xeon Silver 4410Y (24C total)
- **Server 2**: 2× Xeon Gold 6430 (64C total)
- **Server 3**: 2× Xeon Silver 4410Y (24C total)

### Balanced Build (~$35,000 total)
- **Server 1**: 2× Xeon Gold 6430 (64C total)
- **Server 2**: 2× Xeon Gold 6438Y+ (64C total)
- **Server 3**: 2× Xeon Gold 5420+ (56C total)

### Performance Build (~$50,000 total)
- **All servers**: 2× Xeon Gold 6438Y+ or 6448Y (64C each)
- Maximize cores for future growth

**Recommendation**: Start with balanced build. DL380 Gen11 allows CPU upgrades later if needed.

---

## HP iLO 6 Features (Included for Free)

**Standard iLO 6** (no license required):
- Web-based remote management console
- Remote power control (power on/off/reset)
- Health monitoring (fans, temps, PSUs)
- BIOS/firmware management
- Basic remote console (text mode)
- Email alerts for hardware failures

**iLO Advanced** (requires license, ~$250/server):
- Full graphical remote console (KVM over IP)
- Virtual media (mount ISOs remotely for OS install)
- Video recording/playback
- Directory integration (LDAP/AD)

**iLO Advanced Premium Security Edition** (~$500/server):
- Server Configuration Lock
- Workload Performance Advisor
- FIPS/Common Criteria security mode

**For colocation**: iLO Advanced is HIGHLY recommended
- Allows remote OS installation without datacenter hands
- Mount ISO over network, install hypervisor remotely
- No need to ship USB drives or use crash cart

---

## DL380 Gen11 Advantages

1. **Latest Intel Xeon Scalable 4th Gen** (Sapphire Rapids)
   - DDR5 memory support (faster, more bandwidth)
   - PCIe Gen5 support (future NVMe/GPU expansion)
   - Better performance per watt

2. **Flexible Storage**
   - Up to 12× 2.5" SFF drive bays (front)
   - 3× NVMe PCIe slots (rear)
   - Mix SATA/SAS/NVMe as needed

3. **Excellent Serviceability**
   - Tool-free drive bays
   - Redundant hot-swap fans
   - Hot-plug PSUs
   - Easy RAM access (no CPU removal needed)

4. **Proven Platform**
   - 10+ generations of DL380 reliability
   - Wide support in virtualization (VMware, Proxmox, KVM)

5. **Energy Efficient**
   - 80 PLUS Platinum PSUs (>94% efficiency)
   - Lower power bills in colocation

---

## Colocation Datacenter Requirements

### Rack Space
- **6U** (3 servers × 2U each)
- Request: **1/4 rack (10U)** or **1/2 rack (21U)** to allow for growth
  - Extra space for future expansion
  - Cable management

### Power
- **Per server**: ~300-500W under load (depending on CPUs)
- **Total**: ~1.5 kW sustained, 2 kW peak
- Request: **2× 15A 120V circuits** or **1× 16A 230V circuit**
- Ensure **redundant PDUs** (A+B power feeds)

### Network Connectivity
- **6× 10 Gbps ports** (2 per server for NIC bonding/teaming)
- **3× 1 Gbps ports** (for iLO management, separate VLAN)
- Ask datacenter for:
  - Native VLAN for Kubernetes nodes
  - Management VLAN for iLO (isolated, VPN access)

### IP Addressing
- **Option A**: Provider gives you IP block (e.g., /29 = 8 IPs)
  - Assign public IPs to load balancer/ingress only
  - Use private IPs (10.0.0.0/8) for internal nodes

- **Option B**: Use datacenter private network
  - All servers on private IPs
  - NAT/firewall at datacenter edge

**Recommendation**: Use private IPs for all Kubernetes nodes, public IPs only for ingress/load balancer.

### Remote Access
- **iLO Access**:
  - Option A: Datacenter provides VPN to management VLAN
  - Option B: Configure iLO with public IP (use strong passwords + 2FA!)

- **Kubernetes Access**:
  - VPN to cluster network (recommended)
  - Or bastion host with firewall rules

### SLA Requirements
- **Uptime SLA**: Look for 99.9% or higher
- **Power**: N+1 redundancy (colo standard)
- **Cooling**: N+1 redundancy
- **Network**: Multiple upstream providers
- **Remote hands**: Available for reboots, cable swaps (~$100-200/hour)

---

## Pre-Order Checklist

### Information to Gather
- [ ] Choose CPU models (Silver vs Gold vs Platinum)
- [ ] Confirm memory configuration (192GB vs 256GB per server)
- [ ] Decide on iLO Advanced licenses (highly recommended)
- [ ] Decide on NVMe capacity (1.92TB vs 3.84TB)
- [ ] Confirm colocation provider and contract
- [ ] Get colocation network details:
  - [ ] IP addressing scheme
  - [ ] VLAN IDs for management and data
  - [ ] Upstream bandwidth (1 Gbps? 10 Gbps?)
  - [ ] Monthly cost per U and per amp

### Order from HP
- [ ] Request quote from HP reseller (CDW, Insight, SHI, etc.)
- [ ] Include 3-year warranty (NBD on-site or 4-hour response)
- [ ] Add iLO Advanced licenses
- [ ] Order SFP+ cables or transceivers (if not using colo's)

### Colocation Setup
- [ ] Sign colocation contract
- [ ] Request rack space (1/4 or 1/2 rack)
- [ ] Request power (2× 15A circuits or 1× 16A 230V)
- [ ] Request network ports (6× 10G data + 3× 1G management)
- [ ] Set up VPN for remote access to iLO
- [ ] Arrange shipping address for servers

### Before Shipping to Colo
- [ ] Unbox and test servers at your location (if possible)
- [ ] Update firmware via HP Service Pack for ProLiant (SPP)
- [ ] Configure iLO with static IP or DHCP reservation
- [ ] Set strong iLO password (store in password manager)
- [ ] Document MAC addresses for iLO and NICs
- [ ] Create RAID 1 array for boot disks
- [ ] Test boot from virtual media (mount Ubuntu ISO via iLO)

---

## Estimated Costs (New Hardware)

| Item | Qty | Unit Price | Total |
|------|-----|------------|-------|
| **HP DL380 Gen11** (base, 64C/192G) | 2 | $12,000 | $24,000 |
| **HP DL380 Gen11** (heavy, 64C/256G) | 1 | $15,000 | $15,000 |
| **NVMe SSDs** (1.92TB, extra) | 8 | $300 | $2,400 |
| **SATA SSDs** (480GB boot) | 6 | $100 | $600 |
| **iLO Advanced licenses** | 3 | $250 | $750 |
| **SFP+ DAC cables** (3m) | 6 | $30 | $180 |
| **Rack rails** (included) | 3 | $0 | $0 |
| **3-year NBD warranty** | 3 | $500 | $1,500 |
| **SUBTOTAL** | | | **$44,430** |
| **Colocation** (1/2 rack, 6 months) | 6 | $500/mo | $3,000 |
| **TOTAL (6 months)** | | | **$47,430** |

*Prices are estimates; actual quotes will vary*

**Colocation monthly costs** (typical):
- 1/2 rack space: $200-400/month
- 2-3 kW power: $100-200/month
- 10 Gbps network: $100-300/month
- **Total**: ~$500-1000/month depending on provider

---

## Next Steps

1. **Get quote** from HP reseller with exact part numbers
2. **Sign colocation agreement** and confirm network setup
3. **Order servers** (lead time: 4-8 weeks)
4. **Configure iLO** and test remotely before shipping
5. **Ship to colocation** with detailed setup instructions
6. **Install hypervisor** (Proxmox or ESXi) via iLO virtual media
7. **Deploy VMs** and install Kubernetes

See next documents for hypervisor installation and Kubernetes setup.
