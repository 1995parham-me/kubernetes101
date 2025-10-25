# PostgreSQL with CloudNativePG (CNPG) Setup Guide

## Overview

This guide covers deploying a highly available PostgreSQL cluster using **CloudNativePG (CNPG)** operator on dedicated database worker nodes with local storage.

**PostgreSQL HA Strategy:**
- **3 replicas** across 3 dedicated PostgreSQL workers
- **Synchronous replication** for zero data loss
- **Automatic failover** when primary fails
- **Node affinity** ensures each replica on different physical host
- **Local storage** for high performance

---

## What is CloudNativePG (CNPG)?

**CloudNativePG** is a Kubernetes operator for PostgreSQL that provides:
- ✅ Automated deployment and management
- ✅ High availability with automatic failover
- ✅ Streaming replication (async and sync)
- ✅ Point-in-time recovery (PITR)
- ✅ Connection pooling (PgBouncer)
- ✅ Backup and restore (Barman)
- ✅ Monitoring (Prometheus metrics)

**Why CNPG over alternatives?**
- Modern, cloud-native design
- Active development by EDB (PostgreSQL company)
- Better than Stolon, Patroni operators (simpler, more K8s-native)
- Built-in backup with S3/Minio support

---

## Step 1: Install CNPG Operator

### 1.1 Install via Helm

```bash
# Add CNPG Helm repository
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm repo update

# Install CNPG operator
helm install cnpg \
  --namespace cnpg-system \
  --create-namespace \
  cnpg/cloudnative-pg

# Verify operator is running
kubectl get pods -n cnpg-system

# Should show:
# NAME                                          READY   STATUS    RESTARTS   AGE
# cnpg-cloudnative-pg-<hash>   1/1     Running   0          30s
```

### 1.2 Verify CRDs Installed

```bash
# Check Custom Resource Definitions
kubectl get crds | grep cnpg

# Should show:
# backups.postgresql.cnpg.io
# clusters.postgresql.cnpg.io
# poolers.postgresql.cnpg.io
# scheduledbackups.postgresql.cnpg.io
```

---

## Step 2: Configure Storage for PostgreSQL

### 2.1 Create StorageClass for PostgreSQL (Optional)

If you want a dedicated storage class for PostgreSQL with specific settings:

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path-postgres
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: rancher.io/local-path
reclaimPolicy: Retain  # Don't delete data if PVC deleted
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
EOF
```

**Or** use the default `local-path` storage class (created in previous guide).

---

## Step 3: Deploy PostgreSQL Cluster

### 3.1 Basic PostgreSQL Cluster (3 Replicas)

Create a PostgreSQL cluster with 3 instances, one on each PostgreSQL worker node:

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: pg-cluster-01
  namespace: default
spec:
  instances: 3

  # PostgreSQL version
  imageName: ghcr.io/cloudnative-pg/postgresql:16.3

  # Storage configuration
  storage:
    size: 100Gi
    storageClass: local-path

  # Resource limits (important for PHP memory-heavy workloads)
  resources:
    requests:
      memory: "32Gi"
      cpu: "8"
    limits:
      memory: "48Gi"
      cpu: "12"

  # Node affinity - run only on database workers
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: workload
            operator: In
            values:
            - database

    # Pod anti-affinity - ensure replicas on different nodes
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
      # Memory settings (tune for 48 GB limit)
      shared_buffers: "12GB"              # 25% of RAM
      effective_cache_size: "36GB"        # 75% of RAM
      maintenance_work_mem: "2GB"
      work_mem: "128MB"

      # Connection settings
      max_connections: "500"

      # Performance
      random_page_cost: "1.1"             # For SSD/NVMe
      effective_io_concurrency: "200"     # For SSD/NVMe

      # WAL settings
      wal_buffers: "16MB"
      min_wal_size: "1GB"
      max_wal_size: "4GB"

      # Checkpoint settings
      checkpoint_completion_target: "0.9"

      # Logging
      log_min_duration_statement: "1000"  # Log slow queries (>1s)
      log_line_prefix: "%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h "

      # Locale
      lc_messages: "C"
      lc_monetary: "C"
      lc_numeric: "C"
      lc_time: "C"

  # High Availability configuration
  minSyncReplicas: 1
  maxSyncReplicas: 2

  # Monitoring
  monitoring:
    enablePodMonitor: true

  # Bootstrap (empty database for new cluster)
  bootstrap:
    initdb:
      database: appdb
      owner: appuser
      secret:
        name: pg-cluster-01-app-user
EOF
```

### 3.2 Create Application User Secret

