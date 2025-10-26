# Storage Setup with TopoLVM

## Overview

This guide covers setting up **TopoLVM** as the storage provider for Kubernetes, replacing the basic local-path-provisioner with a production-grade LVM-based CSI driver.

**Why TopoLVM?**
- ✅ **Dynamic provisioning** - automatically creates volumes on demand
- ✅ **Thin provisioning** - efficient disk space usage
- ✅ **Volume snapshots** - instant backups for PostgreSQL
- ✅ **Capacity-aware scheduling** - schedules pods on nodes with available storage
- ✅ **Performance** - native LVM performance on NVMe
- ✅ **Production-ready** - used by Cybozu and many others

---

## Architecture

### How TopoLVM Works

```
Application Pod → PVC → TopoLVM CSI Driver → LVM → NVMe Disk

Each Node:
├── NVMe SSD (physical disk)
│   └── LVM Volume Group
│       └── Thin Pool
│           ├── Logical Volume 1 (PV for PostgreSQL)
│           ├── Logical Volume 2 (PV for app data)
│           └── Logical Volume N...
```

**Benefits over local-path-provisioner:**
- Snapshots without copying data (LVM thin provisioning)
- Capacity tracking (Kubernetes knows how much space is available)
- Better performance (no filesystem overhead, direct LVM access)

---

## Part 1: Prepare LVM on All Nodes

### 1.1 Identify Available Disks

On **each Kubernetes node** (all 8 VMs), identify the data disk:

```bash
# SSH to each node
ssh root@k8s-app-worker-01

# List all disks
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT

# Expected output:
# NAME   SIZE  TYPE MOUNTPOINT
# sda    100G  disk              # Root disk (OS)
# ├─sda1  99G  part /            # OS partition
# └─sda2   1G  part [SWAP]       # Swap (if any)
# sdb    600G  disk              # Data disk (for storage) ← Use this
```

**For your setup:**
- **Master node**: Only 100 GB disk (no extra data disk needed)
- **App workers**: `sdb` = 400-500 GB data disk
- **PostgreSQL workers**: `sdb` = 200-300 GB data disk

### 1.2 Create LVM Volume Groups

On **each worker node** (not master):

```bash
# Install LVM tools (if not already installed)
sudo apt update
sudo apt install -y lvm2

# Create Physical Volume
sudo pvcreate /dev/sdb

# Create Volume Group named "topolvm-vg"
sudo vgcreate topolvm-vg /dev/sdb

# Verify
sudo vgs
# Should show:
#   VG         #PV #LV #SN Attr   VSize   VFree
#   topolvm-vg   1   0   0 wz--n- 599.99g 599.99g
```

**Important**: TopoLVM will manage thin pools automatically, so **don't create thin pool manually**.

### 1.3 Repeat for All Workers

Run the above on:
- k8s-app-worker-01 through k8s-app-worker-04
- k8s-pg-worker-01 through k8s-pg-worker-03

**Skip the master node** (k8s-master-01) unless you want to store data there.

---

## Part 2: Install TopoLVM Operator

### 2.1 Install via Helm

```bash
# On your workstation or master node (with kubectl access)

# Add TopoLVM Helm repository
helm repo add topolvm https://topolvm.github.io/topolvm
helm repo update

# Create namespace
kubectl create namespace topolvm-system

# Install TopoLVM
helm install topolvm topolvm/topolvm \
  --namespace topolvm-system \
  --set controller.replicaCount=1 \
  --set snapshot.enabled=true

# Wait for pods to be ready (2-3 minutes)
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=topolvm -n topolvm-system --timeout=300s

# Verify installation
kubectl get pods -n topolvm-system
```

**Expected output:**
```
NAME                                  READY   STATUS    RESTARTS   AGE
topolvm-controller-xxxxxxxxxx-xxxxx   4/4     Running   0          2m
topolvm-node-xxxxx                    4/4     Running   0          2m  # One per worker node
topolvm-node-xxxxx                    4/4     Running   0          2m
topolvm-node-xxxxx                    4/4     Running   0          2m
...
```

