# LinguaLink Backend Monitoring - Testing Guide

This guide will walk you through testing and verifying the monitoring infrastructure for the LinguaLink Backend API.

## Prerequisites

- Access to the Kubernetes cluster
- kubectl configured with proper credentials
- Port-forward capability or Ingress access to Grafana

---

## 1. Verify Flux Reconciliation

First, check that Flux has picked up and applied the new monitoring resources.

### Check Kustomization Status

```bash
# Check the monitoring kustomization status
flux get kustomizations -n flux-system

# Look for the monitoring kustomization and ensure it shows "Applied revision"
# Example output:
# NAME       REVISION     SUSPENDED  READY   MESSAGE
# monitoring main/42b29d7  False      True    Applied revision: main/42b29d7
```

### Check ServiceMonitor

```bash
# Verify ServiceMonitor exists and is configured correctly
kubectl get servicemonitor lingualink-backend -n lingua-app -o yaml

# Expected output should show:
# - honorLabels: true
# - relabelings with service, pod, namespace mappings
```

### Check Dashboard ConfigMap

```bash
# Verify the backend API dashboard ConfigMap was created
kubectl get configmap grafana-dashboard-backend-api -n monitoring

# Check it has the grafana_dashboard label
kubectl get configmap grafana-dashboard-backend-api -n monitoring -o jsonpath='{.metadata.labels.grafana_dashboard}'
# Should output: 1
```

---

## 2. Access Grafana

### Option A: Port Forward (Development)

```bash
# Port forward Grafana service
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

# Access at: http://localhost:3000
# Default credentials (if not changed):
#   Username: admin
#   Password: prom-operator (or check your values)
```

### Option B: Ingress (Production)

```bash
# Check Grafana Ingress
kubectl get ingress -n monitoring

# Access via your configured domain (e.g., https://grafana.yourdomain.com)
```

---

## 3. Verify Dashboard Auto-Discovery

The Grafana sidecar should automatically load the dashboard within 1-2 minutes.

### In Grafana UI:

1. Navigate to **Dashboards** > **Browse**
2. Look for **"LinguaLink Backend API"** dashboard
3. If you don't see it, check the following:

### Troubleshoot Dashboard Loading

```bash
# Check Grafana sidecar logs
kubectl logs -n monitoring deployment/prometheus-grafana -c grafana-sc-dashboard

# Look for logs like:
# "Added /tmp/dashboards/monitoring-grafana-dashboard-backend-api.json"

# Verify the ConfigMap is visible to Grafana
kubectl get configmap -n monitoring -l grafana_dashboard=1
# Should list: grafana-dashboard-backend-api (and others)

# Force Grafana pod restart if needed
kubectl rollout restart deployment/prometheus-grafana -n monitoring
```

---

## 4. Verify Metrics Collection

### Check Prometheus is Scraping the Backend

```bash
# Port forward Prometheus
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090

# Access Prometheus UI at: http://localhost:9090
```

In Prometheus UI:

1. Go to **Status** > **Targets**
2. Search for `lingua-app/lingualink-backend`
3. Verify the target is **UP** and scraping successfully

### Test Metrics Queries

In Prometheus UI (**Graph** tab), try these queries:

```promql
# Check if backend metrics exist
up{namespace="lingua-app",service="lingualink-backend"}

# Check request rate
rate(http_requests_total{namespace="lingua-app",app="lingualink-backend"}[5m])

# Check request duration
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{namespace="lingua-app",app="lingualink-backend"}[5m])) by (le))
```

If queries return **no data**, check:

```bash
# Verify backend pods are running
kubectl get pods -n lingua-app -l app=lingualink-backend

# Check backend service exists
kubectl get service lingualink-backend -n lingua-app

# Verify metrics endpoint is accessible
kubectl port-forward -n lingua-app svc/lingualink-backend 8000:80
curl http://localhost:8000/metrics
# Should return Prometheus-formatted metrics
```

---

## 5. Generate Test Traffic

To populate the dashboard with data, generate some API traffic.

### Using kubectl port-forward

```bash
# Port forward the backend service
kubectl port-forward -n lingua-app svc/lingualink-backend 8000:80

# Generate test requests
curl http://localhost:8000/health
curl http://localhost:8000/api/v1/pricing/rules
curl http://localhost:8000/docs

# Generate some load (using Apache Bench or similar)
ab -n 1000 -c 10 http://localhost:8000/health
```

### Using In-Cluster Test Pod

```bash
# Create a test pod
kubectl run -n lingua-app test-client --image=curlimages/curl:latest --rm -it -- sh

# Inside the pod, run:
while true; do
  curl -s http://lingualink-backend/health
  curl -s http://lingualink-backend/api/v1/pricing/rules
  sleep 1
done
```

---

## 6. Verify Dashboard Panels

Open the **LinguaLink Backend API** dashboard in Grafana and verify each section:

### Request Overview Section

- **Request Rate (RPS) by Endpoint**: Should show lines for different endpoints
- **Total Requests**: Should show increasing counter
- **In-Progress Requests**: Should show current active requests
- **Total Request Rate**: Should show overall RPS

### Latency Metrics Section

- **Response Time Percentiles**: Should show p50, p95, p99 lines
- **Average Response Time**: Gauge showing average latency

### Error Metrics Section

- **Error Rate (4xx / 5xx)**: Should be empty if no errors (good!)
- **HTTP Status Distribution**: Pie chart showing 200s (and others if present)
- **5xx Error Rate %**: Should be 0% or green

### Health & Database Section

- **Failed Readiness Probes**: Should show "Healthy" (0 failures)
- **Metrics Endpoint Status**: Should show "Up" (green)
- **Pod Health Status**: Should show running pods count
- **Avg CPU Usage**: Should show current CPU percentage
- **Avg Memory Usage**: Should show current memory percentage

