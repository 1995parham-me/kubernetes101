# Upgrade Strategy and Maintenance Guide

## Overview

This guide provides a comprehensive upgrade strategy for all components of the Kubernetes cluster, ensuring minimal downtime and safe rollback capabilities.

**Upgrade Philosophy:**
- 🔄 **Rolling upgrades** - update one component at a time
- 📸 **Snapshot before upgrade** - always have a rollback path
- 🧪 **Test in dev first** - if possible, test upgrades on a dev cluster
- 📋 **Follow vendor release notes** - check for breaking changes
- ⏱️ **Schedule during maintenance windows** - minimize production impact

---

## Upgrade Timeline and Support

### Ubuntu 24.04 LTS (Noble Numbat)

**Support Timeline:**
- **Release Date**: April 2024
- **Standard Support**: Until April 2029 (5 years)
- **ESM Support**: Until April 2034 (10 years, paid)

**Upgrade Path:**
```
Ubuntu 24.04 → 24.10 (optional, interim) → 26.04 LTS (2026)
```

**Recommendation**: Stay on LTS releases (24.04, 26.04, etc.) for stability.

### Kubernetes Version Support

**Support Policy:**
- Kubernetes supports **latest 3 minor versions** (e.g., 1.30, 1.29, 1.28)
- New minor version every **~4 months**
- Patch releases every **~1 month**

**Upgrade Cadence:**
- **Patch upgrades** (1.30.1 → 1.30.2): Monthly, low risk
- **Minor upgrades** (1.30.x → 1.31.0): Every 6-12 months
- **Never skip minor versions** (must go 1.30 → 1.31 → 1.32, not 1.30 → 1.32)

### PostgreSQL Version Support

**Support Policy:**
- Each major version supported for **5 years**
- PostgreSQL 16: Released Sept 2023, supported until Sept 2028

**Upgrade Path:**
```
PostgreSQL 16 → 17 (2024) → 18 (2025) → ...
```

---

## Part 1: OS Upgrades (Ubuntu)

### Strategy: Rolling Node Upgrade

Upgrade nodes one at a time to avoid cluster downtime.

### 1.1 Minor Updates (Security Patches)

**Frequency**: Weekly or monthly

**Process** (per node):

```bash
# SSH to node
ssh root@k8s-app-worker-01

# Update package list
sudo apt update

# See available updates
sudo apt list --upgradable

# Upgrade packages (excluding kernel if you want to defer reboot)
sudo apt upgrade -y

# If kernel updated, reboot required
# First, drain node
kubectl drain k8s-app-worker-01 --ignore-daemonsets --delete-emptydir-data

# Reboot
sudo reboot

# After reboot, uncordon node
kubectl uncordon k8s-app-worker-01

# Verify node is Ready
kubectl get nodes
```

**Automate with `unattended-upgrades`** (optional):

```bash
# Install unattended-upgrades
sudo apt install unattended-upgrades -y

# Configure to auto-install security updates
sudo dpkg-reconfigure -plow unattended-upgrades

# Edit config
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades

# Enable automatic reboot (optional, requires testing)
Unattended-Upgrade::Automatic-Reboot "false";  # Set to true if you want auto-reboot
```

**Best Practice**: Upgrade workers first, then master.

### 1.2 Major Version Upgrade (24.04 → 26.04)

**When**: Every 2 years (when new LTS is released)

**Pre-Upgrade Checklist:**
- [ ] Backup etcd (cluster state)
- [ ] Snapshot all VMs in Proxmox
- [ ] Test upgrade on one worker node first
- [ ] Check Kubernetes compatibility with new Ubuntu version

**Process** (per node):

```bash
# Drain node
kubectl drain k8s-app-worker-01 --ignore-daemonsets --delete-emptydir-data

# SSH to node
ssh root@k8s-app-worker-01

# Update current system
sudo apt update && sudo apt upgrade -y && sudo apt dist-upgrade -y

# Install update-manager
sudo apt install update-manager-core -y

# Ensure Prompt=lts in config
sudo nano /etc/update-manager/release-upgrades
# Set: Prompt=lts

# Run upgrade (interactive)
sudo do-release-upgrade

# Follow prompts
# After upgrade completes, reboot
sudo reboot

# Verify new version
lsb_release -a
# Should show: Ubuntu 26.04 LTS

# Uncordon node
kubectl uncordon k8s-app-worker-01
```

