# Cilium Networking Setup

## Overview

This guide covers installing **Cilium** as the CNI (Container Network Interface) plugin for Kubernetes, replacing Calico with a modern eBPF-based networking solution.

**Why Cilium?**
- ✅ **eBPF-native** - kernel-level networking, no iptables overhead
- ✅ **Better performance** - lower latency, higher throughput
- ✅ **Hubble observability** - built-in network flow visualization
- ✅ **Service mesh** - mTLS, load balancing without Istio
- ✅ **kube-proxy replacement** - faster service load balancing
- ✅ **Network policies** - same as Calico but more powerful

**Perfect for your setup:**
- Low-latency PostgreSQL connections from PHP apps
- Network observability (debug connection issues)
- Encryption for database traffic
- Better performance on bare metal

---

## Part 1: Prerequisites

### 1.1 Kernel Requirements

Cilium requires Linux kernel **4.19.57+** (Ubuntu 24.04 has 6.8+, so we're good).

Verify on each node:

```bash
uname -r
# Should show: 6.8.0-xx-generic (Ubuntu 24.04)
```

### 1.2 Install Cilium CLI (on your workstation/master)

```bash
# Download Cilium CLI
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64

curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

# Verify installation
cilium version --client
```

---

## Part 2: Install Cilium (Instead of Calico)

### Option A: Fresh Kubernetes Cluster (Recommended Path)

If you're setting up Kubernetes from scratch, install Cilium **during cluster init**.

#### 2.1 Initialize Kubernetes WITHOUT Default CNI

On **master node**:

```bash
# Initialize cluster, skip installing kube-proxy
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=10.10.30.10 \
  --control-plane-endpoint=10.10.30.10 \
  --skip-phases=addon/kube-proxy \
  --upload-certs

# Configure kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**Note**: `--skip-phases=addon/kube-proxy` - Cilium will replace kube-proxy for better performance.

#### 2.2 Install Cilium

```bash
# Install Cilium with Hubble observability
cilium install \
  --version 1.16.0 \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=10.10.30.10 \
  --set k8sServicePort=6443 \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true

# Wait for Cilium to be ready (2-3 minutes)
cilium status --wait

# Expected output:
#     /¯¯\
#  /¯¯\__/¯¯\    Cilium:             OK
#  \__/¯¯\__/    Operator:           OK
#  /¯¯\__/¯¯\    Envoy DaemonSet:    OK
#  \__/¯¯\__/    Hubble Relay:       OK
#     \__/       ClusterMesh:        disabled
```

#### 2.3 Join Worker Nodes

Now join worker nodes as usual:

```bash
# On each worker node, use the join command from kubeadm init output
sudo kubeadm join 10.10.30.10:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

Cilium will automatically configure networking on each node.

---

### Option B: Migrate from Calico to Cilium (Existing Cluster)

⚠️ **Warning**: This causes temporary network disruption. Do during maintenance window.

#### 2.1 Backup Current State

```bash
# Backup etcd
sudo ETCDCTL_API=3 etcdctl snapshot save /root/etcd-backup-before-cilium.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Snapshot VMs
# (via Proxmox UI or: qm snapshot <vmid> before-cilium)
```

#### 2.2 Remove Calico

```bash
# Delete Calico resources
kubectl delete -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml
kubectl delete -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml

# Wait for Calico pods to terminate
kubectl get pods -n calico-system
# Should be empty

# On each node, remove Calico CNI config
sudo rm -rf /etc/cni/net.d/10-calico.conflist
sudo rm -rf /etc/cni/net.d/calico-kubeconfig
```

#### 2.3 Install Cilium

```bash
cilium install --version 1.16.0 \
  --set kubeProxyReplacement=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true

# Restart all pods to get new networking
kubectl rollout restart deployment -A
kubectl rollout restart daemonset -A
```

#### 2.4 Verify Migration

```bash
# Check connectivity
cilium connectivity test

# If test passes, migration successful!
```

---

## Part 3: Enable Hubble Observability

Hubble provides network flow visualization.

### 3.1 Install Hubble CLI

```bash
# Download Hubble CLI
HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)

curl -L --fail --remote-name-all https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-amd64.tar.gz{,.sha256sum}

sha256sum --check hubble-linux-amd64.tar.gz.sha256sum
sudo tar xzvfC hubble-linux-amd64.tar.gz /usr/local/bin
rm hubble-linux-amd64.tar.gz{,.sha256sum}

# Verify
hubble version
```

### 3.2 Access Hubble Relay

```bash
# Port-forward Hubble Relay
cilium hubble port-forward &

# In another terminal, observe flows
hubble observe

# Example output:
# Oct 26 08:00:00.123: 10.244.1.5:54321 -> 10.244.2.3:5432 to-endpoint FORWARDED (TCP)
#   php-app-xxx => pg-cluster-01-rw
```

### 3.3 Access Hubble UI (Web Interface)

```bash
# Port-forward Hubble UI
cilium hubble ui

# Opens browser to http://localhost:12000
# Visualize network flows in real-time!
```

**Hubble UI Features:**
- Service map (which pods talk to which)
- Flow logs (live network traffic)
- Latency metrics
- DNS queries

---

## Part 4: Configure Network Policies

Cilium supports Kubernetes NetworkPolicy + extended CiliumNetworkPolicy.

### 4.1 Basic PostgreSQL Access Policy

Restrict PostgreSQL access to only app workers:

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: postgres-access-policy
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      cnpg.io/cluster: pg-cluster-01  # PostgreSQL pods

  ingress:
  - fromEndpoints:
    - matchLabels:
        app: php-app  # Only PHP app pods
    toPorts:
    - ports:
      - port: "5432"
        protocol: TCP

  # Allow PostgreSQL replication between replicas
  - fromEndpoints:
    - matchLabels:
        cnpg.io/cluster: pg-cluster-01
    toPorts:
    - ports:
      - port: "5432"
        protocol: TCP

  egress:
  - {} # Allow all outbound (for updates, backups, etc.)
EOF
```

### 4.2 Deny All by Default (Best Practice)

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: default-deny-all
  namespace: default
spec:
  endpointSelector: {}
  ingress:
  - {} # Empty = deny all
  egress:
  - toEntities:
    - kube-dns  # Allow DNS
  - toEndpoints:
    - matchLabels:
        k8s:io.kubernetes.pod.namespace: kube-system
EOF
```

Then whitelist specific traffic (like PostgreSQL above).

---

## Part 5: Enable Encryption (mTLS for Database Traffic)

Encrypt all pod-to-pod traffic, especially important for PostgreSQL connections.

### 5.1 Enable WireGuard Encryption

```bash
# Enable WireGuard on all nodes
cilium config set enable-wireguard true

# Verify encryption enabled
cilium status | grep Encryption
# Should show: Encryption: WireGuard
```

**Result**: All traffic between PHP apps and PostgreSQL is now **encrypted**, even on the same node.

### 5.2 Verify Encryption

```bash
# Check encrypted connections
cilium encrypt status

# Example output:
# Encryption: Wireguard
# Keys in use: 8
```

---

## Part 6: Replace kube-proxy (Performance Boost)

Cilium can replace kube-proxy with eBPF for **faster service load balancing**.

### 6.1 Verify kube-proxy Replacement Enabled

```bash
# Check if kube-proxy replacement is active
cilium status | grep KubeProxyReplacement
# Should show: KubeProxyReplacement: True

# Or check detailed status
kubectl -n kube-system exec ds/cilium -- cilium status --verbose | grep KubeProxyReplacement
```

### 6.2 Remove kube-proxy (Optional, if you installed it)

If you installed kube-proxy initially:

```bash
# Delete kube-proxy
kubectl -n kube-system delete ds kube-proxy

# On each node, clean up iptables rules
# (SSH to each node)
sudo iptables-save | grep -v KUBE | sudo iptables-restore
```

---

## Part 7: Monitoring and Observability

### 7.1 Prometheus Metrics

Cilium exports Prometheus metrics automatically.

```bash
# Install Prometheus (if not already)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring --create-namespace

# Cilium metrics are auto-discovered
# Query examples:
# - cilium_forward_count_total (packets forwarded)
# - cilium_drop_count_total (packets dropped)
# - cilium_policy_l7_total (L7 policy decisions)
```

### 7.2 Grafana Dashboards

Import Cilium dashboards to Grafana:

```bash
# Port-forward Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

# Login to Grafana (admin/prom-operator)
# Import dashboards:
# - Cilium Operator: https://grafana.com/grafana/dashboards/16611
# - Cilium Hubble: https://grafana.com/grafana/dashboards/16612
```

### 7.3 Monitor PostgreSQL Connections

```bash
# See all connections to PostgreSQL
hubble observe --protocol tcp --port 5432

# See connection latency
hubble observe --protocol tcp --port 5432 --verdict FORWARDED -o json | jq '.Summary.latency'

# Monitor PgBouncer connections
hubble observe --to-service pgbouncer --protocol tcp
```

---

## Part 8: Advanced Features for Production

### 8.1 BGP for LoadBalancer Services (Bare Metal)

Since you're on bare metal, use Cilium's BGP to advertise LoadBalancer IPs:

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPPeeringPolicy
metadata:
  name: bgp-peering
spec:
  nodeSelector:
    matchLabels:
      workload: application  # Only app workers announce IPs

  virtualRouters:
  - localASN: 65000
    exportPodCIDR: false
    neighbors:
    - peerAddress: 10.10.30.1  # Your gateway/router
      peerASN: 65001
EOF
```

**Use case**: Expose services with real LoadBalancer IPs instead of NodePort.

### 8.2 Local Redirect Policy (Optimize PostgreSQL Connections)

Keep PostgreSQL connections **local to the node** when possible (reduces network hops):

```yaml
apiVersion: cilium.io/v2
kind: CiliumLocalRedirectPolicy
metadata:
  name: postgres-local-redirect
spec:
  redirectFrontend:
    serviceMatcher:
      serviceName: pg-cluster-01-rw
      namespace: default
  redirectBackend:
    localEndpointSelector:
      matchLabels:
        cnpg.io/cluster: pg-cluster-01
    toPorts:
    - port: "5432"
      protocol: TCP
```

**Result**: PHP pods prefer PostgreSQL pods on the same node (if available).

### 8.3 Bandwidth Manager (QoS for PostgreSQL)

Prioritize PostgreSQL traffic over other traffic:

```bash
# Enable bandwidth manager
cilium config set enable-bandwidth-manager true

# Then annotate PostgreSQL pods
kubectl annotate pod <pg-pod-name> \
  kubernetes.io/egress-bandwidth=1G \
  kubernetes.io/ingress-bandwidth=1G
```

---

## Part 9: Troubleshooting

### Issue: Pods not getting IPs

```bash
# Check Cilium status on all nodes
cilium status

# Check Cilium agent logs
kubectl -n kube-system logs ds/cilium

# Check CNI config
ls -la /etc/cni/net.d/
# Should show: 05-cilium.conflist
```

### Issue: Network connectivity broken

```bash
# Run connectivity test
cilium connectivity test

# If fails, check eBPF programs loaded
kubectl -n kube-system exec ds/cilium -- cilium bpf lb list

# Restart Cilium
kubectl -n kube-system rollout restart ds/cilium
```

### Issue: Hubble not showing flows

```bash
# Check Hubble Relay
kubectl -n kube-system get pods -l k8s-app=hubble-relay

# Check Hubble enabled on agents
kubectl -n kube-system exec ds/cilium -- cilium status | grep Hubble
# Should show: Hubble: Ok
```

---

## Part 10: Performance Tuning for PostgreSQL Workloads

### 10.1 Optimize Socket Load Balancing

For PostgreSQL connections, use socket-level load balancing:

```bash
cilium config set bpf-lb-sock-hostns-only false
cilium config set enable-socket-lb true
```

### 10.2 Tune eBPF Map Sizes

For high connection count (PostgreSQL with many clients):

```yaml
# Edit Cilium ConfigMap
kubectl -n kube-system edit cm cilium-config

# Add:
data:
  bpf-ct-global-tcp-max: "1000000"  # Connection tracking (increase for many connections)
  bpf-nat-global-max: "1000000"     # NAT table size
```

### 10.3 Enable Direct Server Return (DSR)

For better load balancing performance:

```bash
cilium config set loadbalancer-mode dsr
```

---

## Comparison: Cilium vs Calico Performance

**Benchmark results** (typical):

| Metric | Calico | Cilium | Improvement |
|--------|--------|--------|-------------|
| Pod-to-pod latency | 0.5 ms | 0.3 ms | **40% faster** |
| Service load balancing | 2 ms | 0.8 ms | **60% faster** |
| Throughput | 9 Gbps | 9.5 Gbps | Slightly better |
| CPU usage (networking) | 5% | 3% | **40% less CPU** |

**For your PostgreSQL workload:**
- Lower latency = faster queries
- Less CPU = more resources for PHP/PostgreSQL
- Better observability = easier debugging

---

## Summary

You now have Cilium configured with:

✅ **eBPF-native networking** - kernel-level performance
✅ **Hubble observability** - visualize all network flows
✅ **kube-proxy replacement** - faster service load balancing
✅ **WireGuard encryption** - secure PostgreSQL traffic
✅ **Network policies** - control database access
✅ **Prometheus metrics** - monitor networking health

**Benefits for your PHP + PostgreSQL cluster:**
- 📉 Lower latency between app and database
- 📊 Full visibility into connection patterns
- 🔒 Encrypted database traffic
- ⚡ Better performance on bare metal
- 🛡️ Fine-grained access control

**Next steps:**
- Deploy Contour for ingress (see next guide)
- Configure network policies for production
- Set up Hubble UI for troubleshooting
- Monitor metrics in Grafana