Before creating the cluster, create a secret for the application user:

```bash
# Generate strong password
PG_PASSWORD=$(openssl rand -base64 32)

# Create secret
kubectl create secret generic pg-cluster-01-app-user \
  --from-literal=username=appuser \
  --from-literal=password="${PG_PASSWORD}"

# Save password somewhere safe (password manager)
echo "PostgreSQL appuser password: ${PG_PASSWORD}"
```

### 3.3 Verify Cluster Deployment

```bash
# Watch cluster creation (takes 2-5 minutes)
kubectl get cluster pg-cluster-01 -w

# Should eventually show:
# NAME            AGE   INSTANCES   READY   STATUS                     PRIMARY
# pg-cluster-01   5m    3           3       Cluster in healthy state   pg-cluster-01-1

# Check pods
kubectl get pods -l cnpg.io/cluster=pg-cluster-01 -o wide

# Should show 3 pods on 3 different PostgreSQL workers:
# NAME              READY   STATUS    NODE
# pg-cluster-01-1   1/1     Running   k8s-pg-worker-01
# pg-cluster-01-2   1/1     Running   k8s-pg-worker-02
# pg-cluster-01-3   1/1     Running   k8s-pg-worker-03

# Check which is primary
kubectl get cluster pg-cluster-01 -o jsonpath='{.status.currentPrimary}'
```

---

## Step 4: Connect to PostgreSQL

### 4.1 Get Connection Information

CNPG creates several services:

```bash
# List services
kubectl get svc -l cnpg.io/cluster=pg-cluster-01

# Should show:
# pg-cluster-01-rw   ClusterIP   10.x.x.x   <none>   5432/TCP   (read-write, points to primary)
# pg-cluster-01-ro   ClusterIP   10.x.x.x   <none>   5432/TCP   (read-only, load balanced to replicas)
# pg-cluster-01-r    ClusterIP   10.x.x.x   <none>   5432/TCP   (read-only, includes primary)
```

**Service types:**
- **pg-cluster-01-rw**: Read-write (primary only) - use for writes
- **pg-cluster-01-ro**: Read-only replicas - use for read queries
- **pg-cluster-01-r**: All instances (primary + replicas)

### 4.2 Get Superuser Password

```bash
# Get postgres superuser password
kubectl get secret pg-cluster-01-superuser \
  -o jsonpath='{.data.password}' | base64 -d

# Get app user password
kubectl get secret pg-cluster-01-app-user \
  -o jsonpath='{.data.password}' | base64 -d
```

### 4.3 Connect from Within Cluster

```bash
# Create a psql client pod
kubectl run psql-client -it --rm --restart=Never \
  --image=postgres:16 \
  --env="PGPASSWORD=$(kubectl get secret pg-cluster-01-app-user -o jsonpath='{.data.password}' | base64 -d)" \
  -- psql -h pg-cluster-01-rw -U appuser -d appdb

# Once connected:
appdb=> SELECT version();
appdb=> \l          -- List databases
appdb=> \dt         -- List tables
appdb=> \q          -- Quit
```

### 4.4 Connect from PHP Application

**Connection string for write operations:**

```php
$host = 'pg-cluster-01-rw.default.svc.cluster.local';
$port = 5432;
$dbname = 'appdb';
$user = 'appuser';
$password = getenv('PG_PASSWORD');  // From Kubernetes secret

$dsn = "pgsql:host=$host;port=$port;dbname=$dbname";
$pdo = new PDO($dsn, $user, $password);
```

**Connection string for read operations** (read replicas):

```php
$host = 'pg-cluster-01-ro.default.svc.cluster.local';  // Read-only service
// ... same as above
```

**Using environment variables** (recommended):

```yaml
# In your PHP deployment:
env:
- name: DB_HOST
  value: pg-cluster-01-rw.default.svc.cluster.local
- name: DB_PORT
  value: "5432"
- name: DB_NAME
  value: appdb
- name: DB_USER
  valueFrom:
    secretKeyRef:
      name: pg-cluster-01-app-user
      key: username
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: pg-cluster-01-app-user
      key: password
```

---

## Step 5: Configure Connection Pooling (PgBouncer)

For PHP applications with many short-lived connections, use PgBouncer:

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: postgresql.cnpg.io/v1
kind: Pooler
metadata:
  name: pg-cluster-01-pooler
  namespace: default
