# Monitoring and Observability Guide

## 📊 Overview

This guide provides comprehensive instructions for accessing, using, and understanding the monitoring and observability stack deployed on the LinguaLink production infrastructure.

---

## 🎯 Quick Access

### Production URLs

| Service | URL | Purpose |
|---------|-----|---------|
| **Application** | https://lingua.cachefly.site | Production Next.js application |
| **Grafana** | https://grafana.cachefly.site | Metrics visualization and dashboards |
| **Prometheus** | Port-forward required | Metrics database and queries |
| **Alertmanager** | Port-forward required | Alert management |
| **Loki** | Via Grafana | Log aggregation and queries |

### Credentials

**Grafana:**
- Username: `admin`
- Password: Changed during first login (not default)

---

## 📈 Grafana Dashboards

### Accessing Grafana

1. Navigate to https://grafana.cachefly.site
2. Log in with admin credentials
3. Dashboard home appears automatically

### Pre-installed Dashboards

The kube-prometheus-stack Helm chart includes comprehensive dashboards out of the box:

#### 1. **Kubernetes / Compute Resources / Cluster**

**Path**: Dashboards → Kubernetes / Compute Resources / Cluster

**Metrics Displayed:**
- Cluster-wide CPU usage (total and per-node)
- Cluster-wide memory usage
- Network I/O
- Pod count and distribution
- Resource requests vs limits vs actual usage

**Use Cases:**
- Capacity planning
- Identifying resource-hungry workloads
- Detecting cluster-wide issues

**Key Panels:**
```
┌─────────────────────────────────────────┐
│ CPU Usage (% of allocatable)           │
│ ▓▓▓▓▓▓▓▓░░░░░░░░░░░░░░░░░░ 35%        │
├─────────────────────────────────────────┤
│ Memory Usage (% of allocatable)        │
│ ▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░░░░░ 46%         │
├─────────────────────────────────────────┤
│ Pod Count: 22 / 110 (20%)              │
└─────────────────────────────────────────┘
```

#### 2. **Kubernetes / Compute Resources / Namespace (Workloads)**

**Path**: Dashboards → Kubernetes / Compute Resources / Namespace

**Metrics Displayed:**
- Per-namespace resource consumption
- CPU and memory usage by workload
- Pod count per namespace
- Network traffic by namespace

**Use Cases:**
- Multi-tenant resource monitoring
- Namespace quota enforcement
- Cost allocation by team/application

**Example View:**
```
Namespace: lingua-app
┌─────────────────────────────────────────┐
│ Deployment: lingua-app                  │
│ Pods: 2/2                               │
│ CPU: 50m / 200m (25%)                   │
│ Memory: 150Mi / 256Mi (58%)             │
└─────────────────────────────────────────┘
```

#### 3. **Kubernetes / Compute Resources / Pod**

**Path**: Dashboards → Kubernetes / Compute Resources / Pod

**Metrics Displayed:**
- Individual pod resource usage
- CPU throttling events
- Memory working set
- Network receive/transmit rates
- Container restart count

**Use Cases:**
- Debugging specific pod issues
- Identifying resource constraints
- Container-level troubleshooting

#### 4. **Node Exporter / Nodes**

**Path**: Dashboards → Node Exporter / Nodes

**Metrics Displayed:**
- System load average (1m, 5m, 15m)
- CPU breakdown (user, system, iowait, idle)
- Memory breakdown (used, cached, buffered, free)
- Disk I/O, space usage, inodes
- Network interface statistics
- File descriptor usage

**Use Cases:**
- Node health monitoring
- Disk space alerts
- Network troubleshooting
- System performance analysis

**Critical Metrics:**
```
┌─────────────────────────────────────────┐
│ Node: vps-c9e6ac23                      │
│ Uptime: 7 days                          │
│ Load Average: 0.82 / 0.95 / 0.88        │
│ CPU: 35% | Memory: 46% | Disk: 22%     │
└─────────────────────────────────────────┘
```

#### 5. **Kubernetes / Networking / Pod**

**Path**: Dashboards → Kubernetes / Networking / Pod

**Metrics Displayed:**
- Network bandwidth usage per pod
- Packets transmitted/received
- Errors and drops
- Bandwidth saturation

**Use Cases:**
- Network performance troubleshooting
- Identifying chatty services
- Bandwidth utilization tracking

---

## 🔍 Prometheus Queries

