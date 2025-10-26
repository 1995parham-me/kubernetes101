# Quick Reference Guide

Essential commands and configurations for daily operations.

---

## Cluster Health Checks

### Quick Health Check
```bash
# Check all nodes
kubectl get nodes

# Check all pods
kubectl get pods -A

# Check PostgreSQL cluster
kubectl get cluster pg-cluster-01

# Check Cilium status
cilium status

# Check Contour
kubectl get httpproxy -A
```

### Detailed Health Check
```bash
# Node resources
kubectl top nodes

# Pod resources
kubectl top pods -A

# Check for failed pods
kubectl get pods -A | grep -v Running | grep -v Completed

# Recent events
kubectl get events --sort-by='.lastTimestamp' | tail -20

# PostgreSQL replication status
kubectl exec -it pg-cluster-01-1 -- psql -U postgres -c "SELECT * FROM pg_stat_replication;"
```

---

## Common Operations

### Restart a Pod
```bash
# Delete pod (Kubernetes will recreate it)
kubectl delete pod <pod-name>

# Or rollout restart deployment
kubectl rollout restart deployment/<deployment-name>
```

### Scale Deployment
```bash
# Scale up/down
kubectl scale deployment/<deployment-name> --replicas=5

# Check rollout status
kubectl rollout status deployment/<deployment-name>
```

### View Logs
```bash
# Pod logs
kubectl logs <pod-name>

# Follow logs
kubectl logs -f <pod-name>

# Previous crashed container
kubectl logs <pod-name> --previous

# All pods in deployment
kubectl logs -l app=<app-label> --tail=50

# Cilium logs
kubectl logs -n kube-system daemonset/cilium

# Contour logs
kubectl logs -n projectcontour deployment/contour
```

### Exec into Pod
```bash
# Interactive shell
kubectl exec -it <pod-name> -- /bin/bash

# Run single command
kubectl exec <pod-name> -- ls -la /data

# PostgreSQL access
kubectl exec -it pg-cluster-01-1 -- psql -U postgres
```

---

## Networking (Cilium)

### Check Network Connectivity
```bash
# Cilium connectivity test
cilium connectivity test

# Check Cilium status on all nodes
kubectl -n kube-system exec daemonset/cilium -- cilium status

# View network policies
kubectl get ciliumnetworkpolicy
```

### Hubble (Network Observability)
```bash
# Port-forward Hubble
cilium hubble port-forward &

# Watch all network flows
hubble observe

# Watch PostgreSQL connections
hubble observe --protocol tcp --port 5432

# Watch specific pod traffic
hubble observe --from-pod <pod-name>

# Hubble UI
cilium hubble ui
# Opens browser to http://localhost:12000
```

### Network Policies
```bash
# List network policies
kubectl get ciliumnetworkpolicy

# Describe policy
kubectl describe ciliumnetworkpolicy <policy-name>

# Test connectivity
kubectl run test-pod --rm -it --image=busybox -- wget -O- http://<service-name>
```

---

## Storage (TopoLVM)

### Check Storage
```bash
# List PVCs
kubectl get pvc

# Check PV status
kubectl get pv

# Node storage capacity
kubectl get nodes -o custom-columns=NAME:.metadata.name,CAPACITY:.status.capacity.topolvm\\.io/capacity

# On each node, check LVM
ssh root@<node-ip>
sudo lvs
sudo vgs
```

### Create PVC
```yaml
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-data
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: topolvm-provisioner
  resources:
    requests:
      storage: 50Gi
EOF
```

### Volume Snapshots
```bash
# Create snapshot
kubectl create -f - <<EOF
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: my-snapshot
spec:
  volumeSnapshotClassName: topolvm-snapshot
  source:
    persistentVolumeClaimName: my-pvc
EOF

# List snapshots
kubectl get volumesnapshot

# Restore from snapshot
kubectl create -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-pvc
spec:
  dataSource:
    name: my-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
  - ReadWriteOnce
  storageClassName: topolvm-provisioner
  resources:
    requests:
      storage: 50Gi
EOF
```

### Expand Volume
```bash
# Increase PVC size
kubectl patch pvc <pvc-name> -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}'

# Check expansion status
kubectl describe pvc <pvc-name>
```

---

## Ingress (Contour)

### List Routes
```bash
# All HTTPProxy objects
kubectl get httpproxy -A

# Describe HTTPProxy
kubectl describe httpproxy <name>

# Check Envoy service
kubectl get svc -n projectcontour envoy
```

