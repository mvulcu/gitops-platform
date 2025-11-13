# Monitoring & Observability Architecture

## 📐 System Architecture Overview

This document provides a technical overview of the monitoring and observability stack deployed in the GitOps platform.

```
┌─────────────────────────────────────────────────────────────────┐
│                     Kubernetes Cluster                          │
│                                                                 │
│  ┌──────────────┐         ┌──────────────┐                    │
│  │  Application │         │ Ingress-NGINX│                    │
│  │  Pods        │────────▶│  Controller  │◀────── External    │
│  │ (lingua-app) │         │              │        Traffic     │
│  └──────────────┘         └──────────────┘                    │
│         │                                                       │
│         │ logs                                                  │
│         ▼                                                       │
│  ┌──────────────────────────────────────────────────┐         │
│  │            Logging Layer (namespace: logging)    │         │
│  │                                                  │         │
│  │  ┌──────────┐          ┌─────────────┐         │         │
│  │  │ Promtail │─────────▶│    Loki     │         │         │
│  │  │DaemonSet │  push    │ SingleBinary│         │         │
│  │  └──────────┘  logs    │    + PVC    │         │         │
│  │                         └─────────────┘         │         │
│  │                               │                  │         │
│  │                               │ queries          │         │
│  └───────────────────────────────┼──────────────────┘         │
│                                  │                             │
│  ┌───────────────────────────────┼─────────────────────────┐  │
│  │       Monitoring Layer (namespace: monitoring)          │  │
│  │                                │                         │  │
│  │  ┌────────────┐    ┌──────────▼────────┐               │  │
│  │  │ Prometheus │    │     Grafana       │               │  │
│  │  │            │◀───│  Visualization    │               │  │
│  │  │  Metrics   │    │                   │               │  │
│  │  │  Storage   │    │  Datasources:     │               │  │
│  │  │            │    │  - Prometheus     │               │  │
│  │  │  + PVC     │    │  - Loki           │               │  │
│  │  └────────────┘    │                   │               │  │
│  │         ▲          │  + PVC            │               │  │
│  │         │          └───────────────────┘               │  │
│  │         │ scrape                │                       │  │
│  │         │                       │ https (Ingress)       │  │
│  │  ┌──────┴──────────────┐       │                       │  │
│  │  │  Service Monitors:  │       └───────────────────────┼─▶│
│  │  │  - Node Exporter    │                           External│
│  │  │  - Kube State       │                           Access  │
│  │  │  - Kubelet          │                                   │
│  │  │  - Loki             │                grafana.cachefly   │
│  │  │  - Promtail         │                      .site        │
│  │  └─────────────────────┘                                   │
│  │                                                             │
│  │  ┌──────────────────┐                                     │  │
│  │  │  Alertmanager    │                                     │  │
│  │  │  (not configured)│                                     │  │
│  │  └──────────────────┘                                     │  │
│  └─────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘

                         ┌─────────────┐
                         │   Flux CD   │
                         │  GitOps     │
                         │  Operator   │
                         └─────────────┘
                               ▲
                               │ sync
                               │
                         ┌─────────────┐
                         │ GitHub Repo │
                         │ gitops-     │
                         │ platform    │
                         └─────────────┘
```

---

## 🏗️ Component Details

### 1. Logging Stack (namespace: `logging`)

#### Loki
- **Version**: 6.22.0 (via Grafana Helm chart)
- **Deployment Mode**: SingleBinary (optimized for single-node VPS)
- **Storage**: 10Gi PVC (filesystem-based)
- **Schema**: TSDB v13 (from 2024-01-01)
- **Replication Factor**: 1 (single instance)
- **Resources**:
  - Requests: 100m CPU, 256Mi memory
  - Limits: 500m CPU, 1Gi memory

**Configuration Highlights:**
```yaml
loki:
  auth_enabled: false
  storage:
    type: 'filesystem'
  schemaConfig:
    configs:
      - from: "2024-01-01"
        store: tsdb
        object_store: filesystem
        schema: v13
```

**Gateway Configuration:**
- Enabled: Yes
- Replicas: 1
- Service Type: ClusterIP
- Endpoint: `http://loki-gateway.logging.svc.cluster.local`
- Resources: 50m CPU / 64Mi memory (requests)

