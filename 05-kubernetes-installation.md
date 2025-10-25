# Kubernetes Cluster Installation Guide

## Overview

This guide covers installing a production-ready Kubernetes cluster on 8 VMs (1 master + 7 workers) using **kubeadm**.

**Cluster details:**
- **Kubernetes version**: 1.30.x (latest stable)
- **Container runtime**: containerd
- **CNI plugin**: Calico (supports NetworkPolicy)
- **Master node**: 1× control plane
- **Worker nodes**: 4× app workers + 3× PostgreSQL workers

---

## Pre-Requisites

### VMs Created and Running

From Proxmox/ESXi, you should have created 8 VMs:

| VM Name | vCPU | RAM | Disk | IP | Role |
|---------|------|-----|------|-----|------|
| k8s-master-01 | 8 | 16 GB | 100 GB | 10.10.30.10 | Control plane |
| k8s-app-worker-01 | 16 | 64 GB | 600 GB | 10.10.30.21 | App workloads |
| k8s-app-worker-02 | 16 | 64 GB | 600 GB | 10.10.30.22 | App workloads |
| k8s-app-worker-03 | 16 | 64 GB | 600 GB | 10.10.30.23 | App workloads |
| k8s-app-worker-04 | 16 | 64 GB | 600 GB | 10.10.30.24 | App workloads |
| k8s-pg-worker-01 | 16 | 64 GB | 300 GB | 10.10.30.31 | PostgreSQL |
| k8s-pg-worker-02 | 16 | 64 GB | 300 GB | 10.10.30.32 | PostgreSQL |
| k8s-pg-worker-03 | 16 | 64 GB | 300 GB | 10.10.30.33 | PostgreSQL |

### OS Requirements

- **Ubuntu 22.04 LTS** (or Rocky Linux 9, CentOS Stream 9)
- **2 GB+ RAM** per node (we have plenty)
- **2+ CPUs** per node (we have plenty)
- **Full network connectivity** between all nodes
- **Unique hostname**, MAC address, product_uuid per node
- **Swap disabled** (Kubernetes requirement)

---

## Step 1: Prepare All Nodes

Run these steps on **all 8 VMs** (master + workers).

### 1.1 Set Hostnames

```bash
# On each VM, set unique hostname
sudo hostnamectl set-hostname k8s-master-01    # on master
sudo hostnamectl set-hostname k8s-app-worker-01  # on worker 1
# ... etc for all nodes
```

### 1.2 Configure /etc/hosts

Add all nodes to `/etc/hosts` on **all VMs**:

```bash
sudo tee -a /etc/hosts <<EOF
# Kubernetes cluster nodes
10.10.30.10 k8s-master-01
10.10.30.21 k8s-app-worker-01
10.10.30.22 k8s-app-worker-02
10.10.30.23 k8s-app-worker-03
10.10.30.24 k8s-app-worker-04
10.10.30.31 k8s-pg-worker-01
10.10.30.32 k8s-pg-worker-02
10.10.30.33 k8s-pg-worker-03
EOF
```

### 1.3 Disable Swap

```bash
# Disable swap immediately
sudo swapoff -a

# Disable swap permanently
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Verify swap is off
free -h  # Swap should show 0
```

### 1.4 Enable Kernel Modules

```bash
# Load required modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Verify
lsmod | grep br_netfilter
lsmod | grep overlay
```

### 1.5 Configure Kernel Parameters

```bash
# Set sysctl params required by Kubernetes
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system

# Verify
sudo sysctl net.bridge.bridge-nf-call-iptables net.ipv4.ip_forward
```

### 1.6 Install containerd

```bash
# Update packages
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release

# Add Docker's official GPG key (containerd is from Docker)
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install containerd
sudo apt-get update
sudo apt-get install -y containerd.io

# Configure containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable SystemdCgroup (required for Kubernetes)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Restart containerd
sudo systemctl restart containerd
sudo systemctl enable containerd

# Verify containerd is running
sudo systemctl status containerd
```

### 1.7 Install kubeadm, kubelet, kubectl

```bash
# Add Kubernetes apt repository
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install Kubernetes packages
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# Hold packages at current version (prevent auto-upgrade)
sudo apt-mark hold kubelet kubeadm kubectl

# Verify installation
kubeadm version
kubelet --version
kubectl version --client
```

### 1.8 Configure kubelet (for SystemdCgroup)

```bash
# Set kubelet to use systemd cgroup driver
sudo mkdir -p /etc/systemd/system/kubelet.service.d

cat <<EOF | sudo tee /etc/systemd/system/kubelet.service.d/20-cgroup.conf
[Service]
Environment="KUBELET_EXTRA_ARGS=--cgroup-driver=systemd"
EOF

# Reload systemd
sudo systemctl daemon-reload
```

---

## Step 2: Initialize Master Node

Run these steps **only on k8s-master-01**.

### 2.1 Initialize Cluster