### Accessing Prometheus

Since Prometheus is not exposed via Ingress, use port-forwarding:

```bash
# On VPS or with proper kubeconfig
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
```

Then open: http://localhost:9090

### Useful PromQL Queries

#### Application Metrics

**1. Pod CPU Usage (Current)**
```promql
rate(container_cpu_usage_seconds_total{namespace="lingua-app",pod=~"lingua-app-.*"}[5m])
```

**2. Pod Memory Usage (Current)**
```promql
container_memory_working_set_bytes{namespace="lingua-app",pod=~"lingua-app-.*"}
```

**3. Pod Restart Count**
```promql
kube_pod_container_status_restarts_total{namespace="lingua-app"}
```

**4. Pod Availability**
```promql
sum(kube_pod_status_phase{namespace="lingua-app",phase="Running"}) / sum(kube_pod_status_phase{namespace="lingua-app"}) * 100
```

#### Cluster Health

**5. Node CPU Usage**
```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**6. Node Memory Usage**
```promql
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```

**7. Disk Space Usage**
```promql
100 - ((node_filesystem_avail_bytes{mountpoint="/",fstype!="rootfs"} * 100) / node_filesystem_size_bytes{mountpoint="/",fstype!="rootfs"})
```

**8. HPA Current Replicas**
```promql
kube_horizontalpodautoscaler_status_current_replicas{namespace="lingua-app"}
```

#### Kubernetes Resources

**9. Pods per Namespace**
```promql
count by (namespace) (kube_pod_info)
```

**10. Container Restarts (Last Hour)**
```promql
increase(kube_pod_container_status_restarts_total[1h]) > 0
```

**11. Failed Pods**
```promql
kube_pod_status_phase{phase="Failed"} == 1
```

**12. Pending Pods**
```promql
kube_pod_status_phase{phase="Pending"} == 1
```

---

## 📝 Loki Log Queries

### Accessing Loki via Grafana

1. Open https://grafana.cachefly.site
2. Navigate to **Explore** (compass icon in left sidebar)
3. Select **Loki** from datasource dropdown (top)

### LogQL Query Examples

#### Basic Log Retrieval

**1. All logs from lingua-app**
```logql
{namespace="lingua-app"}
```

**2. Last 5 minutes of logs**
```logql
{namespace="lingua-app"} |= "" [5m]
```

**3. Logs from specific pod**
```logql
{namespace="lingua-app", pod=~"lingua-app-.*-k4m85"}
```

#### Filtered Searches

**4. Error logs only**
```logql
{namespace="lingua-app"} |= "error"
```

**5. Case-insensitive error search**
```logql
{namespace="lingua-app"} |~ "(?i)error"
```

**6. Exclude certain patterns**
```logql
{namespace="lingua-app"} != "health check"
```

**7. Multiple conditions (AND)**
```logql
{namespace="lingua-app"} |= "error" |= "database"
```

**8. Multiple conditions (OR)**
```logql
{namespace="lingua-app"} |~ "error|warning|critical"
```

#### Advanced Queries

**9. Extract and count HTTP status codes**
```logql
sum by (status) (
  count_over_time(
    {namespace="lingua-app"}
    | regexp `status=(?P<status>\\d{3})`
    [1h]
  )
)
```

**10. Error rate (errors per minute)**
```logql
sum(rate({namespace="lingua-app"} |= "error" [1m]))
```

**11. Top 10 error messages**
```logql
topk(10,
  sum by (msg) (
    count_over_time(
      {namespace="lingua-app"}
      | json
      | __error__=""
      | level="error"
      [1h]
    )
  )
)
```

#### Infrastructure Logs

**12. All cert-manager logs (TLS troubleshooting)**
```logql
{namespace="cert-manager"}
```

**13. Flux reconciliation logs**
```logql
{namespace="flux-system"} |~ "reconcil"
```

**14. Ingress controller logs**
```logql
{namespace="ingress-nginx"}
```

**15. All error logs cluster-wide**
```logql
{namespace=~".+"} |~ "(?i)error|fail|exception"
```

### Log Visualization Options

- **Logs**: Raw log lines with timestamps
- **Table**: Structured view with columns
- **Time series**: Count of logs over time (useful with `count_over_time`)
- **Stats**: Aggregate statistics

---

## 🚨 Alertmanager

### Accessing Alertmanager

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-alertmanager 9093:9093
```