**Monitoring Integration:**
- ServiceMonitor enabled for Prometheus scraping
- Label: `release: kube-prometheus-stack`

**File Location**: `/mnt/d/Projects/gitops-platform/apps/infra/logging/loki-helmrelease.yaml`

---

#### Promtail
- **Version**: 6.16.6 (via Grafana Helm chart)
- **Deployment**: DaemonSet (runs on every node)
- **Client URL**: `http://loki-gateway.logging.svc.cluster.local/loki/api/v1/push`
- **Resources**:
  - Requests: 50m CPU, 64Mi memory
  - Limits: 200m CPU, 128Mi memory

**Pipeline Configuration:**
```yaml
pipelineStages:
  - cri: {}                    # Parse CRI format logs
  - labeldrop:                 # Drop unnecessary labels
      - filename
  - timestamp:                 # Extract timestamp
      source: time
      format: RFC3339Nano
```

**Tolerations**: Configured to run on all nodes (including control plane)

**Dependencies**: `dependsOn: loki` (ensures Loki is ready first)

**File Location**: `/mnt/d/Projects/gitops-platform/apps/infra/logging/promtail-helmrelease.yaml`

---

### 2. Monitoring Stack (namespace: `monitoring`)

#### Prometheus (via kube-prometheus-stack)
- **Chart Version**: 65.5.1
- **Retention Period**: 7 days
- **Datasource**: Available to Grafana at `http://kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090`
- **Scrape Interval**: 30s (configured in datasource)

**Key Components Included:**
- Prometheus Operator
- Node Exporter (DaemonSet)
- Kube State Metrics
- Prometheus Alert Manager
- Grafana

**ServiceMonitor Targets:**
- Node Exporter (system metrics)
- Kube State Metrics (Kubernetes object state)
- Kubelet cAdvisor (container metrics)
- Loki (log system metrics)
- Promtail (log collector metrics)

**File Location**: `/mnt/d/Projects/gitops-platform/apps/infra/monitoring/helmrelease.yaml`

---

#### Grafana
- **Admin Credentials**: admin/admin (should be changed on first login)
- **Persistence**: 1Gi PVC enabled
- **Ingress**:
  - Enabled: Yes
  - Hostname: `grafana.cachefly.site`
  - TLS: Let's Encrypt (cert-manager)
  - Ingress Class: nginx

**Datasources:**

1. **Prometheus** (primary)
   ```yaml
   - name: Prometheus
     type: prometheus
     uid: prometheus
     url: http://kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090
     access: proxy
     isDefault: true
   ```

2. **Loki** (additional)
   ```yaml
   - name: Loki
     type: loki
     uid: loki
     url: http://loki-gateway.logging.svc.cluster.local
     access: proxy
     jsonData:
       maxLines: 5000
       timeout: 60
   ```

**Dashboard Auto-loading Configuration:**
```yaml
sidecar:
  dashboards:
    enabled: true
    label: grafana_dashboard
    labelValue: "1"
    folder: /tmp/dashboards
    searchNamespace: monitoring
    provider:
      foldersFromFilesStructure: true
```

**How it works:**
1. Grafana runs a sidecar container alongside main container
2. Sidecar watches for ConfigMaps with label `grafana_dashboard: "1"`
3. When found, sidecar extracts JSON and places it in `/tmp/dashboards`
4. Grafana auto-loads dashboards from this directory
5. Changes to ConfigMaps trigger automatic dashboard updates

---

### 3. Custom Dashboards

#### Dashboard 1: Kubernetes Logs (Loki)
- **UID**: `kubernetes-logs-loki`
- **Datasource**: Loki
- **ConfigMap**: `grafana-dashboard-loki-logs`
- **Auto-refresh**: 10 seconds

**Panels (6 total):**
1. **Log Rate** (Time Series)
   - Query: `sum(rate({namespace=~"$namespace"}[$__interval]))`
   - Shows logs per second

2. **Error Count** (Gauge)
   - Query: `sum(count_over_time({namespace=~"$namespace"} |~ "(?i)error|fail|exception"[5m]))`
   - Thresholds: green <10, yellow 10-50, red >50

