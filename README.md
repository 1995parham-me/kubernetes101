# Kubernetes 101: Production Cluster on Bare Metal

Complete guide for deploying a production-ready Kubernetes cluster on HP ProLiant Gen11 servers in a colocation datacenter.

## Architecture Overview

**Infrastructure:**

- **3× HP ProLiant DL380 Gen11** servers (colocation)
- **Hypervisor**: Proxmox VE (recommended) or VMware ESXi
- **Kubernetes**: 8 VMs (1 master + 7 workers)
  - 4 workers for PHP applications (16 cores, 64 GB each)
  - 3 workers for PostgreSQL databases (16 cores, 64 GB each)
- **Storage**: Local NVMe SSDs (no SAN/NAS)
- **Database**: PostgreSQL HA cluster with CloudNativePG

**Total Resources:**

- 120 vCPUs (from ~176 physical cores)
- 464 GB RAM (from 640 GB physical)
- ~3 TB storage across NVMe drives

**Key Features:**

- ✅ High availability (tolerate 1 server failure)
- ✅ Dedicated PostgreSQL workers (isolated from apps)
- ✅ Local storage for performance
- ✅ Automatic failover for PostgreSQL
- ✅ Production-ready monitoring and backups

---

## Documentation Index

### Phase 1: Planning

1. **[01-architecture-overview.md](01-architecture-overview.md)**
   - System architecture and design decisions
   - VM distribution across physical hosts
   - Hypervisor comparison (Proxmox vs ESXi)
   - Resource allocation strategy

2. **[02-hp-gen11-hardware-specs.md](02-hp-gen11-hardware-specs.md)**
   - Detailed HP DL380 Gen11 server specifications
   - CPU, RAM, storage configurations
   - HP part numbers for ordering
   - Cost estimates and procurement checklist

### Phase 2: Infrastructure Setup

3. **[03-colocation-setup-guide.md](03-colocation-setup-guide.md)**
   - Colocation datacenter requirements
   - Network topology (VLANs, IP addressing)
   - Physical installation procedures
   - Remote management via iLO 6
   - Power, cooling, and SLA considerations

4. **[04-hypervisor-installation.md](04-hypervisor-installation.md)**
   - Proxmox VE installation (remote via iLO)
   - VMware ESXi installation (alternative)
   - Network bonding and VLAN configuration
   - Storage configuration (NVMe, RAID)
   - VM template creation

### Phase 3: Kubernetes Deployment

5. **[05-kubernetes-installation.md](05-kubernetes-installation.md)**
   - Complete Kubernetes cluster setup with kubeadm
   - Container runtime (containerd) configuration
   - Basic cluster initialization
   - Node labeling and resource management
   - Backup and disaster recovery

6. **[09-cilium-networking.md](09-cilium-networking.md)**
   - Cilium CNI installation (eBPF-based networking)
   - Hubble observability setup (network flow visualization)
   - kube-proxy replacement for better performance
   - Network policies for PostgreSQL access control
   - WireGuard encryption for database traffic
   - Performance tuning for low-latency connections

7. **[08-storage-topolvm-setup.md](08-storage-topolvm-setup.md)**
   - TopoLVM installation (production-grade storage)
   - LVM volume group configuration on each node
   - Storage classes for applications and databases
   - Volume snapshots for backups
   - Capacity-aware scheduling
   - Migration from basic storage provisioner

8. **[10-contour-ingress.md](10-contour-ingress.md)**
   - Contour ingress controller installation
   - HTTPProxy configuration (type-safe routing)
   - TLS automation with cert-manager
   - Advanced routing (path, header, weighted)
   - Canary and blue-green deployments
   - Rate limiting and protection

### Phase 4: Database Layer

9. **[06-postgresql-cnpg-setup.md](06-postgresql-cnpg-setup.md)**
   - CloudNativePG operator installation
   - PostgreSQL HA cluster deployment (3 replicas)
   - Node affinity and anti-affinity rules
   - Connection pooling with PgBouncer
   - Backup configuration (S3/MinIO)
   - Monitoring with Prometheus/Grafana
   - Performance tuning for PHP workloads

### Phase 5: Maintenance & Upgrades