Then open: http://localhost:9093

### Pre-configured Alerts

The kube-prometheus-stack includes default alerts:

| Alert Name | Severity | Condition | Description |
|------------|----------|-----------|-------------|
| **KubePodCrashLooping** | Warning | Pod restarting frequently | Detects crash loops |
| **KubePodNotReady** | Warning | Pod not ready > 15min | Pod stuck in non-ready state |
| **KubeDeploymentReplicasMismatch** | Warning | Desired ≠ Current replicas | Deployment scaling issues |
| **KubeNodeNotReady** | Critical | Node not ready > 5min | Node failure |
| **KubeMemoryOvercommit** | Warning | Memory requests > capacity | Resource overcommitment |
| **PrometheusScrapeError** | Warning | Scrape failing | Metrics collection issues |
| **NodeDiskRunningFull** | Warning | Disk > 80% | Disk space alert |
| **NodeMemoryPressure** | Warning | Memory > 80% | Memory pressure |

### Alert Configuration

Alerts are defined in PrometheusRule CRDs. To view:

```bash
kubectl get prometheusrules -n monitoring
kubectl get prometheusrules -n monitoring -o yaml
```

### Silencing Alerts

Temporary alert silencing (useful during maintenance):

1. Open Alertmanager UI
2. Click **Silences**
3. Click **New Silence**
4. Fill in:
   - **Matchers**: `alertname="KubePodNotReady"`
   - **Duration**: 1h
   - **Creator**: your-name
   - **Comment**: "Planned maintenance"
5. Click **Create**

---

## 📊 Creating Custom Dashboards

### Example: lingua-app Application Dashboard

1. **Login to Grafana**
2. Click **+** → **Dashboard**
3. Click **Add visualization**
4. Select **Prometheus** datasource

#### Panel 1: Request Rate

**Title**: Requests per Second
**Query**:
```promql
rate(nginx_ingress_controller_requests{ingress="lingua-app"}[5m])
```
**Visualization**: Time series graph

#### Panel 2: Active Pods

**Title**: Running Pods
**Query**:
```promql
sum(kube_pod_status_phase{namespace="lingua-app",phase="Running"})
```
**Visualization**: Stat (single number)

#### Panel 3: CPU Usage

**Title**: CPU Usage by Pod
**Query**:
```promql
rate(container_cpu_usage_seconds_total{namespace="lingua-app",pod=~"lingua-app-.*"}[5m])
```
**Visualization**: Time series graph
**Legend**: `{{pod}}`

#### Panel 4: Memory Usage

**Title**: Memory Usage by Pod
**Query**:
```promql
container_memory_working_set_bytes{namespace="lingua-app",pod=~"lingua-app-.*"} / 1024 / 1024
```
**Visualization**: Time series graph
**Unit**: MiB

#### Panel 5: HPA Status

**Title**: Auto-scaling Status
**Query**:
```promql
kube_horizontalpodautoscaler_status_current_replicas{namespace="lingua-app"}
```
**Visualization**: Time series graph

#### Panel 6: Recent Logs (Loki)

**Datasource**: Loki
**Query**:
```logql
{namespace="lingua-app"} |= ""
```
**Visualization**: Logs
**Options**: Show time, Show labels

5. **Arrange panels** in a logical layout
6. **Save dashboard** with name "LinguaLink Application"

---

## 🔔 Setting Up Alert Notifications

### Slack Integration