3. **Logs by Namespace** (Pie Chart)
   - Query: `sum by (namespace) (count_over_time({namespace=~"$namespace"}[1h]))`
   - Distribution visualization

4. **Top 10 Pods by Log Volume** (Table)
   - Query: `topk(10, sum by (pod) (count_over_time({namespace=~"$namespace"}[1h])))`

5. **Live Logs** (Logs Panel)
   - Query: `{namespace=~"$namespace"} |~ "(?i)$search"`
   - Real-time log streaming

6. **Error Logs** (Logs Panel)
   - Query: `{namespace=~"$namespace"} |~ "(?i)error|fail|exception|fatal|panic"`
   - Filtered error view

**Variables:**
- `$namespace`: Multi-select, query from Loki labels, includeAll: true
- `$search`: Textbox for live filtering

**File Locations**:
- JSON: `/mnt/d/Projects/gitops-platform/apps/infra/monitoring/dashboards/loki-logs-dashboard.json`
- ConfigMap: `/mnt/d/Projects/gitops-platform/apps/infra/monitoring/dashboards/loki-logs-configmap.yaml`

---

#### Dashboard 2: LinguaLink Application
- **UID**: `lingua-app-dashboard`
- **Datasources**: Prometheus (primary), Loki (logs panel)
- **ConfigMap**: `grafana-dashboard-lingua-app`
- **Auto-refresh**: 10 seconds
- **Time Range**: Last 1 hour (default)

**Panels (9 total):**

1. **Running Pods** (Gauge)
   - Query: `sum(kube_pod_status_phase{namespace="lingua-app",phase="Running"})`
   - Thresholds: red 0, yellow 1, green ≥2
   - Size: 4h × 6w

2. **HPA Current Replicas** (Gauge)
   - Query: `kube_horizontalpodautoscaler_status_current_replicas{namespace="lingua-app"}`
   - Thresholds: green <2, yellow 2-4, red >4
   - Size: 4h × 6w

3. **Avg CPU Usage** (Gauge)
   - Query: `avg(rate(container_cpu_usage_seconds_total{namespace="lingua-app",pod=~"lingua-app-.*",container!=""}[5m])) * 100`
   - Unit: percent
   - Thresholds: green <60%, yellow 60-80%, red >80%
   - Size: 4h × 6w

4. **Avg Memory Usage** (Gauge)
   - Query: `avg(container_memory_working_set_bytes{...} / container_spec_memory_limit_bytes{...}) * 100`
   - Unit: percent
   - Thresholds: green <70%, yellow 70-85%, red >85%
   - Size: 4h × 6w

5. **CPU Usage by Pod** (Time Series)
   - Query: `rate(container_cpu_usage_seconds_total{namespace="lingua-app",pod=~"lingua-app-.*",container!=""}[5m])`
   - Legend: `{{pod}}`
   - Unit: percentunit
   - Shows: mean, max in legend
   - Size: 8h × 12w

6. **Memory Usage by Pod** (Time Series)
   - Query: `container_memory_working_set_bytes{namespace="lingua-app",pod=~"lingua-app-.*",container!=""}`
   - Legend: `{{pod}}`
   - Unit: bytes
   - Shows: mean, max in legend
   - Size: 8h × 12w

7. **HPA Scaling Activity** (Time Series)
   - Queries (4):
     - Current: `kube_horizontalpodautoscaler_status_current_replicas{namespace="lingua-app"}`
     - Desired: `kube_horizontalpodautoscaler_status_desired_replicas{namespace="lingua-app"}`
     - Min: `kube_horizontalpodautoscaler_spec_min_replicas{namespace="lingua-app"}`
     - Max: `kube_horizontalpodautoscaler_spec_max_replicas{namespace="lingua-app"}`
   - Threshold: red line at 5 replicas
   - Size: 8h × 12w

8. **Pod Restarts** (Time Series)
   - Query: `sum(kube_pod_container_status_restarts_total{namespace="lingua-app",pod=~"lingua-app-.*"}) by (pod)`
   - Legend: `{{pod}}`
   - Size: 8h × 12w

