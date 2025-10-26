# Technology Stack Decisions

## Summary of Chosen Technologies

This document explains the technology choices for this production Kubernetes cluster and why they were selected.

---

## Core Infrastructure

### ✅ Ubuntu 24.04 LTS (Noble Numbat)

**Chosen over**: Ubuntu 22.04, Rocky Linux, Debian

**Why**:
- 5-year support until April 2029
- Latest kernel (6.8+) with better eBPF support for Cilium
- Extended Security Maintenance available until 2034
- Native containerd and Kubernetes support

**Alternatives considered**:
- Ubuntu 22.04 LTS: Shorter support (until 2027)
- Rocky Linux 9: Good choice, but Ubuntu has better cloud-init support
- Debian 12: More conservative, less frequent updates

---

## Hypervisor

### ✅ Proxmox VE 8.x

**Chosen over**: VMware ESXi, Harvester

**Why**:
- **Free and open-source** (no licensing costs)
- Full-featured clustering (like vCenter but free)
- Built-in HA and live migration
- No vCPU limits (ESXi Free has 8 vCPU limit)
- Excellent web UI and API
- LVM/ZFS storage support

**Cost savings**: ~$12,000-18,000 vs VMware vCenter Standard

**Alternatives considered**:
- VMware ESXi + vCenter: Excellent but expensive ($6K-12K)
- VMware ESXi Free: 8 vCPU limit makes it unusable for 16-core workers
- Harvester: Interesting but less mature than Proxmox

---

## Networking (CNI)

### ✅ Cilium (eBPF-based)

**Chosen over**: Calico, Flannel, Weave

**Why**:
- **eBPF-native**: Kernel-level networking, no iptables overhead
- **40% lower latency**: Critical for PHP → PostgreSQL connections
- **Built-in observability**: Hubble provides network flow visualization
- **kube-proxy replacement**: Faster service load balancing
- **Service mesh**: Built-in mTLS, load balancing (no Istio needed)
- **WireGuard encryption**: Secure database traffic
- **Lower CPU usage**: 40% less CPU for networking tasks

**Performance comparison** (typical):
```
Calico:  Pod-to-pod latency: 0.5ms, CPU usage: 5%
Cilium:  Pod-to-pod latency: 0.3ms, CPU usage: 3%
         = 40% faster, 40% less CPU
```

**Perfect for**:
- Low-latency database connections (PostgreSQL)
- High-throughput replication between PostgreSQL replicas
- Network troubleshooting with Hubble UI

**Alternatives considered**:
- Calico: Good, but iptables-based (slower)
- Flannel: Too simple, lacks network policies
- Weave: Deprecated

---

## Storage

### ✅ TopoLVM (LVM-based CSI)

**Chosen over**: local-path-provisioner, OpenEBS Device, raw block devices

**Why**:
- **Dynamic provisioning** with LVM thin pools
- **Volume snapshots**: Instant PostgreSQL backups (copy-on-write)
- **Capacity-aware scheduling**: Kubernetes knows available storage per node
- **Thin provisioning**: Efficient disk usage (can overcommit safely)
- **Volume expansion**: Grow PVs online without downtime
- **Native performance**: Direct LVM access, same speed as raw disks

**Example efficiency**:
```
With raw block devices:
  2 TB NVMe → 1 PV (100 GB used) = 1.9 TB wasted ❌

With TopoLVM:
  2 TB NVMe → LVM VG
    ├── PV 1: 100 GB (PostgreSQL)
    ├── PV 2: 200 GB (app data)
    ├── PV 3: 100 GB (another PG replica)
    └── Free: 1.6 TB ✅
```

**Perfect for**:
- PostgreSQL with CNPG (uses volume snapshots)
- Multiple PVs per physical disk
- Production environments needing snapshots

**Alternatives considered**:
- local-path-provisioner: Too basic, no snapshots
- OpenEBS Device Manager: Wasteful (1 disk = 1 PV)
- OpenEBS LVM: Very similar to TopoLVM, both good choices

