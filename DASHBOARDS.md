# Automated Grafana Dashboards

## 📊 Overview

This infrastructure includes **automated dashboard provisioning** via GitOps. All dashboards are automatically loaded into Grafana when you push changes to Git - no manual import required!

---

## 🎯 Available Dashboards

### Custom Dashboards (Auto-loaded via ConfigMaps)

#### 1. **Kubernetes Logs (Loki)**
**Path**: Dashboards → Kubernetes Logs (Loki)
**UID**: `kubernetes-logs-loki`

**What it shows:**
- **Log Rate**: Real-time logs per second across selected namespaces
- **Error Count**: Gauge showing errors in last 5 minutes (errors, failures, exceptions)
- **Logs by Namespace**: Pie chart distribution of log volume
- **Top 10 Pods**: Table showing pods generating most logs
- **Live Logs**: Real-time log stream with search capability
- **Error Logs**: Filtered view showing only errors/failures

**Variables:**
- `$namespace` - Multi-select namespace filter (default: All)
- `$search` - Text search filter for live logs

**Use Cases:**
- Quick error detection across cluster
- Identify chatty pods
- Real-time troubleshooting
- Log volume analysis

---

#### 2. **LinguaLink Application**
**Path**: Dashboards → LinguaLink Application
**UID**: `lingua-app-dashboard`

**What it shows:**
- **Running Pods**: Gauge (threshold: red <1, yellow 1-2, green ≥2)
- **HPA Current Replicas**: Real-time replica count
- **Avg CPU Usage**: Cluster-wide CPU utilization %
- **Avg Memory Usage**: Cluster-wide memory utilization %
- **CPU Usage by Pod**: Time series graph per pod
- **Memory Usage by Pod**: Time series graph per pod
- **HPA Scaling Activity**: Shows current/desired/min/max replicas over time
- **Pod Restarts**: Tracks container restart events
- **Application Logs**: Integrated Loki logs for lingua-app namespace

**Auto-refresh**: Every 10 seconds

**Use Cases:**
- Application health monitoring
- Auto-scaling visualization
- Performance troubleshooting
- Resource usage tracking
- Quick access to application logs

---

### Community Dashboards (Auto-loaded from grafana.com)

#### 3. **Kubernetes Cluster Monitoring**
**Folder**: Kubernetes
**Dashboard ID**: 7249
**Source**: https://grafana.com/grafana/dashboards/7249

**What it shows:**
- Cluster-wide resource usage (CPU, memory, network)
- Node status and health
- Pod distribution and states
- Resource requests vs limits vs actual usage
- Capacity planning metrics

---

#### 4. **Kubernetes Deployments/StatefulSets/DaemonSets**
**Folder**: Kubernetes
**Dashboard ID**: 8588
**Source**: https://grafana.com/grafana/dashboards/8588

**What it shows:**
- Deployment replica status
- StatefulSet health
- DaemonSet coverage
- Resource consumption per workload
- Pod scheduling and evictions

---

#### 5. **Node Exporter Full**
**Folder**: Kubernetes
**Dashboard ID**: 1860
**Source**: https://grafana.com/grafana/dashboards/1860

**What it shows:**
- Comprehensive node metrics
- System load averages (1m, 5m, 15m)
- CPU breakdown (user, system, iowait, idle)
- Memory breakdown (used, cached, buffered, free, swap)
- Disk I/O, space usage, inode usage
- Network interface statistics (bandwidth, errors, drops)
- File descriptor usage
- System uptime and kernel version

**Use Cases:**
- Node performance analysis
- Disk space monitoring
- Network troubleshooting
- System health checks

---

#### 6. **Loki Metrics Dashboard**
**Folder**: Kubernetes
**Dashboard ID**: 13639
**Source**: https://grafana.com/grafana/dashboards/13639

**What it shows:**
- Loki ingestion rate
- Query performance
- Log stream count
- Chunk operations
- Cache hit rates
- Distributor/Ingester/Querier metrics

**Use Cases:**
- Loki performance monitoring
- Log pipeline health
- Capacity planning for log storage

---

## 🔧 How It Works

### Automated Loading via GitOps