**Order of upgrade:**
1. Upgrade **one worker** node first (test)
2. Wait 24-48 hours, monitor for issues
3. Upgrade remaining **worker nodes** (one at a time)
4. Upgrade **master node** last

**Rollback**: If upgrade fails, restore from VM snapshot in Proxmox.

---

## Part 2: Kubernetes Upgrades

### Strategy: Control Plane First, Then Workers

**Kubernetes Upgrade Policy:**
- **Control plane** can be **N+1** version ahead of workers
- Example: Master on 1.31.0, workers can be 1.30.x
- Workers must be upgraded within one minor version

### 2.1 Patch Upgrade (1.30.1 → 1.30.2)

**Low risk**, can be done monthly.

**On Master Node:**

```bash
# Check current version
kubectl version --short

# Update apt package list
sudo apt update

# Check available kubeadm versions
apt-cache madison kubeadm | grep 1.30

# Upgrade kubeadm
sudo apt-mark unhold kubeadm
sudo apt install -y kubeadm=1.30.2-1.1
sudo apt-mark hold kubeadm

# Verify kubeadm version
kubeadm version

# Plan upgrade
sudo kubeadm upgrade plan

# Apply upgrade
sudo kubeadm upgrade apply v1.30.2

# Drain master node
kubectl drain k8s-master-01 --ignore-daemonsets

# Upgrade kubelet and kubectl
sudo apt-mark unhold kubelet kubectl
sudo apt install -y kubelet=1.30.2-1.1 kubectl=1.30.2-1.1
sudo apt-mark hold kubelet kubectl

# Restart kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Uncordon master
kubectl uncordon k8s-master-01
```

**On Each Worker Node:**

```bash
# Drain worker
kubectl drain k8s-app-worker-01 --ignore-daemonsets --delete-emptydir-data

# SSH to worker
ssh root@k8s-app-worker-01

# Upgrade kubeadm
sudo apt-mark unhold kubeadm
sudo apt install -y kubeadm=1.30.2-1.1
sudo apt-mark hold kubeadm

# Upgrade node
sudo kubeadm upgrade node

# Upgrade kubelet and kubectl
sudo apt-mark unhold kubelet kubectl
sudo apt install -y kubelet=1.30.2-1.1 kubectl=1.30.2-1.1
sudo apt-mark hold kubelet kubectl

# Restart kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Exit SSH
exit

# Uncordon worker
kubectl uncordon k8s-app-worker-01
```

Repeat for all workers (one at a time).

### 2.2 Minor Version Upgrade (1.30.x → 1.31.0)

**Medium risk**, do every 6-12 months.

**Pre-Upgrade:**
- [ ] Read release notes: https://github.com/kubernetes/kubernetes/releases
- [ ] Check for deprecated APIs: `kubectl get apiservices`
- [ ] Backup etcd: `sudo etcdctl snapshot save`
- [ ] Snapshot all VMs

**Process**: Same as patch upgrade, but use version 1.31.0.

**Important**: Check for breaking changes in release notes!

**Example Breaking Changes to Watch:**
- API deprecations (e.g., `batch/v1beta1` CronJob removed in 1.25)
- Feature gate changes
- Component flag changes

**Verification After Upgrade:**

```bash
# Check all nodes
kubectl get nodes -o wide

# Check all pods
kubectl get pods -A

# Check API resources
kubectl api-resources

# Check component health
kubectl get componentstatuses

# Run cluster validation
kubectl cluster-info
kubectl get --raw /healthz
```

---

## Part 3: PostgreSQL Upgrades with CNPG

### Strategy: CNPG Handles Rolling Upgrades

CloudNativePG can perform **in-place upgrades** or **blue-green upgrades**.

### 3.1 Minor Version Upgrade (16.3 → 16.4)

**Very low risk**, done automatically by CNPG.