spec:
  cluster:
    name: pg-cluster-01

  instances: 3  # 3 PgBouncer instances for HA
  type: rw      # Read-write pooler (for primary)

  pgbouncer:
    poolMode: transaction  # Good for PHP (session or transaction)
    parameters:
      max_client_conn: "1000"
      default_pool_size: "25"
      reserve_pool_size: "5"
      reserve_pool_timeout: "5"
      max_db_connections: "100"
      max_user_connections: "100"

  # Run pooler on app workers (not database workers)
  template:
    metadata:
      labels:
        app: pgbouncer
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: workload
                operator: In
                values:
                - application
EOF
```

**Connect to PgBouncer instead of PostgreSQL directly:**

```bash
# List pooler service
kubectl get svc pg-cluster-01-pooler-rw

# Update PHP connection to use pooler:
# $host = 'pg-cluster-01-pooler-rw.default.svc.cluster.local';
```

---

## Step 6: Backup Configuration

### 6.1 Configure S3-Compatible Backup (MinIO or Backblaze B2)

**Option A: Deploy MinIO in-cluster** (for development/testing):

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: minio-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClass: local-path
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
      - name: minio
        image: minio/minio:latest
        args:
        - server
        - /data
        - --console-address
        - ":9001"
        env:
        - name: MINIO_ROOT_USER
          value: "minioadmin"
        - name: MINIO_ROOT_PASSWORD
          value: "minioadmin123"  # Change in production!
        ports:
        - containerPort: 9000
        - containerPort: 9001
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: minio-pvc
      nodeSelector:
        workload: application
---
apiVersion: v1
kind: Service
metadata:
  name: minio
spec:
  selector:
    app: minio
  ports:
  - name: api
    port: 9000
    targetPort: 9000
  - name: console
    port: 9001
    targetPort: 9001
EOF
```

**Create bucket in MinIO:**

```bash
# Port-forward to MinIO console
kubectl port-forward svc/minio 9001:9001

# Access console: http://localhost:9001
# Login: minioadmin / minioadmin123
# Create bucket: "postgres-backups"
```

**Option B: Use Backblaze B2 or AWS S3** (recommended for production):

```bash
# Create secret with S3 credentials
kubectl create secret generic pg-backup-s3-creds \
  --from-literal=ACCESS_KEY_ID=<your-access-key> \
  --from-literal=SECRET_ACCESS_KEY=<your-secret-key>
```

### 6.2 Configure CNPG Backup

Update PostgreSQL cluster with backup configuration:

```yaml
kubectl edit cluster pg-cluster-01

# Add backup section:
spec:
  # ... existing config ...

  backup:
    barmanObjectStore:
      destinationPath: s3://postgres-backups/pg-cluster-01
      endpointURL: http://minio.default.svc.cluster.local:9000  # Or https://s3.amazonaws.com
      s3Credentials:
        accessKeyId:
          name: pg-backup-s3-creds
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: pg-backup-s3-creds
          key: SECRET_ACCESS_KEY

      wal:
        compression: gzip
        maxParallel: 2

      data:
        compression: gzip
        jobs: 2

    retentionPolicy: "30d"  # Keep backups for 30 days
```

### 6.3 Create Backup Schedule

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata:
  name: pg-cluster-01-daily-backup
  namespace: default
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  backupOwnerReference: self
  cluster:
    name: pg-cluster-01
  method: barmanObjectStore
EOF
```

### 6.4 Manual Backup

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: pg-cluster-01-manual-backup-$(date +%Y%m%d-%H%M%S)
  namespace: default
spec:
  cluster:
    name: pg-cluster-01
  method: barmanObjectStore
EOF

# Check backup status
kubectl get backups
```

---

## Step 7: Monitoring

### 7.1 Install Prometheus (if not already installed)

```bash
# Add Prometheus Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

### 7.2 Verify CNPG Metrics

CNPG exports Prometheus metrics automatically if `monitoring.enablePodMonitor: true` is set.

```bash
# Check PodMonitor
kubectl get podmonitor -l cnpg.io/cluster=pg-cluster-01

# Access Prometheus UI
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090

# Query: pg_stat_database_tup_fetched
# Query: cnpg_pg_replication_lag
```

### 7.3 Install Grafana Dashboard

```bash
# Access Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

# Login: admin / prom-operator (default)

# Import CNPG dashboard:
# Dashboard ID: 20417 (from grafana.com)
```

---

## Step 8: High Availability Testing

### 8.1 Test Automatic Failover

**Simulate primary failure:**

```bash
# Identify current primary
PRIMARY=$(kubectl get cluster pg-cluster-01 -o jsonpath='{.status.currentPrimary}')
echo "Current primary: $PRIMARY"

# Delete primary pod (simulate crash)
kubectl delete pod $PRIMARY