9. **Application Logs** (Logs Panel)
   - Datasource: Loki
   - Query: `{namespace="lingua-app"}`
   - Options: enableLogDetails, showTime, sortOrder: Descending
   - Size: 10h × 24w

**File Locations**:
- JSON: `/mnt/d/Projects/gitops-platform/apps/infra/monitoring/dashboards/lingua-app-dashboard.json`
- ConfigMap: `/mnt/d/Projects/gitops-platform/apps/infra/monitoring/dashboards/lingua-app-configmap.yaml`

---

#### Dashboard 3: Application Logs (Simple)
- **UID**: `app-logs-simple`
- **Datasource**: Loki
- **ConfigMap**: `grafana-dashboard-app-logs`
- **Auto-refresh**: 10 seconds
- **Time Range**: Last 6 hours (default)

**Panels (1 total):**

1. **LinguaLink Application Logs** (Logs Panel)
   - Query: `{namespace="lingua-app"}`
   - Full-width display (20h × 24w)
   - Simplified view for focused log analysis

**File Locations**:
- JSON: `/mnt/d/Projects/gitops-platform/apps/infra/monitoring/dashboards/app-logs-simple.json`
- ConfigMap: `/mnt/d/Projects/gitops-platform/apps/infra/monitoring/dashboards/app-logs-configmap.yaml`

---

#### Community Dashboards (Auto-imported)

**Pre-configured in HelmRelease:**
```yaml
dashboards:
  kubernetes:
    k8s-cluster:
      gnetId: 7249        # Kubernetes Cluster Monitoring
      revision: 1
      datasource: Prometheus
    k8s-deployments:
      gnetId: 8588        # Kubernetes Deployments/StatefulSets/DaemonSets
      revision: 1
      datasource: Prometheus
    node-exporter:
      gnetId: 1860        # Node Exporter Full
      revision: 31
      datasource: Prometheus
    loki-metrics:
      gnetId: 13639       # Loki Dashboard
      revision: 2
      datasource: Prometheus
```

---

## 🔄 Data Flow

### Metrics Collection Flow
```
Kubernetes Objects
      │
      │ expose metrics
      ▼
┌─────────────────┐
│ ServiceMonitors │ (CRDs defined by Prometheus Operator)
└────────┬────────┘
         │ configure
         ▼
  ┌─────────────┐
  │ Prometheus  │ scrapes metrics every 30s
  │             │ stores in TSDB (7 days retention)
  └──────┬──────┘
         │ query via PromQL
         ▼
   ┌──────────┐
   │ Grafana  │ visualizes in dashboards
   └──────────┘
```

### Log Collection Flow
```
Container Logs
      │
      │ write to
      ▼
  /var/log/pods/*
      │
      │ read by
      ▼
┌──────────────┐
│  Promtail    │ (DaemonSet on each node)
│              │ parses CRI format
└──────┬───────┘
       │ push logs
       ▼
  ┌────────────┐
  │ Loki       │ stores in chunks (filesystem)
  │ Gateway    │ indexes with labels
  └─────┬──────┘
        │ query via LogQL
        ▼
   ┌──────────┐
   │ Grafana  │ displays in log panels
   └──────────┘
```

### Dashboard Provisioning Flow
```
Git Repository
      │
      │ commit ConfigMap YAML
      ▼
  ┌─────────┐
  │ Flux CD │ reconciles every 1-10 minutes
  └────┬────┘
       │ applies to cluster
       ▼
┌──────────────────┐
│ ConfigMap        │ with label: grafana_dashboard="1"
│ (namespace:      │
│  monitoring)     │
└────────┬─────────┘
         │ watched by
         ▼
┌─────────────────┐
│ Grafana Sidecar │ k8s-sidecar container
│                 │ polls API for ConfigMaps
└────────┬────────┘
         │ extracts JSON
         │ writes to
         ▼
  /tmp/dashboards/
         │
         │ auto-loaded by
         ▼
   ┌──────────┐
   │ Grafana  │ dashboard appears in UI
   └──────────┘
```

---

## 📦 Storage Architecture

### Persistent Volumes