---

## Ingress Controller

### ✅ Contour (Envoy-based)

**Chosen over**: Nginx Ingress, Traefik, HAProxy

**Why**:
- **Envoy proxy**: Modern, high-performance (used by Istio, AWS App Mesh)
- **HTTPProxy CRD**: Type-safe configuration (not untyped annotations)
- **Zero-downtime updates**: Hot reload config (Nginx requires reload)
- **Better HTTP/2**: Native support, better for modern browsers
- **Native gRPC**: If you add gRPC services later
- **Canary deployments**: Built-in weight-based routing
- **Rich observability**: Envoy's excellent metrics

**Configuration comparison**:
```yaml
# Nginx Ingress (annotation hell)
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    # ... 20+ possible annotations, untyped, easy to break

# Contour (clean, type-safe)
spec:
  virtualhost:
    fqdn: app.example.com
    tls:
      secretName: app-tls
  routes:
  - conditions:
    - prefix: /api
    services:
    - name: api-service
      port: 80
```

**Perfect for**:
- Zero-downtime deployments (PHP apps)
- Clean, maintainable configuration
- Blue-green and canary deployments

**Alternatives considered**:
- Nginx Ingress: Good, but annotation-based config is messy
- Traefik: Good, but less mature than Contour for Kubernetes
- HAProxy: Excellent performance, but harder to configure

---

## Database

### ✅ CloudNativePG (CNPG)

**Chosen over**: Patroni, Stolon, CrunchyData PGO

**Why**:
- **Cloud-native design**: Built specifically for Kubernetes
- **Automatic failover**: 30-60 second RTO (Recovery Time Objective)
- **Synchronous replication**: Zero data loss (RPO = 0)
- **Built-in backup**: Barman integration for S3/MinIO backups
- **Connection pooling**: Built-in PgBouncer support
- **Easy upgrades**: Rolling upgrades for minor versions
- **Active development**: Maintained by EDB (PostgreSQL experts)

**Perfect for**:
- Production PostgreSQL on Kubernetes
- HA requirements with automatic failover
- Integration with TopoLVM for snapshots

**Alternatives considered**:
- Patroni: Good, but requires etcd/Consul (extra complexity)
- Stolon: Older, less Kubernetes-native
- CrunchyData PGO: Good alternative, but CNPG is simpler

---

## Modern Stack Benefits

### Performance Improvements

| Component | Traditional | Modern | Improvement |
|-----------|-------------|--------|-------------|
| **Networking** | Calico + kube-proxy | Cilium eBPF | 40% faster latency |
| **Storage** | local-path | TopoLVM | Snapshots + capacity tracking |
| **Ingress** | Nginx | Contour/Envoy | Zero-downtime updates |
| **Observability** | Basic metrics | Hubble + Envoy metrics | Full visibility |

### Total Cost of Ownership (TCO)

**Hardware** (one-time):
- 3× HP DL380 Gen11: $35,000 - $45,000

**Software** (annual):
- Proxmox VE: **$0** (vs VMware vCenter: $6,000-12,000)
- Cilium: **$0** (vs commercial service mesh: $5,000+)
- Contour: **$0** (all open-source)
- TopoLVM: **$0**
- CNPG: **$0** (vs commercial DBaaS: $10,000+)

**Annual software savings**: ~$21,000+ vs commercial alternatives

**Colocation** (recurring):
- 1/2 rack + power + bandwidth: $500-900/month = $6,000-11,000/year

**Total first year**: ~$42,000-58,000 (hardware + colo)
**Subsequent years**: ~$6,000-11,000/year (colo only)

---

## Technology Matrix

