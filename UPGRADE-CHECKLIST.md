# Upgrade Checklist Template

Use this checklist during planned maintenance windows. Print or keep this open during upgrades.

---

## Pre-Upgrade Phase

**Date**: _______________
**Maintenance Window**: _______________ to _______________
**Components to Upgrade**: ☐ OS  ☐ Kubernetes  ☐ PostgreSQL  ☐ Proxmox  ☐ Applications

### Preparation

- [ ] Read release notes for all components being upgraded
- [ ] Schedule maintenance window (notify users)
- [ ] Verify backup retention (check backups are recent and valid)
- [ ] Document current versions:
  ```
  Ubuntu version: _____________
  Kubernetes version: _____________
  PostgreSQL version: _____________
  Proxmox version: _____________
  Application version: _____________
  ```

### Backup Everything

- [ ] **etcd backup** (Kubernetes cluster state)
  ```bash
  sudo ETCDCTL_API=3 etcdctl snapshot save /root/etcd-backup-$(date +%Y%m%d-%H%M%S).db \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key
  ```
  Backup location: _________________________

- [ ] **Proxmox VM snapshots** (all VMs)
  ```bash
  # Snapshot all VMs
  for vm in $(qm list | awk '{print $1}' | grep -v VMID); do
    qm snapshot $vm pre-upgrade-$(date +%Y%m%d-%H%M)
  done
  ```
  Snapshot name: _________________________

- [ ] **PostgreSQL backup** (CNPG)
  ```bash
  cat <<EOF | kubectl apply -f -
  apiVersion: postgresql.cnpg.io/v1
  kind: Backup
  metadata:
    name: pre-upgrade-$(date +%Y%m%d)
  spec:
    cluster:
      name: pg-cluster-01
  EOF
  ```
  Backup name: _________________________

- [ ] **Configuration backup**
  ```bash
  # Backup Kubernetes manifests
  kubectl get all --all-namespaces -o yaml > all-resources-$(date +%Y%m%d).yaml

  # Backup Proxmox config
  tar czf pve-config-$(date +%Y%m%d).tar.gz /etc/pve
  ```
  Backup location: _________________________

- [ ] Copy all backups to off-site location
  Off-site location: _________________________

### Pre-Upgrade Health Check

- [ ] **Cluster health**
  ```bash
  kubectl get nodes
  # All nodes should be "Ready"

  kubectl get pods -A | grep -v Running | grep -v Completed
  # Should be empty (no unhealthy pods)
  ```

- [ ] **PostgreSQL health**
  ```bash
  kubectl get cluster pg-cluster-01
  # Should show "Cluster in healthy state"

  kubectl exec -it pg-cluster-01-1 -- psql -U postgres -c "SELECT * FROM pg_stat_replication;"
  # Should show active replication
  ```

- [ ] **Application health**
  ```bash
  curl http://<ingress-ip>/health
  # Should return 200 OK
  ```

- [ ] **Proxmox cluster health** (if clustered)
  ```bash
  pvecm status
  # All nodes should show as online
  ```

---

## Upgrade Phase

### Option A: Ubuntu OS Upgrade

**Target version**: Ubuntu _____________

#### Worker Nodes (one at a time)

**Node**: k8s-app-worker-01

- [ ] Drain node
  ```bash
  kubectl drain k8s-app-worker-01 --ignore-daemonsets --delete-emptydir-data
  ```

- [ ] SSH to node and run upgrade
  ```bash
  ssh root@k8s-app-worker-01
  sudo apt update && sudo apt upgrade -y && sudo apt dist-upgrade -y
  ```

- [ ] If minor update: check for reboot requirement
  ```bash
  [ -f /var/run/reboot-required ] && echo "Reboot required"
  ```

- [ ] If major version upgrade:
  ```bash
  sudo do-release-upgrade
  # Follow prompts
  ```

- [ ] Reboot node
  ```bash
  sudo reboot
  ```

- [ ] Wait for node to come back online (2-5 minutes)
  ```bash
  ping 10.10.30.21  # Wait for response
  ```

- [ ] Uncordon node
  ```bash
  kubectl uncordon k8s-app-worker-01
  ```

- [ ] Verify node is Ready
  ```bash
  kubectl get node k8s-app-worker-01
  # Should show "Ready"
  ```

- [ ] **Repeat for remaining workers**: worker-02, worker-03, worker-04, pg-worker-01, pg-worker-02, pg-worker-03