1. **Dashboard JSON files** are stored in Git:
   ```
   apps/infra/monitoring/dashboards/
   ├── loki-logs-dashboard.json
   ├── lingua-app-dashboard.json
   ├── loki-logs-configmap.yaml
   ├── lingua-app-configmap.yaml
   └── kustomization.yaml
   ```

2. **ConfigMaps** are created with label `grafana_dashboard: "1"`:
   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     labels:
       grafana_dashboard: "1"
     name: grafana-dashboard-loki-logs
     namespace: monitoring
   data:
     loki-logs-dashboard.json: |
       {...dashboard JSON...}
   ```

3. **Grafana sidecar** automatically detects ConfigMaps with this label:
   ```yaml
   grafana:
     sidecar:
       dashboards:
         enabled: true
         label: grafana_dashboard
         labelValue: "1"
         searchNamespace: monitoring
   ```

4. **Flux CD** syncs changes from Git → Kubernetes → Grafana loads dashboards

**Result**: Push to Git → Wait 1 minute → Dashboards appear in Grafana automatically!

---

## 🚀 Adding New Dashboards

### Method 1: Via ConfigMap (Recommended for GitOps)

1. **Create your dashboard** in Grafana UI or export existing one

2. **Export JSON**:
   - Dashboard → Settings (⚙️) → JSON Model
   - Copy JSON

3. **Create ConfigMap file**:
   ```bash
   cd apps/infra/monitoring/dashboards

   # Save JSON to file
   cat > my-dashboard.json <<EOF
   {
     "title": "My Dashboard",
     "uid": "my-dashboard",
     ...your JSON...
   }
   EOF

   # Create ConfigMap YAML
   kubectl create configmap grafana-dashboard-my-dashboard \
     --from-file=my-dashboard.json \
     --dry-run=client -o yaml > my-dashboard-configmap.yaml
   ```

4. **Add metadata**:
   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: grafana-dashboard-my-dashboard
     namespace: monitoring
     labels:
       grafana_dashboard: "1"  # Important!
   data:
     my-dashboard.json: |
       {...}
   ```

5. **Add to kustomization**:
   ```yaml
   # apps/infra/monitoring/dashboards/kustomization.yaml
   resources:
     - loki-logs-configmap.yaml
     - lingua-app-configmap.yaml
     - my-dashboard-configmap.yaml  # Add this line
   ```

6. **Commit and push**:
   ```bash
   git add apps/infra/monitoring/dashboards/
   git commit -m "Add my custom dashboard"
   git push
   ```

7. **Wait ~1 minute** for Flux to sync and Grafana sidecar to reload

---

### Method 2: Via grafana.com Dashboard ID

1. **Find dashboard** on https://grafana.com/grafana/dashboards/

2. **Add to Helm values** in `apps/infra/monitoring/helmrelease.yaml`:
   ```yaml
   grafana:
     dashboards:
       kubernetes:
         my-dashboard:
           gnetId: 12345  # Dashboard ID from grafana.com
           revision: 1
           datasource: Prometheus
   ```

3. **Commit and push**:
   ```bash
   git add apps/infra/monitoring/helmrelease.yaml
   git commit -m "Add dashboard ID 12345"
   git push
   ```

---

## 🔍 Dashboard Variables

### Using Variables in Queries

Variables make dashboards dynamic and reusable:

**Namespace selector:**
```yaml
{
  "name": "namespace",
  "type": "query",
  "datasource": "Loki",
  "query": {"label": "namespace"},
  "multi": true,
  "includeAll": true
}
```

**Usage in query:**
```promql
{namespace=~"$namespace"}
```

**Text search:**
```yaml
{
  "name": "search",
  "type": "textbox"
}
```

**Usage in query:**
```logql
{namespace="lingua-app"} |~ "(?i)$search"
```

---

## 📝 Best Practices

### 1. **Dashboard Organization**

- Use folders to group related dashboards
- Naming convention: `[Category] - [Specific Focus]`
- Example: `Kubernetes - Logs`, `Application - LinguaLink`

### 2. **UID Management**

Always set explicit UIDs:
```json
{
  "uid": "my-unique-dashboard-id",
  "title": "My Dashboard"
}
```

This ensures:
- Consistent URLs
- No duplicate dashboards
- Proper versioning

### 3. **Datasource Configuration**