```bash
# Edit cluster manifest
kubectl edit cluster pg-cluster-01

# Update imageName
spec:
  imageName: ghcr.io/cloudnative-pg/postgresql:16.4  # Change from 16.3

# Save and exit
# CNPG will perform rolling upgrade automatically
# - Creates new replica with new version
# - Waits for it to sync
# - Promotes it to primary
# - Upgrades old primary as replica
# - Continues for all replicas

# Monitor upgrade progress
kubectl get cluster pg-cluster-01 -w

# Check pod versions
kubectl get pods -l cnpg.io/cluster=pg-cluster-01 -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
```

**Rollback**: Edit cluster again and revert to old image version.

### 3.2 Major Version Upgrade (16.x → 17.0)

**High risk**, requires careful planning.

**Two Options:**

#### Option A: In-Place Upgrade (Using pg_upgrade)

**Not recommended for production** - requires downtime.

#### Option B: Blue-Green Upgrade (Recommended)

Create new cluster with PostgreSQL 17, migrate data, switch over.

**Process:**

1. **Create new PostgreSQL 17 cluster:**

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: pg-cluster-02-pg17
spec:
  instances: 3
  imageName: ghcr.io/cloudnative-pg/postgresql:17.0

  # Copy all config from pg-cluster-01
  # ... (storage, resources, affinity, etc.)

  # Bootstrap from pg-cluster-01 backup
  bootstrap:
    recovery:
      source: pg-cluster-01

  externalClusters:
  - name: pg-cluster-01
    connectionParameters:
      host: pg-cluster-01-rw
      user: postgres
      dbname: postgres
    password:
      name: pg-cluster-01-superuser
      key: password
```

2. **Apply and wait for sync:**

```bash
kubectl apply -f pg-cluster-02-pg17.yaml

# Monitor replication
kubectl get cluster pg-cluster-02-pg17 -w
```

3. **Test new cluster:**

```bash
# Connect to new cluster
kubectl run psql-client -it --rm --restart=Never \
  --image=postgres:17 \
  --env="PGPASSWORD=$(kubectl get secret pg-cluster-02-pg17-superuser -o jsonpath='{.data.password}' | base64 -d)" \
  -- psql -h pg-cluster-02-pg17-rw -U postgres

# Run tests
\l
SELECT version();
```

4. **Switch application traffic:**

```bash
# Update application deployment
kubectl set env deployment/my-php-app \
  DB_HOST=pg-cluster-02-pg17-rw.default.svc.cluster.local
```

5. **Monitor for issues**, if all good, delete old cluster:

```bash
kubectl delete cluster pg-cluster-01
```

**Rollback**: Switch DB_HOST back to pg-cluster-01-rw.

---

## Part 4: Hypervisor Upgrades (Proxmox VE)

### Strategy: One Host at a Time (Live Migration)

**Proxmox Cluster Advantage**: Live migrate VMs to other hosts during upgrade.

### 4.1 Proxmox Patch Updates

**Frequency**: Monthly

**On each host** (one at a time):

```bash
# SSH to Proxmox host
ssh root@10.10.20.11

# Update package list
apt update

# See available updates
apt list --upgradable

# Upgrade Proxmox packages
apt full-upgrade -y

# If kernel updated, reboot required
# First, migrate VMs to other hosts

# In Proxmox web UI:
# Select k8s-host-01 → Migrate all VMs to k8s-host-02 and k8s-host-03

# Or via CLI:
# qm migrate 101 k8s-host-02  # Migrate VM 101 to host-02

# After all VMs migrated, reboot
reboot

# After reboot, verify cluster health
pvecm status
```

**Order**: Upgrade hosts one at a time (host-03, then host-02, then host-01).

### 4.2 Major Version Upgrade (Proxmox 8.x → 9.x)

**When**: Every 2-3 years

**Pre-Upgrade:**
- [ ] Backup all VMs
- [ ] Snapshot all VMs (Proxmox snapshots)
- [ ] Backup /etc/pve directory
- [ ] Read release notes: https://pve.proxmox.com/wiki/Roadmap

**Process** (per host):

```bash
# Migrate all VMs off this host
# (Use web UI or qm migrate)

# Update to latest 8.x version first
apt update && apt full-upgrade -y