| Component | PVC Name | Size | StorageClass | Purpose |
|-----------|----------|------|--------------|---------|
| Loki | loki-loki-0 | 10Gi | Default | Log chunk storage |
| Prometheus | prometheus-kube-prometheus-stack-prometheus-0 | Varies | Default | Metrics TSDB |
| Grafana | kube-prometheus-stack-grafana | 1Gi | Default | Dashboard definitions & settings |

### Retention Policies

| System | Retention | Configuration Location |
|--------|-----------|------------------------|
| Prometheus | 7 days | `helmrelease.yaml:29` (prometheus.prometheusSpec.retention) |
| Loki | Based on disk | Implicitly by 10Gi PVC size |
| Grafana | Persistent | Dashboard changes saved to PVC |

---

## 🔌 Network Architecture

### Internal Service Endpoints

```yaml
# Prometheus
kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090

# Loki Gateway
loki-gateway.logging.svc.cluster.local:80

# Loki Push Endpoint (for Promtail)
loki-gateway.logging.svc.cluster.local/loki/api/v1/push

# Grafana
kube-prometheus-stack-grafana.monitoring.svc.cluster.local:80
```

### External Access

```yaml
# Grafana Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kube-prometheus-stack-grafana
  namespace: monitoring
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - grafana.cachefly.site
      secretName: grafana-tls
  rules:
    - host: grafana.cachefly.site
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kube-prometheus-stack-grafana
                port:
                  number: 80
```

**DNS**: `grafana.cachefly.site` → Cloudflare DNS → NGINX Ingress → Grafana Service

---

## 🎛️ Configuration Management

### Helm Repositories

```yaml
# Prometheus Community (kube-prometheus-stack)
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: prometheus-community
  namespace: flux-system
spec:
  interval: 10m
  url: https://prometheus-community.github.io/helm-charts

# Grafana (Loki, Promtail)
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: grafana
  namespace: flux-system
spec:
  interval: 10m
  url: https://grafana.github.io/helm-charts
```

### GitOps Structure

```
apps/infra/
├── logging/
│   ├── namespace.yaml              # logging namespace
│   ├── helm-repository.yaml        # Grafana Helm repo
│   ├── loki-helmrelease.yaml       # Loki deployment
│   ├── promtail-helmrelease.yaml   # Promtail deployment
│   └── kustomization.yaml          # Kustomize manifest
│
└── monitoring/
    ├── namespace.yaml              # monitoring namespace
    ├── helm-repository.yaml        # Prometheus Community Helm repo
    ├── helmrelease.yaml            # kube-prometheus-stack deployment
    ├── kustomization.yaml          # Kustomize manifest
    └── dashboards/
        ├── kustomization.yaml
        ├── loki-logs-dashboard.json
        ├── loki-logs-configmap.yaml
        ├── lingua-app-dashboard.json
        ├── lingua-app-configmap.yaml
        ├── app-logs-simple.json
        └── app-logs-configmap.yaml
```

### Resource Labels & Selectors

**ServiceMonitor Selection:**
```yaml
# In Prometheus Operator
serviceMonitorSelector:
  matchLabels:
    release: kube-prometheus-stack
```

**Dashboard Auto-loading:**
```yaml
# ConfigMap label
labels:
  grafana_dashboard: "1"

# Grafana sidecar selector
sidecar:
  dashboards:
    label: grafana_dashboard
    labelValue: "1"
```

---

## 🔧 Resource Allocation

### Per-Component Resource Requests/Limits

| Component | Namespace | Replicas | CPU Request | CPU Limit | Memory Request | Memory Limit |
|-----------|-----------|----------|-------------|-----------|----------------|--------------|
| Loki (single-binary) | logging | 1 | 100m | 500m | 256Mi | 1Gi |
| Loki Gateway | logging | 1 | 50m | 200m | 64Mi | 128Mi |
| Promtail | logging | DaemonSet | 50m | 200m | 64Mi | 128Mi |
| Prometheus | monitoring | 1 | ~500m* | ~2000m* | ~2Gi* | ~4Gi* |
| Grafana | monitoring | 1 | ~100m* | ~200m* | ~128Mi* | ~256Mi* |
| Alertmanager | monitoring | 1 | ~50m* | ~100m* | ~64Mi* | ~128Mi* |