Use datasource UIDs, not names:
```json
{
  "datasource": {
    "type": "prometheus",
    "uid": "prometheus"  ← Use this, not "Prometheus"
  }
}
```

Available UIDs in our setup:
- `prometheus` - Prometheus datasource
- `loki` - Loki datasource

### 4. **Query Performance**

- Use appropriate time ranges: `$__interval` for dynamic intervals
- Limit data points: `rate()[5m]` instead of raw metrics
- Use recording rules for frequently queried metrics
- Add `topk()` or `bottomk()` to limit series

### 5. **Panel Naming**

- Clear, descriptive titles
- Include units in title if not obvious
- Example: "Memory Usage (GB)" not "Memory"

### 6. **Variables**

- Provide sensible defaults
- Use `includeAll` option where appropriate
- Test all variable combinations

---

## 🎨 Customizing Existing Dashboards

All dashboards are editable! Changes can be:

### Temporary (Session only)
- Edit in Grafana UI
- Changes lost on dashboard reload

### Permanent (GitOps)
1. Make changes in Grafana UI
2. Export JSON (Settings → JSON Model)
3. Update ConfigMap file in Git
4. Commit and push
5. Dashboard will be overwritten with your version on next sync

---

## 🐛 Troubleshooting

### Dashboard Not Appearing

**1. Check ConfigMap exists:**
```bash
kubectl get configmap -n monitoring | grep dashboard
```

**2. Check label is correct:**
```bash
kubectl get configmap grafana-dashboard-loki-logs -n monitoring -o yaml | grep grafana_dashboard
```

Should show: `grafana_dashboard: "1"`

**3. Check Grafana sidecar logs:**
```bash
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana -c grafana-sc-dashboard
```

Look for: `"Updating dashboard: my-dashboard"`

**4. Force Grafana pod restart:**
```bash
kubectl rollout restart deployment kube-prometheus-stack-grafana -n monitoring
```

---

### Dashboard Shows "No Data"

**1. Check datasource connection:**
- Go to Grafana → Connections → Data sources
- Click on Prometheus or Loki
- Click "Test" button
- Should show: ✅ "Data source is working"

**2. Check datasource UID matches:**
```json
{
  "datasource": {
    "uid": "prometheus"  ← Must match actual UID
  }
}
```

**3. Verify time range:**
- Check dashboard time picker (top right)
- Try "Last 5 minutes" first
- Ensure data exists in that time range

**4. Test query directly:**
- Prometheus: http://localhost:9090 (port-forward)
- Loki: Grafana → Explore → Loki

---

### Dashboard Appears But Wrong Version

**1. Check ConfigMap content:**
```bash
kubectl get configmap grafana-dashboard-loki-logs -n monitoring -o yaml
```

**2. Force dashboard reload:**
```bash
# Delete and recreate (Flux will re-apply)
kubectl delete configmap grafana-dashboard-loki-logs -n monitoring
flux reconcile kustomization vps -n flux-system
```

**3. Clear Grafana cache:**
- Restart Grafana pod (see above)

---

## 📚 Additional Resources

### Grafana Documentation
- [Dashboard Best Practices](https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/best-practices/)
- [Dashboard JSON Model](https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/view-dashboard-json-model/)
- [Variables](https://grafana.com/docs/grafana/latest/dashboards/variables/)

### Query Languages
- [PromQL Basics](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [LogQL Basics](https://grafana.com/docs/loki/latest/logql/)

### Community Dashboards
- [Grafana Dashboard Repository](https://grafana.com/grafana/dashboards/)

---

## 🎓 Learning Path

**Beginner:**
1. Explore existing dashboards
2. Modify panel titles and descriptions
3. Adjust time ranges and refresh intervals
4. Add simple panels with basic queries

**Intermediate:**
5. Create custom queries with variables
6. Add conditional formatting (thresholds)
7. Build multi-panel dashboards
8. Export and version control in Git

**Advanced:**
9. Use transformations and overrides
10. Create recording rules for complex queries
11. Build alerting rules based on dashboard queries
12. Develop organization-wide dashboard templates

---

**Last Updated**: 2025-01-13
**Maintained By**: Maria Vulcu
**Environment**: Production (vps-c9e6ac23)

---

*This dashboard automation demonstrates production-grade observability practices with full GitOps workflow integration.*
