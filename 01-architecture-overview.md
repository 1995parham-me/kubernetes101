# Kubernetes Cluster Architecture on Bare Metal

## Overview

- **3× HP ProLiant DL380 Gen11 servers** in colocation datacenter
- **Hypervisor**: Proxmox VE (recommended) or VMware ESXi (paid)
- **8 Kubernetes VMs**: 1 master (control plane) + 7 workers
  - **4 workers** dedicated to PHP applications
  - **3 workers** dedicated to PostgreSQL databases
- **Workload**: PHP-based applications (memory-heavy)
- **Storage**: Local disks only (no SAN/NAS)
- **Database**: PostgreSQL cluster using CloudNativePG (CNPG) on dedicated nodes

## Resource Requirements

### Per Worker Node
- **CPU**: 16 cores
- **RAM**: 64 GB
- **Disk**: Minimum 200 GB (recommend 500 GB for app data + PostgreSQL)

### Master Node (Control Plane)
- **CPU**: 4-8 cores
- **RAM**: 8-16 GB
- **Disk**: 100 GB

### Total Requirements
- **CPU**: 120 cores (7 workers × 16 + 1 master × 8)
- **RAM**: 464 GB (7 workers × 64 + 1 master × 16)
- **Disk**: ~3 TB
  - Master: 100 GB
  - Application workers (4×): 4 × 500 GB = 2 TB
  - PostgreSQL workers (3×): 3 × 300 GB = 900 GB

## Bare Metal Server Specifications Needed

To run this setup, each of your 3 bare metal servers should have **AT MINIMUM**:

- **CPU**: 40-44 physical cores (with hyperthreading = 80-88 vCPUs)
- **RAM**: 160-192 GB DDR4 ECC
- **Disk**:
  - Boot: 2× 240 GB SSD (RAID 1 for hypervisor)
  - Data: 2-4× 1 TB NVMe SSD (for VM storage)
- **Network**: Dual 10 Gbps NICs (for redundancy and bonding)

**Total across 3 servers**: ~128 physical cores, 512+ GB RAM, 6-12 TB storage

## VM Distribution Strategy (8 VMs across 3 Bare Metal)

### Recommended Distribution for HA

```
Bare Metal Server 1 (40 vCPUs allocated, 160 GB RAM):
  - k8s-master-01       (8 cores, 16 GB)   [Control Plane]
  - k8s-app-worker-01   (16 cores, 64 GB)  [PHP Applications]
  - k8s-app-worker-02   (16 cores, 64 GB)  [PHP Applications]
  Total: 40 cores, 144 GB

Bare Metal Server 2 (40 vCPUs allocated, 160 GB RAM):
  - k8s-app-worker-03   (16 cores, 64 GB)  [PHP Applications]
  - k8s-pg-worker-01    (16 cores, 64 GB)  [PostgreSQL]
  - k8s-pg-worker-02    (16 cores, 64 GB)  [PostgreSQL]
  Total: 48 cores, 192 GB ⚠️ Needs more resources

Bare Metal Server 3 (40 vCPUs allocated, 160 GB RAM):
  - k8s-app-worker-04   (16 cores, 64 GB)  [PHP Applications]
  - k8s-pg-worker-03    (16 cores, 64 GB)  [PostgreSQL]
  Total: 32 cores, 128 GB
```

### Alternative: Balanced Distribution

```
Bare Metal Server 1 (40 vCPUs, 160 GB):
  - k8s-master-01       (8 cores, 16 GB)   [Control Plane]
  - k8s-app-worker-01   (16 cores, 64 GB)  [PHP Applications]
  - k8s-pg-worker-01    (16 cores, 64 GB)  [PostgreSQL]

Bare Metal Server 2 (48 vCPUs, 192 GB):
  - k8s-app-worker-02   (16 cores, 64 GB)  [PHP Applications]
  - k8s-app-worker-03   (16 cores, 64 GB)  [PHP Applications]
  - k8s-pg-worker-02    (16 cores, 64 GB)  [PostgreSQL]

Bare Metal Server 3 (32 vCPUs, 128 GB):
  - k8s-app-worker-04   (16 cores, 64 GB)  [PHP Applications]
  - k8s-pg-worker-03    (16 cores, 64 GB)  [PostgreSQL]
```

**HA Benefits**:
- Each PostgreSQL worker on different physical host (3-way replication)
- Application workers spread across all 3 hosts
- Can tolerate 1 bare metal server failure
- PostgreSQL remains available with 2/3 replicas if 1 server down

**Constraints**:
- Bare Metal 2 needs more resources (48 cores) - consider 2-socket server
- Anti-affinity rules ensure PostgreSQL replicas never co-locate on same physical host

## Hypervisor Choice

### Option A: VMware ESXi + vCenter (PAID)
- **Cost**: ~$995/CPU for Standard, ~$3495/CPU for Enterprise Plus
- **Pros**: Industry standard, mature, excellent management
- **Cons**: Expensive, requires licensing

### Option B: VMware ESXi Free
- **Cost**: Free
- **Pros**: Reliable, VMware quality
- **Cons**:
  - **NO vCenter support**
  - **NO API access**
  - Manage each host individually
  - No vMotion, no HA
  - Limited to 8 vCPUs per VM (won't work for 16-core workers!)

⚠️ **ESXi Free WILL NOT WORK** - 8 vCPU limit prevents 16-core workers

### Option C: Proxmox VE (FREE & RECOMMENDED)
- **Cost**: Free (open source)
- **Pros**:
  - Supports clustering (like vCenter)
  - Built-in HA
  - Live migration
  - Web-based management
  - Full API support
  - Active community
  - No vCPU limits
- **Cons**:
  - Different from VMware ecosystem
  - Paid support optional

### Option D: Harvester (FREE, Kubernetes-native)
- **Cost**: Free (SUSE open source)
- **Pros**:
  - Built on Kubernetes (K3s)
  - VM management via Kubernetes
  - Modern architecture
- **Cons**:
  - Newer, smaller community
  - More complex

## Recommendation

**Use Proxmox VE** for your use case because:
1. Free and feature-complete
2. Supports clustering across 3 nodes
3. No vCPU limitations
4. Built-in HA and live migration
5. Excellent for local storage setups
6. Easy VM provisioning via API/Terraform
