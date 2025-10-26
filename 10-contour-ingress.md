# Contour Ingress Controller Setup

## Overview

This guide covers installing **Contour** as the ingress controller for Kubernetes, using Envoy proxy for modern HTTP/2, gRPC, and WebSocket support.

**Why Contour?**
- ✅ **Envoy proxy** - modern, high-performance L7 load balancer
- ✅ **HTTPProxy CRD** - type-safe, no annotation hell
- ✅ **Dynamic configuration** - zero-downtime updates (no Nginx reload)
- ✅ **gRPC support** - native support for gRPC backends
- ✅ **WebSockets** - perfect for real-time apps
- ✅ **Multi-tenancy** - namespace delegation
- ✅ **Observability** - rich metrics, distributed tracing

**Perfect for your PHP apps:**
- Clean configuration (no complex annotations)
- Zero-downtime deployments (hot reload)
- Better HTTP/2 support for modern browsers
- Native TLS management
- Canary and blue-green deployments

---

## Architecture

```
Internet
    ↓
[Load Balancer / NodePort] (10.10.30.x:80/443)
    ↓
[Envoy Proxy Pods] (running on app workers)
    ↓ (route based on HTTPProxy rules)
    ├→ PHP App 1 (app.example.com/api)
    ├→ PHP App 2 (app.example.com/admin)
    └→ PHP App 3 (www.example.com)
```

**Components:**
- **Contour**: Control plane, converts HTTPProxy → Envoy config
- **Envoy**: Data plane, actual proxy handling traffic
- **HTTPProxy**: Custom resource defining routing rules

---

## Part 1: Install Contour

### 1.1 Install via kubectl

```bash
# Download Contour manifest
kubectl apply -f https://projectcontour.io/quickstart/contour.yaml

# Wait for Contour pods (2-3 minutes)
kubectl wait --for=condition=available --timeout=300s deployment/contour -n projectcontour

# Check installation
kubectl get pods -n projectcontour

# Expected output:
# NAME                       READY   STATUS    RESTARTS   AGE
# contour-xxxx               1/1     Running   0          2m
# contour-xxxx               1/1     Running   0          2m
# envoy-xxxx                 2/2     Running   0          2m  # One per node (DaemonSet)
# envoy-xxxx                 2/2     Running   0          2m
```

### 1.2 Verify Installation

```bash
# Check CRDs installed
kubectl get crd | grep projectcontour

# Should show:
# httpproxies.projectcontour.io
# tlscertificatedelegations.projectcontour.io
# extensionservices.projectcontour.io

# Check Envoy service
kubectl get svc -n projectcontour

# Should show:
# NAME      TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)
# contour   ClusterIP      10.96.x.x       <none>        8001/TCP
# envoy     LoadBalancer   10.96.x.x       <pending>     80:30080/TCP,443:30443/TCP
```

---

## Part 2: Expose Envoy Service (Bare Metal Options)

Since you're on bare metal (not cloud), the Envoy LoadBalancer service will stay `<pending>`. You have three options:

### Option A: NodePort (Simplest)

Use NodePort and access via node IPs.

```bash
# Envoy already exposed on NodePorts (from default install)
kubectl get svc envoy -n projectcontour

# Should show:
# envoy   LoadBalancer   10.96.x.x   <pending>   80:30080/TCP,443:30443/TCP

# Access your apps via:
# http://<any-worker-node-ip>:30080
# https://<any-worker-node-ip>:30443
```

**Pros**: Simple, no extra setup
**Cons**: Non-standard ports (30080/30443)

### Option B: MetalLB (Recommended)

Install MetalLB to provide LoadBalancer IPs from your colocation IP range.

```bash
# Install MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.0/config/manifests/metallb-native.yaml

# Wait for MetalLB pods
kubectl wait --for=condition=ready pod -l app=metallb -n metallb-system --timeout=300s

# Configure IP address pool (use your colocation IPs)
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.10.30.100-10.10.30.110  # Adjust to your available IPs
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-advert
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
EOF

# Check Envoy service now has EXTERNAL-IP
kubectl get svc envoy -n projectcontour

# Should show:
# envoy   LoadBalancer   10.96.x.x   10.10.30.100   80:30080/TCP,443:30443/TCP
```

**Pros**: Standard ports (80/443), clean IPs
**Cons**: Requires IP allocation from colo provider