### 2.2 Verify CSI Driver Registered

```bash
# Check CSI driver
kubectl get csidrivers

# Should show:
# NAME                 ATTACHREQUIRED   PODINFOONMOUNT   MODES        AGE
# topolvm.io           true             false            Persistent   2m
```

---

## Part 3: Create Storage Classes

### 3.1 Default Storage Class (Thin Provisioned)

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: topolvm-provisioner
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: topolvm.io
parameters:
  "csi.storage.k8s.io/fstype": "xfs"
  "topolvm.io/device-class": ""
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Delete
EOF
```

**Parameters explained:**
- `fstype: xfs` - XFS filesystem (good for databases, supports online resize)
- `device-class: ""` - Use default volume group (topolvm-vg)
- `volumeBindingMode: WaitForFirstConsumer` - Create volume only when pod is scheduled
- `allowVolumeExpansion: true` - Allow growing volumes later
- `reclaimPolicy: Delete` - Delete LV when PVC is deleted

### 3.2 Storage Class for PostgreSQL (with snapshots)

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: topolvm-postgres
provisioner: topolvm.io
parameters:
  "csi.storage.k8s.io/fstype": "xfs"
  "topolvm.io/device-class": ""
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Retain  # Keep data even if PVC deleted (safer for databases)
EOF
```

**Difference from default:**
- `reclaimPolicy: Retain` - Keeps LVM volume even if PVC is deleted (safety for databases)

### 3.3 Verify Storage Classes

```bash
kubectl get storageclass

# Should show:
# NAME                          PROVISIONER     RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
# topolvm-provisioner (default) topolvm.io      Delete          WaitForFirstConsumer   true                   1m
# topolvm-postgres              topolvm.io      Retain          WaitForFirstConsumer   true                   1m
```

---

## Part 4: Configure Node Labels (Optional)

Label nodes to separate storage tiers (e.g., app workers vs PostgreSQL workers).

```bash
# Already done in earlier guide, but verify:
kubectl get nodes --show-labels | grep workload

# Should show:
# k8s-app-worker-01    workload=application
# k8s-app-worker-02    workload=application
# k8s-app-worker-03    workload=application
# k8s-app-worker-04    workload=application
# k8s-pg-worker-01     workload=database
# k8s-pg-worker-02     workload=database
# k8s-pg-worker-03     workload=database
```

---

## Part 5: Test TopoLVM

### 5.1 Create Test PVC

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-topolvm-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: topolvm-provisioner
EOF
```

### 5.2 Create Test Pod Using PVC

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-topolvm-pod
spec:
  containers:
  - name: test
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: test-topolvm-pvc
EOF
```

### 5.3 Verify Volume Created

```bash
# Check PVC status
kubectl get pvc test-topolvm-pvc
# Should show: Bound

# Check PV created
kubectl get pv
# Should show a PV with size 10Gi

# Check pod is running
kubectl get pod test-topolvm-pod
# Should show: Running

# Exec into pod and write data
kubectl exec -it test-topolvm-pod -- sh
df -h /data
# Should show ~10G mounted on /data

echo "TopoLVM works!" > /data/test.txt
cat /data/test.txt
exit
```

### 5.4 Verify LVM Volume on Node

```bash
# Find which node the pod is on
kubectl get pod test-topolvm-pod -o wide
# Note the NODE column

# SSH to that node
ssh root@<node-ip>

# Check LVM volumes
sudo lvs
# Should show a new Logical Volume created by TopoLVM

# Example output:
#   LV                                      VG         Attr       LSize
#   topolvm--<uuid>                         topolvm-vg -wi-ao---- 10.00g
```

### 5.5 Cleanup Test Resources

```bash
kubectl delete pod test-topolvm-pod
kubectl delete pvc test-topolvm-pvc
```

---

## Part 6: Update PostgreSQL Cluster to Use TopoLVM

Now that TopoLVM is installed, update your PostgreSQL cluster to use it.