*Default values from kube-prometheus-stack chart (not explicitly set)

### Total Cluster Resource Usage (Estimated)

- **CPU**: ~1.5-2 cores
- **Memory**: ~4-6 GB
- **Storage**: ~12 GB (10Gi Loki + 1Gi Grafana + Prometheus TSDB)

---

## 🔍 Query Examples by Use Case

### Debugging Application Issues

**Find when pod restarted:**
```promql
changes(kube_pod_container_status_restarts_total{namespace="lingua-app"}[1h]) > 0
```

**Check logs around restart time:**
```logql
{namespace="lingua-app", pod="lingua-app-xyz"} |= ""
```

**CPU throttling detection:**
```promql
rate(container_cpu_cfs_throttled_seconds_total{namespace="lingua-app"}[5m]) > 0
```

### Capacity Planning

**Forecast pod count:**
```promql
predict_linear(kube_horizontalpodautoscaler_status_current_replicas{namespace="lingua-app"}[1h], 3600)
```

**Memory growth rate:**
```promql
deriv(container_memory_working_set_bytes{namespace="lingua-app"}[1h])
```

### Security Monitoring

**Failed authentication attempts:**
```logql
{namespace="lingua-app"} |~ "(?i)authentication failed|unauthorized"
```

**Suspicious activity:**
```logql
{namespace=~".+"} |~ "(?i)exec|attack|injection|exploit"
```

---

## 🚀 Performance Optimization

### Current Optimizations

1. **Loki SingleBinary Mode**
   - Reduces complexity and resource overhead
   - Suitable for small-to-medium deployments
   - Avoids distributed components (read/write/backend)

2. **Caching Disabled**
   ```yaml
   chunksCache:
     enabled: false
   resultsCache:
     enabled: false
   ```
   - Reduces memory footprint
   - Acceptable for low-to-medium query load

3. **Prometheus Retention**
   - 7 days = balance between history and storage
   - Prevents unbounded TSDB growth

4. **Dashboard Auto-refresh**
   - 10 seconds for real-time monitoring
   - Can be increased to reduce query load

### Potential Improvements

1. **Recording Rules** (not yet configured)
   ```yaml
   apiVersion: monitoring.coreos.com/v1
   kind: PrometheusRule
   metadata:
     name: lingua-app-rules
   spec:
     groups:
       - name: lingua-app
         interval: 30s
         rules:
           - record: lingua_app:cpu_usage:rate5m
             expr: rate(container_cpu_usage_seconds_total{namespace="lingua-app"}[5m])
   ```

2. **Remote Storage** (for long-term retention)
   - S3-compatible storage for Loki chunks
   - Thanos or Cortex for Prometheus

3. **Alert Rules** (basic defaults only)
   - Custom alerts for application-specific SLOs
   - Integration with PagerDuty/Slack

---

## 🔐 Security Considerations

### Current Security Posture

**Authentication:**
- Grafana: Basic auth (admin/admin - should be changed)
- Prometheus: Not exposed externally
- Loki: No authentication (`auth_enabled: false`)

**Network Policies:**
- Not explicitly configured
- Default Kubernetes network policy (all traffic allowed)