| Layer | Technology | Type | License | Support Until |
|-------|------------|------|---------|---------------|
| **OS** | Ubuntu 24.04 LTS | Linux | Free | April 2029 |
| **Hypervisor** | Proxmox VE 8.x | KVM/QEMU | AGPL-3.0 | Ongoing |
| **Orchestration** | Kubernetes 1.30+ | Container | Apache 2.0 | Rolling |
| **Container Runtime** | containerd | Runtime | Apache 2.0 | Ongoing |
| **CNI** | Cilium | Networking | Apache 2.0 | Ongoing |
| **Storage CSI** | TopoLVM | Storage | Apache 2.0 | Ongoing |
| **Ingress** | Contour | L7 Proxy | Apache 2.0 | Ongoing |
| **Database Operator** | CNPG | Operator | Apache 2.0 | Ongoing |
| **Database** | PostgreSQL 16 | RDBMS | PostgreSQL | Sept 2028 |
| **Observability** | Hubble + Prometheus | Monitoring | Apache 2.0 | Ongoing |

**Everything is open-source and free!**

---

## Decision Criteria

### Why Modern Stack Over Traditional?

**Traditional Stack** (common in 2020-2022):
- Calico (iptables-based)
- local-path-provisioner
- Nginx Ingress
- Manual PostgreSQL management

**Modern Stack** (2024-2025):
- Cilium (eBPF-based)
- TopoLVM (LVM with snapshots)
- Contour (Envoy-based)
- CNPG (Kubernetes-native PostgreSQL)

**Key improvements**:
1. **Performance**: eBPF networking is 40% faster
2. **Observability**: Hubble provides full network visibility
3. **Operations**: Zero-downtime config updates (Contour)
4. **Storage**: Snapshots and capacity tracking (TopoLVM)
5. **Database**: Automatic failover and backups (CNPG)

---

## When to Reconsider These Choices

### You might choose differently if:

**Use VMware ESXi if**:
- You already have VMware licenses
- Your team is very familiar with VMware
- You need official VMware support

**Use Calico if**:
- Your kernel is too old for eBPF (< 4.19)
- You need Windows node support (Cilium doesn't support Windows)
- Your team is very familiar with Calico

**Use Nginx Ingress if**:
- You're very familiar with Nginx configuration
- You need specific Nginx modules
- You prefer annotation-based config

**Use local-path-provisioner if**:
- You're running dev/test environments
- You don't need snapshots or capacity tracking
- Simplicity is more important than features

---

## Future-Proofing

### Technology Longevity

All chosen technologies have:
- ✅ Active development and communities
- ✅ CNCF graduated or incubating projects
- ✅ Used in production by major companies
- ✅ Good upgrade paths and backwards compatibility

### Upgrade Path (Next 5 Years)

**2025-2027**: Stable on current stack
- Ubuntu 24.04 → security updates only
- Kubernetes 1.30 → 1.31 → 1.32 → 1.33
- PostgreSQL 16 → 17 → 18
- Cilium, Contour, TopoLVM: regular updates

**2027-2029**: Plan major upgrades
- Ubuntu 24.04 → 26.04 LTS (if needed)
- PostgreSQL → version 20+
- Kubernetes → version 1.38+

**Beyond 2029**:
- Ubuntu 24.04 ESM support until 2034 (paid)
- Or migrate to Ubuntu 28.04 LTS

---

## Summary

**This stack provides**:
- ✅ **Best performance**: eBPF networking, native LVM storage
- ✅ **Best observability**: Hubble network flows, Envoy metrics
- ✅ **Best operations**: Zero-downtime updates, automatic failover
- ✅ **Best cost**: $0 in software licenses, ~$21K/year savings
- ✅ **Best longevity**: All technologies have 5+ year support

**Perfect for**:
- Production workloads (PHP + PostgreSQL)
- Bare metal deployments
- Teams wanting modern, efficient infrastructure
- Organizations minimizing cloud costs

**Your cluster will be**:
- Modern and performant
- Well-supported and maintainable
- Cost-effective
- Production-ready

This is a **2024-2025 best-practice Kubernetes stack** for bare metal production deployments.