#### Master Node (last)

- [ ] Follow same process for k8s-master-01
  ⚠️ **Note**: During master upgrade, kubectl will be temporarily unavailable

---

### Option B: Kubernetes Upgrade

**Target version**: Kubernetes _____________

#### Master Node

- [ ] Upgrade kubeadm
  ```bash
  sudo apt-mark unhold kubeadm
  sudo apt install -y kubeadm=1.XX.X-1.1  # Replace with target version
  sudo apt-mark hold kubeadm
  ```

- [ ] Verify kubeadm version
  ```bash
  kubeadm version
  ```

- [ ] Plan upgrade
  ```bash
  sudo kubeadm upgrade plan
  # Review output for warnings
  ```

- [ ] Apply upgrade
  ```bash
  sudo kubeadm upgrade apply v1.XX.X
  ```
  Time started: _____________
  Time completed: _____________

- [ ] Drain master
  ```bash
  kubectl drain k8s-master-01 --ignore-daemonsets
  ```

- [ ] Upgrade kubelet and kubectl
  ```bash
  sudo apt-mark unhold kubelet kubectl
  sudo apt install -y kubelet=1.XX.X-1.1 kubectl=1.XX.X-1.1
  sudo apt-mark hold kubelet kubectl
  sudo systemctl daemon-reload
  sudo systemctl restart kubelet
  ```

- [ ] Uncordon master
  ```bash
  kubectl uncordon k8s-master-01
  ```

- [ ] Verify master version
  ```bash
  kubectl version --short
  kubectl get nodes
  ```

#### Worker Nodes (one at a time)

**Node**: k8s-app-worker-01

- [ ] Drain node
  ```bash
  kubectl drain k8s-app-worker-01 --ignore-daemonsets --delete-emptydir-data
  ```

- [ ] SSH to node and upgrade kubeadm
  ```bash
  ssh root@k8s-app-worker-01
  sudo apt-mark unhold kubeadm
  sudo apt install -y kubeadm=1.XX.X-1.1
  sudo apt-mark hold kubeadm
  ```

- [ ] Upgrade node config
  ```bash
  sudo kubeadm upgrade node
  ```

- [ ] Upgrade kubelet and kubectl
  ```bash
  sudo apt-mark unhold kubelet kubectl
  sudo apt install -y kubelet=1.XX.X-1.1 kubectl=1.XX.X-1.1
  sudo apt-mark hold kubelet kubectl
  sudo systemctl daemon-reload
  sudo systemctl restart kubelet
  exit
  ```

- [ ] Uncordon node
  ```bash
  kubectl uncordon k8s-app-worker-01
  ```

- [ ] Verify node version
  ```bash
  kubectl get node k8s-app-worker-01 -o wide
  ```

- [ ] **Repeat for remaining workers**: worker-02, worker-03, worker-04, pg-worker-01, pg-worker-02, pg-worker-03

---

### Option C: PostgreSQL Upgrade

**Target version**: PostgreSQL _____________

#### Minor Version (e.g., 16.3 → 16.4)

- [ ] Edit cluster manifest
  ```bash
  kubectl edit cluster pg-cluster-01

  # Update spec.imageName to new version
  # Save and exit
  ```

- [ ] Monitor rolling upgrade
  ```bash
  kubectl get cluster pg-cluster-01 -w
  # Wait for "Cluster in healthy state"
  ```
  Time started: _____________
  Time completed: _____________

- [ ] Verify all pods updated
  ```bash
  kubectl get pods -l cnpg.io/cluster=pg-cluster-01 -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
  ```

#### Major Version (e.g., 16.x → 17.0)

⚠️ **Use blue-green deployment** - see 07-upgrade-strategy.md for detailed procedure

- [ ] Create new cluster with PostgreSQL 17
- [ ] Wait for initial sync
- [ ] Test new cluster thoroughly
- [ ] Switch application traffic to new cluster
- [ ] Monitor for 24-48 hours
- [ ] Delete old cluster (if stable)

---

### Option D: Proxmox VE Upgrade

**Target version**: Proxmox _____________

**Host**: k8s-host-03 (start with least critical)

- [ ] Live migrate all VMs off this host
  ```bash
  # In Proxmox web UI or:
  qm migrate 101 k8s-host-02
  qm migrate 102 k8s-host-02
  # ... repeat for all VMs on this host
  ```