### Option C: hostPort (Advanced)

Run Envoy on ports 80/443 directly on nodes.

```bash
# Edit Envoy DaemonSet
kubectl edit daemonset envoy -n projectcontour

# Add hostPort to container ports:
spec:
  template:
    spec:
      containers:
      - name: envoy
        ports:
        - containerPort: 8080
          hostPort: 80        # Add this
          protocol: TCP
        - containerPort: 8443
          hostPort: 443       # Add this
          protocol: TCP
```

**Pros**: Standard ports, no MetalLB needed
**Cons**: Only one Envoy per node, conflicts if something else uses 80/443

---

## Part 3: Deploy First HTTPProxy

### 3.1 Create Sample PHP Application

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-app-demo
  labels:
    app: php-app-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: php-app-demo
  template:
    metadata:
      labels:
        app: php-app-demo
    spec:
      nodeSelector:
        workload: application  # Run on app workers
      containers:
      - name: php
        image: php:8.2-apache
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: php-app-demo
spec:
  selector:
    app: php-app-demo
  ports:
  - port: 80
    targetPort: 80
EOF
```

### 3.2 Create HTTPProxy

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: php-app-demo
  namespace: default
spec:
  virtualhost:
    fqdn: app.example.com
  routes:
  - conditions:
    - prefix: /
    services:
    - name: php-app-demo
      port: 80
EOF
```

### 3.3 Test Access

```bash
# Get Envoy external IP or NodePort
ENVOY_IP=$(kubectl get svc envoy -n projectcontour -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
# Or if using NodePort:
ENVOY_IP=<any-worker-node-ip>
ENVOY_PORT=30080

# Test with curl
curl -H "Host: app.example.com" http://$ENVOY_IP:$ENVOY_PORT/

# Should return PHP info page
```

---

## Part 4: TLS/HTTPS Configuration

### 4.1 Install cert-manager (for Let's Encrypt)

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.15.0/cert-manager.yaml

# Wait for cert-manager
kubectl wait --for=condition=available --timeout=300s deployment/cert-manager -n cert-manager

# Verify installation
kubectl get pods -n cert-manager
```

### 4.2 Create Let's Encrypt ClusterIssuer

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com  # Change this!
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: contour
EOF
```

### 4.3 Create HTTPProxy with TLS

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: php-app-demo
  namespace: default
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  virtualhost:
    fqdn: app.example.com
    tls:
      secretName: app-example-com-tls  # cert-manager will create this
  routes:
  - conditions:
    - prefix: /
    services:
    - name: php-app-demo
      port: 80
EOF
```

### 4.4 Verify TLS Certificate

```bash
# Check certificate
kubectl get certificate

# Should show:
# NAME                  READY   SECRET                AGE
# app-example-com-tls   True    app-example-com-tls   2m

# Test HTTPS
curl https://app.example.com/
# Should work with valid TLS
```

---

## Part 5: Advanced Routing

### 5.1 Path-Based Routing

Route different paths to different services:

```yaml
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: php-multi-app
spec:
  virtualhost:
    fqdn: app.example.com
    tls:
      secretName: app-tls
  routes:
  # /api → API service
  - conditions:
    - prefix: /api
    services:
    - name: php-api
      port: 80

  # /admin → Admin service
  - conditions:
    - prefix: /admin
    services:
    - name: php-admin
      port: 80

  # / → Main app
  - conditions:
    - prefix: /
    services:
    - name: php-main
      port: 80
```

### 5.2 Header-Based Routing

Route based on HTTP headers:

```yaml
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: php-header-routing
spec:
  virtualhost:
    fqdn: app.example.com
  routes:
  # API v2 (with header: X-API-Version: v2)
  - conditions:
    - header:
        name: X-API-Version
        exact: v2
    services:
    - name: php-api-v2
      port: 80

  # API v1 (default)
  - services:
    - name: php-api-v1
      port: 80
```

### 5.3 Weighted Traffic Splitting (Canary Deployment)

Send 10% traffic to new version:

```yaml
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: php-canary
spec:
  virtualhost:
    fqdn: app.example.com
  routes:
  - services:
    - name: php-app-stable
      port: 80
      weight: 90  # 90% traffic to stable
    - name: php-app-canary
      port: 80
      weight: 10  # 10% traffic to canary