```bash
# On k8s-master-01
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=10.10.30.10 \
  --control-plane-endpoint=10.10.30.10 \
  --upload-certs

# Options explained:
# --pod-network-cidr: Calico default (can be changed)
# --apiserver-advertise-address: Master node IP
# --control-plane-endpoint: For HA (can add more masters later)
# --upload-certs: Allows joining more control plane nodes
```

**Expected output:**

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.10.30.10:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

**⚠️ IMPORTANT**: Save the `kubeadm join` command - you'll need it to add workers!

### 2.2 Configure kubectl for Root User

```bash
# On k8s-master-01 (as root or with sudo)
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Verify
kubectl get nodes
# Should show: k8s-master-01   NotReady   control-plane   1m   v1.30.x
# NotReady is expected - no CNI plugin yet
```

### 2.3 Install Calico CNI Plugin

```bash
# Download Calico manifest
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml

kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml

# Wait for Calico pods to be ready (2-3 minutes)
watch kubectl get pods -n calico-system

# Once all pods Running, check node status
kubectl get nodes
# Should show: k8s-master-01   Ready   control-plane   5m   v1.30.x
```

**Troubleshooting**: If pods stuck in Pending/CrashLoopBackOff, check:
```bash
kubectl describe pod -n calico-system <pod-name>
journalctl -u kubelet -f
```

---

## Step 3: Join Worker Nodes

Run these steps on **all 7 worker nodes**.

### 3.1 Join Cluster

Use the `kubeadm join` command from Step 2.1 output:

```bash
# On each worker node (k8s-app-worker-01, k8s-app-worker-02, etc.)
sudo kubeadm join 10.10.30.10:6443 \
  --token <token-from-init-output> \
  --discovery-token-ca-cert-hash sha256:<hash-from-init-output>
```

**If token expired** (valid 24 hours):

```bash
# On master node, generate new token
kubeadm token create --print-join-command
# Copy and run output on workers
```

### 3.2 Verify Workers Joined

```bash
# On master node
kubectl get nodes

# Expected output:
NAME                 STATUS   ROLES           AGE   VERSION
k8s-master-01        Ready    control-plane   10m   v1.30.x
k8s-app-worker-01    Ready    <none>          5m    v1.30.x
k8s-app-worker-02    Ready    <none>          5m    v1.30.x
k8s-app-worker-03    Ready    <none>          5m    v1.30.x
k8s-app-worker-04    Ready    <none>          5m    v1.30.x
k8s-pg-worker-01     Ready    <none>          5m    v1.30.x
k8s-pg-worker-02     Ready    <none>          5m    v1.30.x
k8s-pg-worker-03     Ready    <none>          5m    v1.30.x
```

All nodes should be **Ready** status.

---

## Step 4: Label Worker Nodes

Label nodes to distinguish app workers from PostgreSQL workers.

```bash
# On master node

# Label app workers
kubectl label node k8s-app-worker-01 workload=application
kubectl label node k8s-app-worker-02 workload=application
kubectl label node k8s-app-worker-03 workload=application
kubectl label node k8s-app-worker-04 workload=application

# Label PostgreSQL workers
kubectl label node k8s-pg-worker-01 workload=database
kubectl label node k8s-pg-worker-02 workload=database
kubectl label node k8s-pg-worker-03 workload=database

# Verify labels
kubectl get nodes --show-labels | grep workload
```

**Why labels?**
- Use node selectors to ensure app pods only run on app workers
- Ensure PostgreSQL pods only run on dedicated database workers
- Prevents resource contention

---

## Step 5: Install Local Storage Provisioner

Kubernetes needs a storage provider for persistent volumes. Since we use local disks, install **local-path-provisioner**.

### 5.1 Install local-path-provisioner

```bash
# Install from Rancher's local-path-provisioner
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.28/deploy/local-path-storage.yaml

# Wait for deployment
kubectl -n local-path-storage get pods

# Set as default storage class
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Verify
kubectl get storageclass
# Should show:
# NAME                   PROVISIONER             RECLAIMPOLICY   ...
# local-path (default)   rancher.io/local-path   Delete          ...
```

### 5.2 Configure Local Storage Paths on Workers

On **each worker node**, create directory for local PVs:

```bash
# SSH to each worker
sudo mkdir -p /opt/local-path-provisioner
sudo chmod 777 /opt/local-path-provisioner
```

**For PostgreSQL workers**, create dedicated disk mount (if using second disk):

```bash
# On k8s-pg-worker-01, k8s-pg-worker-02, k8s-pg-worker-03

# Assuming second disk is /dev/sdb (check with lsblk)
sudo mkfs.ext4 /dev/sdb
sudo mkdir -p /mnt/pgdata
sudo mount /dev/sdb /mnt/pgdata

# Make permanent
echo "/dev/sdb /mnt/pgdata ext4 defaults 0 2" | sudo tee -a /etc/fstab

# Create directory for local-path-provisioner
sudo mkdir -p /mnt/pgdata/local-path-provisioner
sudo chmod 777 /mnt/pgdata/local-path-provisioner
```

**Update local-path-provisioner config** to use `/mnt/pgdata` on PostgreSQL nodes:

```bash
# On master
kubectl edit configmap local-path-config -n local-path-storage

# Add nodePathMap section:
apiVersion: v1
kind: ConfigMap
metadata:
  name: local-path-config
  namespace: local-path-storage
data:
  config.json: |-
    {
      "nodePathMap":[
        {
          "node":"k8s-pg-worker-01",
          "paths":["/mnt/pgdata/local-path-provisioner"]
        },
        {
          "node":"k8s-pg-worker-02",
          "paths":["/mnt/pgdata/local-path-provisioner"]
        },
        {
          "node":"k8s-pg-worker-03",
          "paths":["/mnt/pgdata/local-path-provisioner"]
        },
        {
          "node":"DEFAULT_PATH_FOR_NON_LISTED_NODES",
          "paths":["/opt/local-path-provisioner"]
        }
      ]
    }
```

**Restart provisioner**:

```bash
kubectl rollout restart deployment local-path-provisioner -n local-path-storage
```

---

## Step 6: Install Kubernetes Dashboard (Optional)

For web-based cluster management:

```bash
# Install Kubernetes Dashboard
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

# Create admin user
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF

# Get access token
kubectl -n kubernetes-dashboard create token admin-user

# Access dashboard via kubectl proxy
kubectl proxy

# From your workstation (via VPN):
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

---

## Step 7: Install Helm (Package Manager)

Helm simplifies installing apps like CNPG.

```bash
# On master node
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify
helm version
```

---

## Step 8: Test Cluster

### 8.1 Deploy Test Pod

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-test
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
  nodeSelector:
    workload: application
EOF

# Check pod status
kubectl get pods -o wide

# Should show Running on one of the app workers
```

### 8.2 Test Persistent Volume Claim

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 1Gi
EOF

# Check PVC
kubectl get pvc
# Should show: Bound status
```

### 8.3 Cleanup Test Resources

```bash
kubectl delete pod nginx-test
kubectl delete pvc test-pvc
```

---

## Cluster Management Commands

### View Cluster Info

```bash
# List all nodes
kubectl get nodes -o wide

# List all pods (all namespaces)
kubectl get pods -A

# Check cluster health
kubectl get componentstatuses

# View cluster resources
kubectl top nodes  # Requires metrics-server
```

### Install Metrics Server (for resource monitoring)

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# For self-signed certs (in testing), add --kubelet-insecure-tls
kubectl edit deployment metrics-server -n kube-system
# Add: --kubelet-insecure-tls to args

# Verify
kubectl top nodes
kubectl top pods -A
```

### Drain Node (for maintenance)

```bash
# Safely evict all pods from a node
kubectl drain k8s-app-worker-01 --ignore-daemonsets --delete-emptydir-data

# Do maintenance...

# Re-enable scheduling
kubectl uncordon k8s-app-worker-01
```

### Remove Node from Cluster

```bash
# On master
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
kubectl delete node <node-name>

# On the node being removed
sudo kubeadm reset
```

---

## Backup and Restore

### Backup etcd (Cluster State)

```bash
# On master node
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /root/etcd-backup-$(date +%Y%m%d-%H%M%S).db

# Copy backup off-server
scp /root/etcd-backup-*.db user@backup-server:/backups/
```

### Restore etcd

```bash
# Stop API server
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/

# Restore snapshot
sudo ETCDCTL_API=3 etcdctl snapshot restore /root/etcd-backup-<timestamp>.db \
  --data-dir=/var/lib/etcd-restore

# Update etcd manifest to use new data dir
sudo sed -i 's|/var/lib/etcd|/var/lib/etcd-restore|g' /etc/kubernetes/manifests/etcd.yaml

# Restart API server
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
```

---

## Troubleshooting

### Pods not starting

```bash
# Check pod events
kubectl describe pod <pod-name>

# Check logs
kubectl logs <pod-name>

# Check node resources
kubectl top nodes
kubectl describe node <node-name>
```

### Network issues

```bash
# Check Calico pods
kubectl get pods -n calico-system

# Check CNI plugin
sudo ls /opt/cni/bin/
sudo ls /etc/cni/net.d/

# Test pod-to-pod networking
kubectl run test-pod-1 --image=busybox --restart=Never -- sleep 3600
kubectl run test-pod-2 --image=busybox --restart=Never -- sleep 3600
kubectl exec test-pod-1 -- ping <test-pod-2-ip>
```

### kubelet not starting

```bash
# Check kubelet logs
sudo journalctl -u kubelet -f

# Check systemd status
sudo systemctl status kubelet

# Restart kubelet
sudo systemctl restart kubelet
```

---

## Next Steps

1. **Install CNPG operator** for PostgreSQL (see 06-postgresql-cnpg-setup.md)
2. **Deploy sample PHP application**
3. **Set up ingress controller** (Nginx or Traefik)
4. **Configure monitoring** (Prometheus + Grafana)
5. **Set up backups** (Velero for Kubernetes, Barman for PostgreSQL)

See next document for PostgreSQL with CNPG deployment.