# Change repositories to Proxmox 9.x
sed -i 's/bookworm/trixie/g' /etc/apt/sources.list
sed -i 's/pve-8/pve-9/g' /etc/apt/sources.list.d/pve-no-subscription.list

# Update package list
apt update

# Perform upgrade
apt full-upgrade -y

# Reboot
reboot

# Verify version
pveversion
```

**Rollback**: Restore from Proxmox backup or reinstall from ISO.

---

## Part 5: Application Upgrades

### Strategy: Blue-Green or Canary Deployments

### 5.1 PHP Application Upgrade

**Use Kubernetes Deployments** for zero-downtime upgrades.

**Example Deployment:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-app
spec:
  replicas: 6  # Run on all 4 app workers

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Create 1 extra pod during update
      maxUnavailable: 1  # Allow 1 pod to be down during update

  selector:
    matchLabels:
      app: php-app

  template:
    metadata:
      labels:
        app: php-app
        version: v2.0.0  # Update this on each release
    spec:
      nodeSelector:
        workload: application  # Run on app workers only

      containers:
      - name: php
        image: myregistry.com/php-app:v2.0.0  # Update image tag

        # Liveness probe (restart if unhealthy)
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10

        # Readiness probe (remove from service if not ready)
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

**Upgrade Process:**

```bash
# Update image tag in deployment
kubectl set image deployment/php-app php=myregistry.com/php-app:v2.0.0

# Watch rollout
kubectl rollout status deployment/php-app

# If issues, rollback immediately
kubectl rollout undo deployment/php-app
```

**Automated Canary Deployment** (with Flagger):

Install Flagger for gradual rollout:

```bash
# Install Flagger
kubectl apply -k github.com/fluxcd/flagger//kustomize/kubernetes

# Create canary resource
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: php-app
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-app

  # Start with 10% traffic, increase by 10% every 1 minute
  canaryAnalysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 10

    metrics:
    - name: request-success-rate
      threshold: 99
      interval: 1m
```

---

## Part 6: Backup Before Upgrade

**Critical**: Always backup before major upgrades.

### 6.1 Kubernetes etcd Backup

```bash
# On master node
sudo ETCDCTL_API=3 etcdctl snapshot save /root/etcd-backup-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
sudo ETCDCTL_API=3 etcdctl snapshot status /root/etcd-backup-*.db

# Copy off-server
scp /root/etcd-backup-*.db user@backup-server:/backups/k8s/
```

### 6.2 Proxmox VM Snapshots

```bash
# Snapshot all VMs before upgrade
qm snapshot 101 before-upgrade-$(date +%Y%m%d)  # k8s-master-01
qm snapshot 102 before-upgrade-$(date +%Y%m%d)  # k8s-app-worker-01
# ... repeat for all VMs

# Or snapshot all VMs in a loop
for vm in $(qm list | awk '{print $1}' | grep -v VMID); do
  qm snapshot $vm before-upgrade-$(date +%Y%m%d)
done
```

**Rollback from snapshot:**

```bash
# Rollback VM to snapshot
qm rollback 101 before-upgrade-20251026
```

### 6.3 PostgreSQL Backup

```bash
# Trigger manual backup (CNPG)
cat <<EOF | kubectl apply -f -
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: pre-upgrade-backup-$(date +%Y%m%d)
spec:
  cluster:
    name: pg-cluster-01
  method: barmanObjectStore
EOF

# Verify backup
kubectl get backups
```

---

## Part 7: Testing and Validation

### 7.1 Post-Upgrade Smoke Tests

**After any upgrade**, run these tests:

```bash
# 1. Check all nodes
kubectl get nodes
# All should show Ready

# 2. Check all pods
kubectl get pods -A | grep -v Running
# Should show no pods in non-Running state (except Completed jobs)

# 3. Check PostgreSQL cluster
kubectl get cluster pg-cluster-01
# Should show "Cluster in healthy state"

# 4. Test database connectivity
kubectl run psql-test --rm -it --restart=Never --image=postgres:16 \
  --env="PGPASSWORD=..." \
  -- psql -h pg-cluster-01-rw -U appuser -d appdb -c "SELECT version();"