```

---

## Part 6: Database Connection Proxy (For PostgreSQL)

### 6.1 Expose PgBouncer via HTTPProxy

If you want external access to PostgreSQL (via PgBouncer):

```yaml
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: pgbouncer-tcp
  namespace: default
spec:
  tcpproxy:
    services:
    - name: pg-cluster-01-pooler-rw
      port: 5432
  virtualhost:
    fqdn: db.example.com
    tls:
      secretName: db-tls
```

**Note**: This is for TCP passthrough, not HTTP. Use only if you need external database access.

---

## Part 7: Rate Limiting and Protection

### 7.1 Enable Rate Limiting

Protect your PHP apps from abuse:

```yaml
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: php-app-ratelimit
spec:
  virtualhost:
    fqdn: app.example.com
    rateLimitPolicy:
      global:
        descriptors:
        - entries:
          - genericKey:
              value: php-app-limit
      local:
        requests: 100
        unit: minute
  routes:
  - services:
    - name: php-app
      port: 80
```

**Result**: Max 100 requests per minute per client IP.

### 7.2 Timeout Configuration

Set timeouts for slow backends:

```yaml
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: php-app-timeout
spec:
  virtualhost:
    fqdn: app.example.com
  routes:
  - timeoutPolicy:
      response: 30s  # Max 30s for response
      idle: 5m       # Max 5 min idle connection
    services:
    - name: php-app
      port: 80
```

### 7.3 Retry Policy

Auto-retry failed requests:

```yaml
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: php-app-retry
spec:
  virtualhost:
    fqdn: app.example.com
  routes:
  - retryPolicy:
      count: 3              # Retry up to 3 times
      perTryTimeout: 5s     # Each attempt max 5s
      retriableStatusCodes:
      - 503                 # Retry on 503 Service Unavailable
    services:
    - name: php-app
      port: 80
```

---

## Part 8: Monitoring and Observability

### 8.1 Prometheus Metrics

Contour and Envoy export Prometheus metrics.

```bash
# Check metrics endpoints
kubectl port-forward -n projectcontour deployment/contour 8000:8000

# In another terminal:
curl http://localhost:8000/metrics

# Key metrics:
# contour_httpproxy_total - Total HTTPProxy objects
# contour_httpproxy_valid - Valid HTTPProxy objects
# envoy_http_downstream_rq_total - Total HTTP requests
# envoy_http_downstream_rq_xx - Requests by status code (2xx, 4xx, 5xx)
```

### 8.2 Grafana Dashboards

Import Envoy dashboards:

```bash
# In Grafana, import dashboard:
# Envoy Global: https://grafana.com/grafana/dashboards/11021
# Envoy Clusters: https://grafana.com/grafana/dashboards/11022
```

### 8.3 Access Logs

Enable Envoy access logs for debugging:

```yaml
# Edit Contour ConfigMap
kubectl edit configmap contour -n projectcontour

# Add:
data:
  contour.yaml: |
    accesslog-format: json
```

View logs:

```bash
kubectl logs -n projectcontour daemonset/envoy -c envoy --tail=50
```

---

## Part 9: Multi-Tenancy (Namespace Delegation)

Allow teams to manage their own HTTPProxy objects.

### 9.1 Root HTTPProxy (Platform Team)

```yaml
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: root-proxy
  namespace: projectcontour
spec:
  virtualhost:
    fqdn: "*.apps.example.com"
    tls:
      secretName: wildcard-tls
  includes:
  - name: team-a-proxy
    namespace: team-a
  - name: team-b-proxy
    namespace: team-b
```

### 9.2 Delegated HTTPProxy (Team A)

```yaml
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: team-a-proxy
  namespace: team-a
spec:
  routes:
  - conditions:
    - prefix: /team-a
    services:
    - name: team-a-app
      port: 80
```

**Result**: Team A can manage their own routing without platform team intervention.

---

## Part 10: Blue-Green Deployments

### 10.1 Setup Two Deployments

```yaml
# Blue deployment (current production)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-app-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: php-app
      version: blue
  template:
    metadata:
      labels:
        app: php-app
        version: blue
    spec:
      containers:
      - name: php
        image: myregistry.com/php-app:v1.0
---
# Green deployment (new version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-app-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: php-app
      version: green
  template:
    metadata:
      labels:
        app: php-app
        version: green
    spec:
      containers:
      - name: php
        image: myregistry.com/php-app:v2.0