### 6.1 Create New PostgreSQL Cluster with TopoLVM

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: pg-cluster-01
  namespace: default
spec:
  instances: 3
  imageName: ghcr.io/cloudnative-pg/postgresql:16.3

  # Storage using TopoLVM
  storage:
    size: 100Gi
    storageClass: topolvm-postgres  # Use TopoLVM storage class

  # Resource limits
  resources:
    requests:
      memory: "32Gi"
      cpu: "8"
    limits:
      memory: "48Gi"
      cpu: "12"

  # Node affinity - PostgreSQL workers only
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: workload
            operator: In
            values:
            - database

    # Pod anti-affinity - different nodes
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: cnpg.io/cluster
            operator: In
            values:
            - pg-cluster-01
        topologyKey: kubernetes.io/hostname

  # PostgreSQL configuration
  postgresql:
    parameters:
      shared_buffers: "12GB"
      effective_cache_size: "36GB"
      maintenance_work_mem: "2GB"
      work_mem: "128MB"
      max_connections: "500"
      random_page_cost: "1.1"
      effective_io_concurrency: "200"

  # HA settings
  minSyncReplicas: 1
  maxSyncReplicas: 2

  # Monitoring
  monitoring:
    enablePodMonitor: true

  # Bootstrap
  bootstrap:
    initdb:
      database: appdb
      owner: appuser
      secret:
        name: pg-cluster-01-app-user
EOF
```

### 6.2 Verify PostgreSQL Using TopoLVM

```bash
# Check cluster status
kubectl get cluster pg-cluster-01

# Check PVCs
kubectl get pvc -l cnpg.io/cluster=pg-cluster-01

# Should show 3 PVCs, all Bound, using topolvm-postgres storage class
```

---

## Part 7: Volume Snapshots (for Backups)

TopoLVM supports volume snapshots via VolumeSnapshot API.

### 7.1 Install Snapshot Controller

```bash
# Install snapshot CRDs
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml

# Install snapshot controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/deploy/kubernetes/snapshot-controller/rbac-snapshot-controller.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/deploy/kubernetes/snapshot-controller/setup-snapshot-controller.yaml

# Verify
kubectl get pods -n kube-system | grep snapshot
```

### 7.2 Create VolumeSnapshotClass

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: topolvm-snapshot
driver: topolvm.io
deletionPolicy: Delete
EOF
```

### 7.3 Take Snapshot of PostgreSQL Volume

```bash
# Find PVC name for PostgreSQL primary
kubectl get pvc -l cnpg.io/cluster=pg-cluster-01

# Create snapshot
cat <<EOF | kubectl apply -f -
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: pg-snapshot-$(date +%Y%m%d)
spec:
  volumeSnapshotClassName: topolvm-snapshot
  source:
    persistentVolumeClaimName: pg-cluster-01-1  # Adjust to actual PVC name
EOF

# Check snapshot status
kubectl get volumesnapshot
# Should show ReadyToUse: true
```

### 7.4 Restore from Snapshot

```yaml
# Create new PVC from snapshot
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pg-restored-from-snapshot
spec:
  storageClassName: topolvm-postgres
  dataSource:
    name: pg-snapshot-20251026
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
EOF
```

---

## Part 8: Monitoring and Capacity Management

### 8.1 Check Available Capacity per Node

TopoLVM exposes metrics for available capacity on each node.

```bash
# Install metrics-server if not already
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Check node capacity (via TopoLVM metrics)
kubectl get nodes -o custom-columns=\
NAME:.metadata.name,\
CAPACITY:.status.capacity.topolvm\\.io/capacity

# Example output:
# NAME               CAPACITY
# k8s-app-worker-01  599Gi
# k8s-app-worker-02  599Gi
# k8s-pg-worker-01   299Gi
```

### 8.2 Monitor LVM Thin Pool Usage

On each node:

```bash
ssh root@k8s-pg-worker-01

# Check thin pool usage
sudo lvs -o lv_name,data_percent,metadata_percent,lv_size

# Example output:
#   LV                    Data%  Meta%  LSize
#   topolvm--pool         45.23  12.34  299.00g
```

**Alert thresholds:**
- **Data% > 80%**: Consider expanding VG or deleting old snapshots
- **Metadata% > 80%**: Increase metadata size (rare issue)

### 8.3 Prometheus Metrics (if using Prometheus)

TopoLVM exports Prometheus metrics:

```bash
# Check TopoLVM metrics endpoint
kubectl port-forward -n topolvm-system svc/topolvm-controller 8080:8080

# In another terminal:
curl http://localhost:8080/metrics | grep topolvm

# Key metrics:
# topolvm_volumegroup_available_bytes - Available capacity
# topolvm_volumegroup_size_bytes - Total capacity
# topolvm_thinpool_data_percent - Thin pool data usage
```

---

## Part 9: Troubleshooting

### Issue: PVC stuck in Pending

```bash
# Check PVC events
kubectl describe pvc <pvc-name>

# Common causes:
# 1. No nodes with enough capacity
# 2. No nodes match nodeSelector/affinity
# 3. TopoLVM node pod not running on target node

# Check TopoLVM logs
kubectl logs -n topolvm-system -l app.kubernetes.io/component=node
```

### Issue: Volume not expanding

```bash
# Check if StorageClass allows expansion
kubectl get sc topolvm-postgres -o yaml | grep allowVolumeExpansion
# Should show: true

# Expand PVC
kubectl patch pvc <pvc-name> -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'

# Check expansion status
kubectl describe pvc <pvc-name>
# Look for "FileSystemResizeSuccessful" condition
```

### Issue: Snapshot fails

```bash
# Check VolumeSnapshot status
kubectl describe volumesnapshot <snapshot-name>

# Check snapshot controller logs
kubectl logs -n kube-system -l app=snapshot-controller

# Verify LVM thin pool has space for snapshots
ssh root@<node-ip>
sudo lvs -o lv_name,data_percent,lv_size
```

---

## Part 10: Migration from local-path-provisioner

If you already deployed with local-path-provisioner, migrate to TopoLVM:

### 10.1 Backup Data

```bash
# For each PVC, backup data
kubectl exec -it <pod-name> -- tar czf /tmp/backup.tar.gz /data

# Copy backup out
kubectl cp <pod-name>:/tmp/backup.tar.gz ./backup.tar.gz
```

### 10.2 Create New PVC with TopoLVM

```yaml
# Delete old pod (but keep PVC for now)
kubectl delete pod <pod-name>

# Create new PVC with TopoLVM
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: <new-pvc-name>
spec:
  storageClassName: topolvm-provisioner
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```

### 10.3 Restore Data and Switch

```bash
# Create pod with new PVC
# Copy data back
kubectl cp ./backup.tar.gz <new-pod-name>:/tmp/backup.tar.gz
kubectl exec -it <new-pod-name> -- tar xzf /tmp/backup.tar.gz -C /

# Verify data
# Delete old PVC
kubectl delete pvc <old-pvc-name>
```

---

## Summary

You now have TopoLVM configured with:

✅ **LVM-based dynamic provisioning** on all worker nodes
✅ **Thin provisioning** for efficient storage usage
✅ **Volume snapshots** for instant backups
✅ **Capacity-aware scheduling** - pods go to nodes with space
✅ **Storage classes** for different workloads
✅ **PostgreSQL integration** - CNPG using TopoLVM storage

**Benefits over local-path-provisioner:**
- 📸 Instant snapshots (LVM thin provisioning)
- 📊 Capacity tracking (Kubernetes knows available space)
- ⚡ Better performance (direct LVM access)
- 🔄 Volume expansion (grow volumes online)
- 🎯 Smarter scheduling (pods avoid full nodes)

**Next steps:**
- Configure snapshot schedules for PostgreSQL
- Set up Prometheus alerts for capacity
- Test volume expansion and snapshots