**Encryption:**
- TLS: Enabled for Grafana ingress (Let's Encrypt)
- In-transit: Internal traffic unencrypted (ClusterIP services)
- At-rest: PVCs not encrypted (depends on storage class)

### Security Recommendations

1. **Enable Loki Authentication**
   ```yaml
   loki:
     auth_enabled: true
   ```

2. **Add Network Policies**
   ```yaml
   # Allow only Grafana → Prometheus
   # Allow only Promtail → Loki
   # Deny all other cross-namespace traffic
   ```

3. **RBAC for Grafana**
   - Configure LDAP/OAuth
   - Remove admin user or enforce strong password

4. **Secrets Management**
   - Use Sealed Secrets or External Secrets Operator
   - Don't commit credentials to Git

---

## 📈 Scaling Considerations

### Horizontal Scaling

**Loki:**
- Current: SingleBinary mode (1 replica)
- Scale to: Microservices mode (read/write/backend components)
- Trigger: >100GB logs/day or >500 queries/minute

**Prometheus:**
- Current: Single instance
- Scale to: Prometheus federation or Thanos
- Trigger: >10 million active time series

**Grafana:**
- Current: 1 replica (stateful)
- Scale to: Multiple replicas with shared database
- Trigger: >100 concurrent users

### Vertical Scaling

**Memory:**
- Prometheus: Increase as time series grow
- Loki: Increase for query performance
- Grafana: Usually sufficient at default

**Storage:**
- Loki: Increase PVC as log volume grows
- Prometheus: Increase PVC or reduce retention

---

## 🧪 Testing & Validation

### Health Checks

```bash
# Check all monitoring pods
kubectl get pods -n monitoring
kubectl get pods -n logging

# Verify Prometheus targets
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
# Open: http://localhost:9090/targets
# All targets should show "UP"

# Verify Loki receiving logs
kubectl logs -n logging -l app.kubernetes.io/name=loki --tail=20
# Should see: POST /loki/api/v1/push

# Test Grafana datasources
# Grafana → Configuration → Data sources → Test
```

### Sample Queries for Validation

```promql
# Should return >0
up{job="kube-state-metrics"}

# Should show node CPU
rate(node_cpu_seconds_total[5m])
```

```logql
# Should return logs
{namespace="logging"}
```

---

## 📊 Monitoring the Monitoring Stack

### Meta-Monitoring Queries

**Prometheus self-monitoring:**
```promql
# Prometheus memory usage
process_resident_memory_bytes{job="prometheus"} / 1024 / 1024 / 1024

# Scrape duration
scrape_duration_seconds{job="kube-state-metrics"}

# Failed scrapes
up == 0
```

**Loki self-monitoring:**
```promql
# Ingestion rate
rate(loki_distributor_bytes_received_total[5m])

# Query latency
histogram_quantile(0.99, rate(loki_request_duration_seconds_bucket[5m]))
```

**Grafana dashboards:**
- Loki Metrics (Dashboard ID 13639) - monitors Loki performance
- Prometheus Stats - built-in dashboard

---

## 🎓 Key Learnings & Best Practices

### What Works Well

1. **GitOps Dashboard Management**
   - Version control for dashboards
   - Easy rollback
   - No manual import needed

2. **SingleBinary Loki**
   - Perfect for VPS/single-node setups
   - Lower resource usage
   - Simpler to manage

3. **kube-prometheus-stack**
   - Batteries included
   - Well-maintained
   - Great defaults

### Lessons Learned

1. **ConfigMap Line Endings**
   - Issue: Windows CRLF in JSON caused parsing errors
   - Solution: Fixed ConfigMap formatting

2. **Namespace Targeting**
   - Important: Grafana sidecar only searches `monitoring` namespace
   - ConfigMaps must be in correct namespace

3. **Dashboard UIDs**
   - Always set explicit UIDs to avoid duplicates
   - Makes URLs predictable

---

## 🔗 Related Documentation

- [MONITORING-GUIDE.md](./MONITORING-GUIDE.md) - User guide for accessing and using monitoring
- [DASHBOARDS.md](./DASHBOARDS.md) - Dashboard catalog and customization guide
- [VERIFICATION.md](./VERIFICATION.md) - Deployment verification steps
- [README.md](./README.md) - Project overview

---

## 📝 Maintenance Tasks

### Weekly
- Check disk usage on Loki/Prometheus PVCs
- Review Grafana for dashboard updates
- Verify all Prometheus targets are UP

### Monthly
- Review and tune alert thresholds
- Update Helm chart versions
- Backup Grafana dashboards (export JSON)
- Review log retention needs

### Quarterly
- Audit Grafana users and permissions
- Review and optimize expensive queries
- Capacity planning review
- Update documentation

---

**Document Version**: 1.0
**Last Updated**: 2025-01-13
**Maintained By**: Maria Vulcu
**Environment**: Production (vps-c9e6ac23)

---

*This architecture document provides a comprehensive technical reference for the monitoring and observability infrastructure deployed via GitOps principles.*