10. **[07-upgrade-strategy.md](07-upgrade-strategy.md)**
   - Comprehensive upgrade strategy for all components
   - Ubuntu 24.04 LTS upgrade path (support until 2029)
   - Kubernetes version upgrades (patch and minor)
   - PostgreSQL major/minor version upgrades with CNPG
   - Proxmox VE hypervisor updates
   - Application deployment strategies (rolling, blue-green, canary)
   - Backup procedures before upgrades
   - Rollback and disaster recovery
   - Quarterly maintenance schedule template
   - Automation with Ansible

---

## Quick Start

### 1. Hardware Procurement

- Review [02-hp-gen11-hardware-specs.md](02-hp-gen11-hardware-specs.md)
- Order 3× HP DL380 Gen11 servers with recommended specs
- Sign colocation agreement (see [03-colocation-setup-guide.md](03-colocation-setup-guide.md))

### 2. Initial Server Setup

- Configure iLO on all servers (see [03-colocation-setup-guide.md](03-colocation-setup-guide.md#physical-installation-steps))
- Update firmware via HP Service Pack for ProLiant (SPP)
- Ship servers to colocation datacenter

### 3. Install Hypervisor

- Follow [04-hypervisor-installation.md](04-hypervisor-installation.md)
- Install Proxmox VE on all 3 servers
- Configure networking (VLANs, bonds)
- Create Proxmox cluster (optional but recommended)
- Create VM templates

### 4. Deploy Kubernetes

- Follow [05-kubernetes-installation.md](05-kubernetes-installation.md)
- Create 8 VMs from template
- Initialize Kubernetes cluster with kubeadm (without CNI initially)
- Label nodes (application vs database)

### 4.5 Setup Networking (Cilium)

- Follow [09-cilium-networking.md](09-cilium-networking.md)
- Install Cilium CNI with eBPF
- Replace kube-proxy for better performance
- Enable Hubble observability
- Configure network policies
- Enable WireGuard encryption

### 4.6 Setup Storage (TopoLVM)

- Follow [08-storage-topolvm-setup.md](08-storage-topolvm-setup.md)
- Configure LVM on all worker nodes
- Install TopoLVM CSI driver
- Create storage classes
- Enable volume snapshots

### 4.7 Setup Ingress (Contour)

- Follow [10-contour-ingress.md](10-contour-ingress.md)
- Install Contour with Envoy
- Configure MetalLB for LoadBalancer IPs (or use NodePort)
- Set up cert-manager for TLS
- Create HTTPProxy for applications

### 5. Deploy PostgreSQL

- Follow [06-postgresql-cnpg-setup.md](06-postgresql-cnpg-setup.md)
- Install CloudNativePG operator
- Deploy 3-replica PostgreSQL cluster (using TopoLVM storage)
- Configure backups and monitoring
- Set up volume snapshots for instant backups

### 6. Deploy Your Applications

- Create Kubernetes deployments for PHP apps
- Use PgBouncer for database connections
- Configure HTTPProxy for routing and TLS
- Set up CI/CD pipeline
- Implement canary or blue-green deployments

### 7. Establish Upgrade Strategy

- Review [07-upgrade-strategy.md](07-upgrade-strategy.md)
- Set up quarterly maintenance schedule
- Configure automated backups (etcd, PostgreSQL, VM snapshots)
- Test rollback procedures
- Document cluster-specific configurations

---

## Network Topology

```
Colocation Datacenter
├── VLAN 10: Management (192.168.10.0/24)
│   ├── 192.168.10.11 - k8s-host-01 iLO
│   ├── 192.168.10.12 - k8s-host-02 iLO
│   └── 192.168.10.13 - k8s-host-03 iLO
│
├── VLAN 20: Hypervisor Management (10.10.20.0/24)
│   ├── 10.10.20.11 - k8s-host-01 Proxmox
│   ├── 10.10.20.12 - k8s-host-02 Proxmox
│   └── 10.10.20.13 - k8s-host-03 Proxmox
│
└── VLAN 30: Kubernetes Cluster (10.10.30.0/24)
    ├── 10.10.30.10 - k8s-master-01
    ├── 10.10.30.21-24 - k8s-app-worker-01 to 04
    └── 10.10.30.31-33 - k8s-pg-worker-01 to 03
```

---

## VM Distribution

### Server 1 (k8s-host-01)

- k8s-master-01 (8 vCPU, 16 GB) - Control plane
- k8s-app-worker-01 (16 vCPU, 64 GB) - PHP apps
- k8s-pg-worker-01 (16 vCPU, 64 GB) - PostgreSQL

### Server 2 (k8s-host-02)

- k8s-app-worker-02 (16 vCPU, 64 GB) - PHP apps
- k8s-app-worker-03 (16 vCPU, 64 GB) - PHP apps
- k8s-pg-worker-02 (16 vCPU, 64 GB) - PostgreSQL

### Server 3 (k8s-host-03)

- k8s-app-worker-04 (16 vCPU, 64 GB) - PHP apps
- k8s-pg-worker-03 (16 vCPU, 64 GB) - PostgreSQL

**HA Benefits:**

- Each PostgreSQL replica on different physical host
- App workers spread across all 3 hosts
- Can tolerate 1 server failure
- PostgreSQL quorum maintained with 2/3 replicas

---

## Why This Stack? (vs Traditional Alternatives)

| Component | Traditional Choice | **Our Choice** | Key Benefit |
|-----------|-------------------|----------------|-------------|
| **CNI** | Calico (iptables) | **Cilium** (eBPF) | 40% lower latency, built-in observability |
| **Storage** | local-path | **TopoLVM** (LVM) | Volume snapshots, capacity tracking |
| **Ingress** | Nginx | **Contour** (Envoy) | Zero-downtime updates, type-safe config |
| **Hypervisor** | ESXi Free | **Proxmox VE** | No vCPU limits, free clustering |
| **Database HA** | Patroni | **CNPG** | Kubernetes-native, simpler setup |

**Result**: Modern stack with better performance, observability, and $21K+/year savings vs commercial alternatives.

See [TECHNOLOGY-DECISIONS.md](TECHNOLOGY-DECISIONS.md) for detailed comparison.

---

## Technology Stack

### Infrastructure Layer

- **Bare Metal**: HP ProLiant DL380 Gen11
- **Hypervisor**: Proxmox VE 8.x (or VMware ESXi 8.0)
- **OS**: Ubuntu 24.04 LTS (Noble Numbat)
- **Storage**: Local NVMe SSDs with LVM

### Kubernetes Layer

- **Kubernetes**: v1.30.x (latest stable)
- **Container Runtime**: containerd
- **CNI**: Cilium (eBPF-based with Hubble observability)
- **Storage**: TopoLVM (LVM-based CSI with snapshots)
- **Ingress**: Contour (Envoy-based with HTTPProxy CRD)

### Database Layer

- **PostgreSQL**: 16.x
- **Operator**: CloudNativePG (CNPG)
- **Connection Pool**: PgBouncer
- **Backup**: Barman (via CNPG) → S3/MinIO
- **HA**: Synchronous replication (minSyncReplicas: 1)

### Monitoring & Observability

- **Metrics**: Prometheus
- **Visualization**: Grafana
- **Logs**: Loki (optional)
- **Tracing**: Jaeger (optional)

---

## Cost Breakdown

### One-Time Costs

| Item                      | Cost                   |
| ------------------------- | ---------------------- |
| 3× HP DL380 Gen11 servers | $35,000 - $45,000      |
| Colocation setup fees     | $500 - $1,000          |
| Shipping and installation | $400 - $600            |
| **Total**                 | **~$36,000 - $47,000** |

### Monthly Recurring Costs

| Item                 | Monthly Cost           |
| -------------------- | ---------------------- |
| 1/2 rack colocation  | $250 - $400            |
| Power (~2 kW)        | $150 - $300            |
| Bandwidth (10G port) | $100 - $200            |
| **Total**            | **~$500 - $900/month** |

**Annual TCO**: ~$42,000 - $58,000 (first year with hardware)
**Subsequent years**: ~$6,000 - $11,000/year (colo only)

### Long-Term Support

**Ubuntu 24.04 LTS Support:**

- Standard support until April 2029 (5 years)
- Extended Security Maintenance (ESM) until 2034 (paid)

**Kubernetes Support:**

- Latest 3 minor versions supported (~1 year rolling support)
- Recommended: Upgrade to new minor version every 6-12 months

**PostgreSQL Support:**

- Each major version supported for 5 years
- PostgreSQL 16: Supported until September 2028

---

## Performance Tuning

### PostgreSQL for PHP Workloads

- Use **PgBouncer** for connection pooling (PHP = many short connections)
- Use **read replicas** (pg-cluster-01-ro) for SELECT queries
- Tune `work_mem` for complex queries (256 MB recommended)
- Enable query logging for slow queries (>500ms)

### Kubernetes Resource Limits

```yaml
# PHP application pod
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "2Gi"
    cpu: "2000m"
```

### Network Performance

- Enable jumbo frames (MTU 9000) on all interfaces
- Use LACP bonding on 10G NICs
- Consider SR-IOV for maximum performance (advanced)

---

## Disaster Recovery

### Backup Strategy

- **PostgreSQL**: Automated daily backups to S3/MinIO (30-day retention)
- **etcd**: Weekly snapshots of Kubernetes cluster state
- **Persistent Volumes**: Application-specific backup (Velero)
- **Configurations**: Git repository for all YAML manifests

### Recovery Procedures

- **Server failure**: Kubernetes reschedules pods to healthy nodes
- **PostgreSQL primary failure**: Automatic failover (30-60 seconds)
- **Total cluster loss**: Restore from etcd backup + PostgreSQL backup
- **Data corruption**: Point-in-time recovery (PITR) via CNPG

---

## Security Best Practices

### Infrastructure Security

- iLO access via VPN only (never expose to internet)
- Strong passwords (20+ chars) stored in password manager
- Regular firmware updates (quarterly)
- Network segmentation (VLANs for management, data, apps)

### Kubernetes Security

- RBAC enabled (default in kubeadm)
- Network policies (Calico supports NetworkPolicy)
- Pod Security Standards (enforce restricted policies)
- Secrets management (consider Sealed Secrets or Vault)
- Regular security updates (patch Kubernetes monthly)

### Database Security

- Strong PostgreSQL passwords (32+ chars)
- TLS for PostgreSQL connections (configure in CNPG)
- Regular backups with encryption at rest
- Limited superuser access (use app users with minimal privileges)

---

## Maintenance Schedule

### Weekly

- Check cluster health (`kubectl get nodes`)
- Review monitoring dashboards (Grafana)
- Check backup success (CNPG backups)

### Monthly

- Update Kubernetes patch versions (e.g., 1.30.1 → 1.30.2)
- Update application containers
- Review resource usage (CPU, RAM, disk)

### Quarterly

- Update server firmware (BIOS, iLO, NIC)
- Update Proxmox VE
- Test disaster recovery procedures
- Review colocation SLA compliance

### Annually

- Kubernetes minor version upgrade (e.g., 1.30 → 1.31)
- PostgreSQL major version upgrade (e.g., 16 → 17)
- Hardware health assessment
- Capacity planning review

---

## Troubleshooting

### Common Issues

**Pod stuck in Pending:**

```bash
kubectl describe pod <pod-name>
# Check: Node resources, PVC binding, node affinity
```

**PostgreSQL failover not working:**

```bash
kubectl logs -n cnpg-system deployment/cnpg-cloudnative-pg
kubectl describe cluster pg-cluster-01
```

**Node NotReady:**

```bash
ssh root@<node-ip>
sudo systemctl status kubelet
sudo journalctl -u kubelet -f
```

**Network connectivity issues:**

```bash
kubectl get pods -n calico-system
kubectl logs -n calico-system <calico-pod>
```

---

## Support and Resources

### Official Documentation

- [Kubernetes Docs](https://kubernetes.io/docs/)
- [Proxmox VE Docs](https://pve.proxmox.com/pve-docs/)
- [CloudNativePG Docs](https://cloudnative-pg.io/documentation/)
- [HP Support](https://support.hpe.com/)

### Community

- [Proxmox Forum](https://forum.proxmox.com/)
- [Kubernetes Slack](https://kubernetes.slack.com/)
- [CNPG Slack](https://cloudnativepg.slack.com/)

### Professional Support

- HP: Hardware support (NBD or 4-hour response)
- Proxmox: Subscription support (optional)
- EDB: PostgreSQL support (CloudNativePG maintainer)

---

## Next Steps After Deployment

1. **Application Migration**
   - Containerize PHP applications
   - Create Kubernetes manifests
   - Set up CI/CD pipeline (GitLab CI, GitHub Actions)

2. **Advanced Features**
   - Install Ingress Controller (Nginx/Traefik)
   - Configure TLS certificates (Let's Encrypt + cert-manager)
   - Set up monitoring stack (Prometheus, Grafana, Loki)
   - Implement GitOps (ArgoCD or Flux)

3. **Scaling**
   - Add more worker nodes if needed
   - Implement Horizontal Pod Autoscaler (HPA)
   - Consider multi-master setup for control plane HA

4. **Optimization**
   - Profile application performance
   - Optimize database queries
   - Fine-tune resource requests/limits
   - Implement caching (Redis, Memcached)