1. **Create Slack Webhook**:
   - Go to Slack → Apps → Incoming Webhooks
   - Choose channel (e.g., #alerts)
   - Copy webhook URL

2. **Edit Alertmanager ConfigMap**:

```bash
kubectl edit configmap alertmanager-kube-prometheus-stack-alertmanager -n monitoring
```

Add receiver configuration:

```yaml
receivers:
- name: 'slack-notifications'
  slack_configs:
  - api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'
    channel: '#alerts'
    title: '{{ .GroupLabels.alertname }}'
    text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

route:
  group_by: ['alertname', 'cluster']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'slack-notifications'
  routes:
  - match:
      severity: critical
    receiver: 'slack-notifications'
```

3. **Reload Alertmanager**:

```bash
kubectl rollout restart statefulset alertmanager-kube-prometheus-stack-alertmanager -n monitoring
```

### Email Integration

Similar process with SMTP configuration:

```yaml
receivers:
- name: 'email-notifications'
  email_configs:
  - to: 'devops@company.com'
    from: 'alertmanager@company.com'
    smarthost: 'smtp.gmail.com:587'
    auth_username: 'alerts@company.com'
    auth_password: 'app-password-here'
```

---

## 📈 Monitoring Best Practices

### 1. **Dashboard Organization**

- **Overview Dashboard**: High-level cluster health
- **Component Dashboards**: Deep dives per service
- **SLO Dashboards**: Business metrics and SLIs

### 2. **Alert Hygiene**

- **Don't alert on symptoms, alert on causes**
- **Use appropriate severities**: Critical (wakes you up), Warning (investigate next day), Info (FYI)
- **Set meaningful thresholds**: Based on actual capacity, not arbitrary percentages
- **Include runbooks**: Link to resolution steps in alert annotations

### 3. **Log Retention**

Current configuration:
- **Prometheus**: 7 days retention
- **Loki**: 10Gi storage (approximately 7-14 days depending on volume)

For longer retention:
- Increase PVC size
- Configure remote storage (S3, GCS)
- Implement log archiving

### 4. **Performance**

- **Use recording rules** for frequently queried metrics
- **Limit query range**: Don't query years of data unnecessarily
- **Use efficient LogQL**: `|= "text"` is faster than `|~ "regex"`
- **Index labels carefully**: Don't use high-cardinality labels

---

## 🐛 Troubleshooting

### Issue: Grafana Shows "No Data"

**Causes:**
1. Datasource not configured
2. Query syntax error
3. Time range mismatch
4. No actual data in that time range

**Solutions:**
```bash
# Check datasource connection
curl -u admin:password http://localhost:3000/api/datasources

# Verify Prometheus has data
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
# Open http://localhost:9090/targets - all should be UP

# Check time range in Grafana (top right)
# Try "Last 5 minutes" first
```

### Issue: Loki Queries Return Nothing

**Causes:**
1. Promtail not running
2. Incorrect namespace/labels
3. Logs not yet shipped

**Solutions:**
```bash
# Check Promtail status
kubectl get pods -n logging -l app.kubernetes.io/name=promtail

# Check Promtail logs
kubectl logs -n logging -l app.kubernetes.io/name=promtail --tail=50

# Verify Loki received logs
kubectl logs -n logging -l app.kubernetes.io/name=loki --tail=50 | grep "POST /loki/api/v1/push"

# Test simple query first
{namespace="logging"}  # Should show Loki's own logs
```

### Issue: High Memory Usage

**Causes:**
1. Too many metrics being scraped
2. Long retention period
3. High-cardinality labels

**Solutions:**
```bash
# Check Prometheus memory
kubectl top pod -n monitoring | grep prometheus

# Reduce retention (in HelmRelease)
retention: 3d  # Instead of 7d

# Check cardinality
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
# Go to Status → TSDB Status → check "Top 10 label names with value count"
```

---

## 📚 Additional Resources

### Official Documentation

- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/grafana/latest/)
- [Loki Documentation](https://grafana.com/docs/loki/latest/)
- [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)

### PromQL Learning

- [PromQL Tutorial](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [PromQL Examples](https://prometheus.io/docs/prometheus/latest/querying/examples/)
- [Common PromQL Mistakes](https://www.robustperception.io/common-query-patterns-in-promql)

### LogQL Learning

- [LogQL Tutorial](https://grafana.com/docs/loki/latest/logql/)
- [LogQL Examples](https://grafana.com/docs/loki/latest/logql/log_queries/)

---

## 🎓 Monitoring Skill Demonstration

This monitoring stack demonstrates proficiency in:

- **Prometheus**: Metrics collection, scraping, storage, and querying
- **Grafana**: Dashboard creation, visualization, datasource management
- **Loki**: Log aggregation, querying (LogQL), retention management
- **Alertmanager**: Alert routing, grouping, silencing
- **Kubernetes**: ServiceMonitors, PodMonitors, PrometheusRules
- **Observability**: SRE practices, SLOs, incident response
- **PromQL**: Advanced metric queries and aggregations
- **LogQL**: Log parsing, filtering, and analysis

---

**Last Updated**: 2025-01-13
**Maintained By**: Maria Vulcu
**Environment**: Production (vps-c9e6ac23)

---

*This guide provides comprehensive monitoring and observability capabilities suitable for production operations and SRE best practices.*