### Application Logs Section

- **Error Logs**: Should show ERROR/CRITICAL level logs (if any)
- **Slow Requests**: Should show requests taking >1s (if any)
- **All Application Logs**: Should show streaming logs from backend

---

## 7. Verify Loki Log Integration

### Check Loki is Receiving Logs

```bash
# Port forward Loki
kubectl port-forward -n monitoring svc/loki 3100:3100

# Query Loki API
curl -G -s "http://localhost:3100/loki/api/v1/query_range" \
  --data-urlencode 'query={namespace="lingua-app",app="lingualink-backend"}' \
  --data-urlencode 'limit=10' | jq
```

### Test Log Queries in Grafana

In Grafana:

1. Go to **Explore**
2. Select **Loki** datasource
3. Try these LogQL queries:

```logql
# All backend logs
{namespace="lingua-app",app="lingualink-backend"}

# Error logs only
{namespace="lingua-app",app="lingualink-backend"} | json | level=~"ERROR|CRITICAL"

# Slow requests (if duration field exists)
{namespace="lingua-app",app="lingualink-backend"} | json | __error__="" | unwrap duration | duration > 1000
```

If no logs appear:

```bash
# Check promtail (or log collector) is running
kubectl get pods -n monitoring -l app=promtail

# Check promtail logs
kubectl logs -n monitoring -l app=promtail --tail=50

# Verify backend pods have logs
kubectl logs -n lingua-app -l app=lingualink-backend --tail=20
```

---

## 8. Test Alert Scenarios (Optional)

Generate scenarios to test dashboard responsiveness:

### Test High Error Rate

```bash
# Call a non-existent endpoint to generate 404s
for i in {1..100}; do
  curl http://localhost:8000/nonexistent-endpoint
done
```

Check:
- **Error Rate** panel should show spike
- **HTTP Status Distribution** should show 404 segment
- **Error Logs** panel should show entries (if logged)

### Test High Latency

```bash
# If backend has a slow endpoint, call it repeatedly
# Or simulate load to increase latency
ab -n 5000 -c 50 http://localhost:8000/api/v1/projects
```

Check:
- **Response Time Percentiles** should increase
- **Average Response Time** gauge should go yellow/red if >0.5s/1s

---

## 9. Common Issues & Troubleshooting

### Dashboard Not Appearing

**Cause**: Grafana sidecar didn't detect the ConfigMap

**Solution**:
```bash
# Verify label exists
kubectl get cm grafana-dashboard-backend-api -n monitoring -o jsonpath='{.metadata.labels}'

# Restart Grafana
kubectl rollout restart deployment/prometheus-grafana -n monitoring

# Check sidecar logs
kubectl logs -n monitoring deployment/prometheus-grafana -c grafana-sc-dashboard
```

### No Metrics Data

**Cause**: ServiceMonitor not targeting the service correctly

**Solution**:
```bash
# Check service labels match ServiceMonitor selector
kubectl get service lingualink-backend -n lingua-app --show-labels

# ServiceMonitor selector expects: app=lingualink-backend
# Update service labels if needed

# Verify Prometheus target
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090
# Check Status > Targets > search for "lingualink-backend"
```

### No Log Data

**Cause**: Loki not configured to scrape lingua-app namespace

**Solution**:
```bash
# Check promtail configuration
kubectl get configmap promtail -n monitoring -o yaml

# Ensure namespace_name: lingua-app is included in scrape configs

# Check promtail is reaching the pods
kubectl logs -n monitoring daemonset/promtail --tail=50 | grep lingua-app
```

### Metrics Showing "N/A"

**Cause**: Metric names don't match the queries or no traffic yet

**Solution**:
1. Generate some test traffic (see section 5)
2. Wait 30-60 seconds for Prometheus to scrape
3. Refresh the dashboard
4. If still N/A, verify metric names:

```bash
# Check what metrics are actually available
kubectl port-forward -n lingua-app svc/lingualink-backend 8000:80
curl http://localhost:8000/metrics | grep http_request

# Compare with dashboard queries in Prometheus UI
```

---

## 10. Success Criteria Checklist

- [ ] ServiceMonitor shows in `kubectl get servicemonitor -n lingua-app`
- [ ] Backend API dashboard visible in Grafana
- [ ] All 17 panels render without errors
- [ ] Request metrics show data after generating traffic
- [ ] Latency metrics show p50/p95/p99 values
- [ ] Error metrics show 0% or actual errors when triggered
- [ ] Pod health metrics show running pods
- [ ] CPU/Memory gauges show percentages
- [ ] Log panels show streaming logs
- [ ] Error log panel filters ERROR/CRITICAL correctly
- [ ] Slow requests panel shows requests >1s (if any exist)

---

## Next Steps

Once monitoring is verified:

1. Set up Grafana alerting rules for critical metrics
2. Configure notification channels (Slack, email, PagerDuty)
3. Create runbooks for common alert scenarios
4. Adjust dashboard refresh rate and time ranges as needed
5. Consider adding more panels for business metrics
6. Set up long-term metric retention policies

---

## Additional Resources

- [Prometheus Operator Documentation](https://prometheus-operator.dev/)
- [Grafana Dashboard Best Practices](https://grafana.com/docs/grafana/latest/dashboards/)
- [PromQL Query Examples](https://prometheus.io/docs/prometheus/latest/querying/examples/)
- [LogQL Documentation](https://grafana.com/docs/loki/latest/logql/)
- [FastAPI Instrumentator](https://github.com/trallnag/prometheus-fastapi-instrumentator)

---

**Last Updated**: 2025-11-14
**Dashboard Version**: 1.0
**Maintained By**: LinguaLink DevOps Team