---
# Services
apiVersion: v1
kind: Service
metadata:
  name: php-app-blue
spec:
  selector:
    app: php-app
    version: blue
  ports:
  - port: 80
---
apiVersion: v1
kind: Service
metadata:
  name: php-app-green
spec:
  selector:
    app: php-app
    version: green
  ports:
  - port: 80
```

### 10.2 Route 100% to Blue (Initial State)

```yaml
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: php-app
spec:
  virtualhost:
    fqdn: app.example.com
  routes:
  - services:
    - name: php-app-blue
      port: 80
      weight: 100
    - name: php-app-green
      port: 80
      weight: 0
```

### 10.3 Switch to Green (Cutover)

```bash
# Edit HTTPProxy
kubectl patch httpproxy php-app --type merge -p '
{
  "spec": {
    "routes": [{
      "services": [
        {"name": "php-app-blue", "port": 80, "weight": 0},
        {"name": "php-app-green", "port": 80, "weight": 100}
      ]
    }]
  }
}'

# Instant cutover! No downtime, Envoy hot-reloads config
```

### 10.4 Rollback if Needed

```bash
# Switch back to blue
kubectl patch httpproxy php-app --type merge -p '
{
  "spec": {
    "routes": [{
      "services": [
        {"name": "php-app-blue", "port": 80, "weight": 100},
        {"name": "php-app-green", "port": 80, "weight": 0}
      ]
    }]
  }
}'
```

---

## Part 11: Troubleshooting

### Issue: HTTPProxy shows "invalid"

```bash
# Check HTTPProxy status
kubectl get httpproxy php-app -o yaml

# Look at status.conditions
# Common issues:
# - Service doesn't exist
# - TLS secret missing
# - Invalid FQDN

# Check Contour logs
kubectl logs -n projectcontour deployment/contour
```

### Issue: 503 Service Unavailable

```bash
# Check backend pods
kubectl get pods -l app=php-app

# Check service endpoints
kubectl get endpoints php-app

# Check Envoy logs
kubectl logs -n projectcontour daemonset/envoy -c envoy | grep php-app

# Check Envoy config
kubectl port-forward -n projectcontour <envoy-pod> 19000:19000
# Visit http://localhost:19000/config_dump
```

### Issue: TLS certificate not working

```bash
# Check cert-manager certificate
kubectl get certificate
kubectl describe certificate app-example-com-tls

# Check ACME challenge
kubectl get challenges

# Check cert-manager logs
kubectl logs -n cert-manager deployment/cert-manager
```

---

## Comparison: Contour vs Nginx Ingress

| Feature | Nginx Ingress | **Contour** |
|---------|---------------|-------------|
| **Config reload** | Restart Nginx (brief downtime) | **Hot reload** (zero downtime) |
| **Configuration** | Annotations (untyped) | **HTTPProxy CRD** (typed) |
| **Multi-tenancy** | Limited | **Namespace delegation** |
| **gRPC** | Limited | **Native support** |
| **WebSockets** | ✅ | ✅ |
| **Canary** | Via annotations | **Native weight-based** |
| **Observability** | Basic | **Rich Envoy metrics** |
| **Performance** | Good | **Excellent** (Envoy) |

---

## Summary

You now have Contour configured with:

✅ **Envoy proxy** - modern, high-performance L7 load balancer
✅ **HTTPProxy CRD** - type-safe routing configuration
✅ **TLS automation** - Let's Encrypt integration via cert-manager
✅ **Advanced routing** - path, header, weight-based
✅ **Zero-downtime updates** - hot reload configuration
✅ **Canary deployments** - weighted traffic splitting
✅ **Blue-green deployments** - instant cutover
✅ **Rate limiting** - protect against abuse
✅ **Prometheus metrics** - full observability

**Benefits for your PHP applications:**
- 🚀 Zero-downtime deployments (no Nginx reload)
- 📝 Clean, type-safe configuration (no annotation hell)
- 🔄 Easy canary and blue-green deployments
- 📊 Rich observability with Envoy metrics
- ⚡ Better HTTP/2 and gRPC support

**Next steps:**
- Deploy your PHP applications with HTTPProxy
- Set up cert-manager for automatic TLS
- Configure rate limiting and timeouts
- Monitor with Prometheus and Grafana
- Implement canary deployment workflow