# 5. Test application endpoint
curl http://<ingress-ip>/health
# Should return 200 OK

# 6. Check metrics
kubectl top nodes
kubectl top pods -n default
```

### 7.2 Rollback Procedures

**If upgrade fails:**

#### Kubernetes Rollback

```bash
# Rollback to previous Kubernetes version
# On master:
sudo apt install -y kubeadm=1.30.1-1.1 kubelet=1.30.1-1.1 kubectl=1.30.1-1.1
sudo kubeadm upgrade apply v1.30.1 --force

# On workers:
# Restore from VM snapshot (faster)
qm rollback 102 before-upgrade-20251026
```

#### PostgreSQL Rollback

```bash
# Revert image version
kubectl edit cluster pg-cluster-01
# Change imageName back to previous version

# Or restore from backup
kubectl cnpg restore pg-cluster-01 --backup pre-upgrade-backup-20251026
```

#### Application Rollback

```bash
# Kubernetes rollout undo
kubectl rollout undo deployment/php-app

# Or explicit version
kubectl rollout undo deployment/php-app --to-revision=3
```

---

## Part 8: Upgrade Schedule Template

### Quarterly Upgrade Maintenance Window

**Schedule**: First Sunday of each quarter, 2:00 AM - 6:00 AM

**Q1 Checklist** (January):
- [ ] Ubuntu security patches (all nodes)
- [ ] Kubernetes patch upgrade (1.30.1 → 1.30.2)
- [ ] Proxmox update
- [ ] PostgreSQL minor update (if available)
- [ ] Update monitoring stack (Prometheus, Grafana)

**Q2 Checklist** (April):
- [ ] Kubernetes minor upgrade (1.30.x → 1.31.0)
- [ ] Application dependencies update
- [ ] Review and update resource limits
- [ ] Test disaster recovery

**Q3 Checklist** (July):
- [ ] Ubuntu security patches
- [ ] Kubernetes patch upgrade
- [ ] PostgreSQL minor update
- [ ] Firmware updates (HP servers)

**Q4 Checklist** (October):
- [ ] Plan for next year's major upgrades
- [ ] Kubernetes minor upgrade (if available)
- [ ] Capacity planning review
- [ ] Year-end backup verification

---

## Part 9: Automation with Ansible (Advanced)

### Automate OS Updates

Create Ansible playbook for rolling node updates:

```yaml
# playbook-update-k8s-nodes.yml
---
- name: Update Kubernetes nodes (rolling)
  hosts: k8s_workers
  serial: 1  # Update one node at a time

  tasks:
    - name: Drain node
      delegate_to: localhost
      command: kubectl drain {{ inventory_hostname }} --ignore-daemonsets --delete-emptydir-data

    - name: Update packages
      apt:
        update_cache: yes
        upgrade: dist

    - name: Check if reboot required
      stat:
        path: /var/run/reboot-required
      register: reboot_required

    - name: Reboot if needed
      reboot:
        reboot_timeout: 300
      when: reboot_required.stat.exists

    - name: Uncordon node
      delegate_to: localhost
      command: kubectl uncordon {{ inventory_hostname }}

    - name: Wait for node to be Ready
      delegate_to: localhost
      command: kubectl wait --for=condition=Ready node/{{ inventory_hostname }} --timeout=300s
```

**Run playbook:**

```bash
ansible-playbook -i inventory.ini playbook-update-k8s-nodes.yml
```

---

## Summary

Your upgrade strategy is now:

✅ **Ubuntu 24.04 LTS** - 5 years of support until 2029
✅ **Rolling upgrades** - minimal downtime
✅ **Snapshots before changes** - safe rollback
✅ **Automated where possible** - Ansible, Kubernetes RollingUpdate
✅ **Quarterly maintenance schedule** - predictable updates
✅ **Tested procedures** - smoke tests after every change

**Key Principles:**
1. Always backup before major changes
2. Upgrade one component at a time
3. Test in development first (if possible)
4. Monitor closely after upgrades
5. Have rollback plan ready

**Next Step**: Set calendar reminders for quarterly maintenance windows and document your specific cluster's upgrade history.