- [ ] Verify no VMs running on this host
  ```bash
  qm list
  ```

- [ ] SSH to host and run upgrade
  ```bash
  ssh root@10.10.20.13
  apt update
  apt full-upgrade -y
  ```

- [ ] If kernel updated, reboot
  ```bash
  reboot
  ```

- [ ] After reboot, verify cluster status
  ```bash
  pvecm status
  pveversion
  ```

- [ ] Migrate some VMs back to balanced load

- [ ] **Repeat for host-02, then host-01**

---

## Post-Upgrade Phase

### Verification Tests

- [ ] **All nodes healthy**
  ```bash
  kubectl get nodes
  # All should be "Ready" with new version
  ```

- [ ] **All pods running**
  ```bash
  kubectl get pods -A | grep -v Running | grep -v Completed
  # Should be empty
  ```

- [ ] **PostgreSQL cluster healthy**
  ```bash
  kubectl get cluster pg-cluster-01
  # Should show "Cluster in healthy state"

  kubectl exec -it pg-cluster-01-1 -- psql -U postgres -c "SELECT version();"
  # Should show new version
  ```

- [ ] **Database connectivity test**
  ```bash
  kubectl run psql-test --rm -it --restart=Never --image=postgres:16 \
    --env="PGPASSWORD=$(kubectl get secret pg-cluster-01-superuser -o jsonpath='{.data.password}' | base64 -d)" \
    -- psql -h pg-cluster-01-rw -U postgres -c "SELECT NOW();"
  # Should return current timestamp
  ```

- [ ] **Application health check**
  ```bash
  curl http://<ingress-ip>/health
  # Should return 200 OK

  # Or test actual application endpoint
  curl http://<ingress-ip>/
  ```

- [ ] **Resource usage check**
  ```bash
  kubectl top nodes
  kubectl top pods -n default
  # Verify normal resource usage
  ```

- [ ] **Check for errors in logs**
  ```bash
  # Kubelet logs
  kubectl get events --sort-by='.lastTimestamp' | tail -20

  # Application logs
  kubectl logs -l app=php-app --tail=50

  # PostgreSQL logs
  kubectl logs -l cnpg.io/cluster=pg-cluster-01 --tail=50
  ```

### Monitoring

- [ ] Monitor cluster for **24 hours** post-upgrade
- [ ] Check Grafana dashboards (if installed)
- [ ] Review Prometheus alerts (if any)
- [ ] Monitor application error rates
- [ ] Check PostgreSQL replication lag

### Cleanup

- [ ] Remove old VM snapshots (after 7 days of stability)
  ```bash
  qm delsnapshot 101 pre-upgrade-20251026
  ```

- [ ] Remove old etcd backups (keep last 3)
  ```bash
  ls -lh /root/etcd-backup-*
  # Delete old backups
  ```

- [ ] Update documentation with new versions
  ```
  Ubuntu: _____________
  Kubernetes: _____________
  PostgreSQL: _____________
  Proxmox: _____________
  ```

---

## Rollback Procedure (if needed)

### If Upgrade Fails - Immediate Rollback

#### Kubernetes Rollback

- [ ] Master node: Restore from VM snapshot
  ```bash
  qm rollback 101 pre-upgrade-20251026
  ```

- [ ] Worker nodes: Restore from snapshots or downgrade packages
  ```bash
  sudo apt install -y kubeadm=1.30.1-1.1 kubelet=1.30.1-1.1 kubectl=1.30.1-1.1
  sudo kubeadm upgrade apply v1.30.1 --force
  ```

#### PostgreSQL Rollback

- [ ] Revert cluster image
  ```bash
  kubectl edit cluster pg-cluster-01
  # Change imageName back to old version
  ```

#### Application Rollback

- [ ] Rollback deployment
  ```bash
  kubectl rollout undo deployment/php-app
  ```

---

## Sign-Off

**Upgrade completed by**: _________________________
**Date/Time completed**: _________________________
**Final cluster status**: ☐ Healthy  ☐ Issues (describe below)

**Issues encountered:**
_________________________________________________________________
_________________________________________________________________

**Actions taken:**
_________________________________________________________________
_________________________________________________________________

**Next scheduled upgrade**: _________________________

---

## Notes

Use this space for upgrade-specific notes, observations, or lessons learned:

_________________________________________________________________
_________________________________________________________________
_________________________________________________________________
_________________________________________________________________