### Create HTTPProxy
```yaml
kubectl apply -f - <<EOF
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: my-app
spec:
  virtualhost:
    fqdn: app.example.com
    tls:
      secretName: app-tls
  routes:
  - services:
    - name: my-app-service
      port: 80
EOF
```

### Check TLS Certificates
```bash
# List certificates
kubectl get certificate

# Check cert-manager
kubectl get certificaterequest
kubectl describe certificate <cert-name>

# Check TLS secret
kubectl get secret <tls-secret-name> -o yaml
```

### Test Ingress
```bash
# Get Envoy IP
ENVOY_IP=$(kubectl get svc envoy -n projectcontour -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Test HTTP
curl -H "Host: app.example.com" http://$ENVOY_IP/

# Test HTTPS
curl https://app.example.com/
```

---

## PostgreSQL (CNPG)

### Cluster Status
```bash
# Check cluster
kubectl get cluster pg-cluster-01

# Detailed status
kubectl describe cluster pg-cluster-01

# Which pod is primary?
kubectl get cluster pg-cluster-01 -o jsonpath='{.status.currentPrimary}'

# Check all PostgreSQL pods
kubectl get pods -l cnpg.io/cluster=pg-cluster-01
```

### Connect to Database
```bash
# Get superuser password
PGPASSWORD=$(kubectl get secret pg-cluster-01-superuser -o jsonpath='{.data.password}' | base64 -d)

# Connect via psql client pod
kubectl run psql-client -it --rm --image=postgres:16 \
  --env="PGPASSWORD=$PGPASSWORD" \
  -- psql -h pg-cluster-01-rw -U postgres -d appdb

# Or exec into PostgreSQL pod directly
kubectl exec -it pg-cluster-01-1 -- psql -U postgres
```

### Replication Status
```bash
# Check replication
kubectl exec -it pg-cluster-01-1 -- psql -U postgres -c "
  SELECT client_addr, state, sync_state, replay_lag
  FROM pg_stat_replication;
"

# Check which is primary
kubectl exec -it pg-cluster-01-1 -- psql -U postgres -c "SELECT pg_is_in_recovery();"
# t = replica, f = primary
```

### Backups
```bash
# List backups
kubectl get backups

# Trigger manual backup
kubectl create -f - <<EOF
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: manual-backup-$(date +%Y%m%d-%H%M%S)
spec:
  cluster:
    name: pg-cluster-01
EOF

# Check backup status
kubectl describe backup <backup-name>
```

### Failover Test
```bash
# Get current primary
PRIMARY=$(kubectl get cluster pg-cluster-01 -o jsonpath='{.status.currentPrimary}')
echo "Current primary: $PRIMARY"

# Delete primary pod (simulate failure)
kubectl delete pod $PRIMARY

# Watch failover (should take 30-60 seconds)
kubectl get cluster pg-cluster-01 -w

# Verify new primary
kubectl get cluster pg-cluster-01 -o jsonpath='{.status.currentPrimary}'
```

---

## Troubleshooting

### Pod Won't Start
```bash
# Check events
kubectl describe pod <pod-name>

# Check logs
kubectl logs <pod-name>

# Check previous container logs (if CrashLoopBackOff)
kubectl logs <pod-name> --previous

# Check node resources
kubectl describe node <node-name>
kubectl top node <node-name>
```

### Network Issues
```bash
# Test DNS
kubectl run test-dns --rm -it --image=busybox -- nslookup kubernetes.default

# Test service connectivity
kubectl run test-svc --rm -it --image=busybox -- wget -O- http://<service-name>

# Check Cilium
cilium connectivity test

# Check network policies
kubectl get ciliumnetworkpolicy
```

### Storage Issues
```bash
# Check PVC status
kubectl describe pvc <pvc-name>

# Check if bound
kubectl get pvc | grep Pending

# Check node storage
kubectl get nodes -o custom-columns=NAME:.metadata.name,CAPACITY:.status.capacity.topolvm\\.io/capacity

# On node, check LVM
ssh root@<node-ip>
sudo lvs
sudo vgs
sudo df -h
```

### PostgreSQL Issues
```bash
# Check cluster status
kubectl get cluster pg-cluster-01

# Check CNPG operator logs
kubectl logs -n cnpg-system deployment/cnpg-cloudnative-pg

# Check PostgreSQL logs
kubectl logs pg-cluster-01-1

# Check replication
kubectl exec -it pg-cluster-01-1 -- psql -U postgres -c "SELECT * FROM pg_stat_replication;"
```