# Watch failover (should take 30-60 seconds)
kubectl get cluster pg-cluster-01 -w

# New primary will be elected
# Cluster should remain healthy
```

**Verify failover:**

```bash
# Check new primary
kubectl get cluster pg-cluster-01 -o jsonpath='{.status.currentPrimary}'

# Check replication status
kubectl exec -it pg-cluster-01-1 -- psql -U postgres -c "SELECT * FROM pg_stat_replication;"
```

### 8.2 Test Read Replica Load Balancing

```bash
# Connect to read-only service multiple times
for i in {1..10}; do
  kubectl run psql-test-$i --rm -it --restart=Never --image=postgres:16 \
    --env="PGPASSWORD=$(kubectl get secret pg-cluster-01-superuser -o jsonpath='{.data.password}' | base64 -d)" \
    -- psql -h pg-cluster-01-ro -U postgres -c "SELECT inet_server_addr();"
done

# Should show different IPs (load balanced across replicas)
```

---

## Step 9: Scaling and Maintenance

### 9.1 Scale PostgreSQL Cluster

```bash
# Scale to 5 replicas (if you add more database workers)
kubectl patch cluster pg-cluster-01 --type merge -p '{"spec":{"instances":5}}'

# Or scale down to 3
kubectl patch cluster pg-cluster-01 --type merge -p '{"spec":{"instances":3}}'
```

### 9.2 Upgrade PostgreSQL Version

```bash
# Update image version
kubectl patch cluster pg-cluster-01 --type merge \
  -p '{"spec":{"imageName":"ghcr.io/cloudnative-pg/postgresql:16.4"}}'

# CNPG will perform rolling upgrade (one pod at a time)
kubectl get cluster pg-cluster-01 -w
```

### 9.3 Expand Storage

```bash
# Increase PVC size (if storage class supports expansion)
kubectl patch cluster pg-cluster-01 --type merge \
  -p '{"spec":{"storage":{"size":"200Gi"}}}'
```

---

## Performance Tuning for PHP Workloads

### Optimize PostgreSQL for PHP Applications

**PHP applications typically have:**
- Many short-lived connections → Use PgBouncer (Step 5)
- Heavy reads vs writes → Use read replicas (pg-cluster-01-ro)
- Memory-intensive queries → Tune work_mem and shared_buffers

**Recommended `postgresql.parameters` for PHP:**

```yaml
postgresql:
  parameters:
    # Connection pooling (with PgBouncer)
    max_connections: "500"

    # Memory for PHP query workloads
    shared_buffers: "12GB"
    effective_cache_size: "36GB"
    work_mem: "256MB"  # Increase if complex queries
    maintenance_work_mem: "2GB"

    # Query planner
    random_page_cost: "1.1"  # For NVMe
    effective_io_concurrency: "200"

    # Autovacuum (important for heavy writes)
    autovacuum_max_workers: "4"
    autovacuum_naptime: "10s"

    # Logging (for slow query analysis)
    log_min_duration_statement: "500"  # Log queries >500ms
    log_statement: "mod"  # Log all DDL/DML
```

---

## Troubleshooting

### Cluster not starting

```bash
# Check operator logs
kubectl logs -n cnpg-system deployment/cnpg-cloudnative-pg

# Check cluster events
kubectl describe cluster pg-cluster-01

# Check pod logs
kubectl logs pg-cluster-01-1
```

### Replication lag

```bash
# Check replication status
kubectl exec -it pg-cluster-01-1 -- psql -U postgres \
  -c "SELECT client_addr, state, sync_state, replay_lag FROM pg_stat_replication;"

# Check cluster status
kubectl get cluster pg-cluster-01 -o yaml | grep -A 20 status
```

### Backup failures

```bash
# Check backup status
kubectl get backups

kubectl describe backup <backup-name>

# Check S3 connectivity from PostgreSQL pod
kubectl exec -it pg-cluster-01-1 -- curl -v http://minio.default.svc.cluster.local:9000
```

---

## Summary

You now have:
- ✅ 3-node PostgreSQL cluster with automatic failover
- ✅ Synchronous replication for zero data loss
- ✅ Node affinity ensuring replicas on different physical hosts
- ✅ PgBouncer connection pooling for PHP applications
- ✅ Automated backups to S3/MinIO
- ✅ Monitoring with Prometheus/Grafana
- ✅ High performance local storage

**Next steps:**
- Deploy your PHP applications
- Configure ingress controller (Nginx/Traefik)
- Set up CI/CD pipeline
- Configure application-level monitoring

Your Kubernetes cluster is production-ready!