---

## Maintenance

### Drain Node (for Maintenance)
```bash
# Drain node (evict pods)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Do maintenance (e.g., reboot, upgrade)

# Uncordon node (allow scheduling)
kubectl uncordon <node-name>

# Verify node ready
kubectl get node <node-name>
```

### Update Deployment Image
```bash
# Set new image
kubectl set image deployment/<deployment-name> <container-name>=<new-image>:tag

# Watch rollout
kubectl rollout status deployment/<deployment-name>

# Rollback if needed
kubectl rollout undo deployment/<deployment-name>

# Check history
kubectl rollout history deployment/<deployment-name>
```

### etcd Backup
```bash
# On master node
sudo ETCDCTL_API=3 etcdctl snapshot save /root/etcd-backup-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
sudo ETCDCTL_API=3 etcdctl snapshot status /root/etcd-backup-*.db

# Copy off-server
scp /root/etcd-backup-*.db user@backup-server:/backups/
```

---

## Monitoring

### Resource Usage
```bash
# Node CPU/Memory
kubectl top nodes

# Pod CPU/Memory
kubectl top pods -A

# Specific namespace
kubectl top pods -n default

# Sort by CPU
kubectl top pods -A --sort-by=cpu

# Sort by memory
kubectl top pods -A --sort-by=memory
```

### Metrics
```bash
# Port-forward Prometheus
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090

# Port-forward Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

# Cilium metrics
kubectl port-forward -n kube-system svc/cilium-agent 9090:9090

# Envoy metrics
kubectl port-forward -n projectcontour <envoy-pod> 19000:19000
# Visit http://localhost:19000/stats
```

---

## Emergency Procedures

### Cluster Not Responding
```bash
# SSH to master node
ssh root@10.10.30.10

# Check kubelet
sudo systemctl status kubelet
sudo journalctl -u kubelet -f

# Check API server
kubectl get componentstatuses

# Restart kubelet (if needed)
sudo systemctl restart kubelet
```

### PostgreSQL Cluster Down
```bash
# Check all PostgreSQL pods
kubectl get pods -l cnpg.io/cluster=pg-cluster-01

# Check cluster status
kubectl describe cluster pg-cluster-01

# Check CNPG operator
kubectl logs -n cnpg-system deployment/cnpg-cloudnative-pg

# If all pods down, check node issues
kubectl get nodes
```

### Restore from Backup
```bash
# Restore PostgreSQL from backup
kubectl create -f - <<EOF
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: pg-cluster-restored
spec:
  instances: 3
  bootstrap:
    recovery:
      backup:
        name: <backup-name>
  # ... rest of config same as original
EOF
```

---

## Useful Aliases

Add to `~/.bashrc`:

```bash
# Kubectl shortcuts
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgn='kubectl get nodes'
alias kdp='kubectl describe pod'
alias kl='kubectl logs'
alias klf='kubectl logs -f'
alias kex='kubectl exec -it'

# Namespace shortcuts
alias kn='kubectl config set-context --current --namespace'

# PostgreSQL
alias pgcli='kubectl run psql-client -it --rm --image=postgres:16 --env="PGPASSWORD=$(kubectl get secret pg-cluster-01-superuser -o jsonpath={.data.password} | base64 -d)" -- psql -h pg-cluster-01-rw -U postgres'

# Cilium
alias cilium-status='cilium status'
alias hubble-observe='hubble observe'

# Watch commands
alias watch-nodes='watch kubectl get nodes'
alias watch-pods='watch kubectl get pods -A'
```

---

## Useful kubectl Plugins

Install with `kubectl krew`:

```bash
# Install krew
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)

# Add to PATH
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"

# Useful plugins
kubectl krew install ctx        # Switch contexts
kubectl krew install ns         # Switch namespaces
kubectl krew install tree       # Show resource tree
kubectl krew install tail       # Tail multiple pods
kubectl krew install resource-capacity  # Node capacity
```

---

## Important URLs

When port-forwarding is active:

- **Kubernetes Dashboard**: http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
- **Prometheus**: http://localhost:9090
- **Grafana**: http://localhost:3000 (admin/prom-operator)
- **Hubble UI**: http://localhost:12000
- **Envoy Admin**: http://localhost:19000

---

This quick reference should cover 90% of daily operations!
