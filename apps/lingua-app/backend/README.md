# LinguaLink Backend - Kubernetes Deployment

## 📋 Overview

This directory contains Kubernetes manifests for the LinguaLink backend API deployment with production-grade security and reliability features.

## 🔐 Security Features

### 1. **Admin API Authorization** ✅
- All `/api/admin/*` endpoints require API key authentication
- API key is stored in SealedSecret and passed via `API_KEY` environment variable
- Enable with `API_KEY_ENABLED=true` in ConfigMap

### 2. **Telegram Webhook Security** ✅
- Webhook endpoint `/api/telegram/webhook` validates requests using `TELEGRAM_WEBHOOK_SECRET`
- Secret token prevents unauthorized webhook calls
- Token is stored in SealedSecret

### 3. **CORS Configuration** ✅
- Production: Only `https://lingua.cachefly.site` is allowed
- No wildcards or `localhost` in production
- Configured via `CORS_ORIGINS` in ConfigMap

### 4. **Metrics Endpoint Protection** ✅
- `/metrics` endpoint is **only accessible within the cluster**
- NetworkPolicy restricts access to Prometheus pods only
- Not exposed via Ingress (internal use only)

### 5. **Container Security** ✅
- Runs as non-root user (UID 1000)
- Read-only root filesystem (where possible)
- Drops all capabilities
- Seccomp profile enabled
- No privilege escalation
- **No `latest` tags** - uses specific version tags (`v1.0.0`)

### 6. **Database Security** ✅
- PostgreSQL user has limited permissions
- Connection string in SealedSecret
- Async connection pooling (10 connections, 20 max overflow)
- No root database access

### 7. **Network Policies** ✅
- Ingress: Only from ingress-controller, Prometheus, and same namespace
- Egress: Only to DNS, PostgreSQL, and HTTPS (for Telegram API)

### 8. **Ingress Security** ✅
- TLS/HTTPS enforced with Let's Encrypt
- Security headers (X-Frame-Options, X-Content-Type-Options, etc.)
- Rate limiting: 100 requests/second
- Request body size limit: 10MB

## 📦 Resources

### Core Components
- **Deployment**: 2 replicas, security context, health checks
- **Service**: ClusterIP on port 80
- **Ingress**: `api.lingua.cachefly.site` with TLS
- **ConfigMap**: Non-sensitive configuration
- **SealedSecret**: Sensitive credentials

### High Availability
- **HorizontalPodAutoscaler**: 2-5 replicas based on CPU/memory
- **PodDisruptionBudget**: Ensures at least 1 replica during updates

### Monitoring
- **ServiceMonitor**: Prometheus metrics scraping
- **NetworkPolicy**: Restricts /metrics access

## 🚀 Deployment Steps

### 1. Build and Push Docker Image

```bash
cd /mnt/d/Projects/lingualink-backend

# Build image
docker build -t ghcr.io/mvulcu/lingualink-api:v1.0.0 .

# Push to GitHub Container Registry
docker push ghcr.io/mvulcu/lingualink-api:v1.0.0
```

### 2. Generate Secrets

```bash
# Get PostgreSQL password from existing secret
POSTGRES_PASSWORD=$(kubectl get secret postgres-secret -n lingua-app -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d)

# Generate API key for admin endpoints
API_KEY=$(openssl rand -hex 32)
echo "Save this API key: $API_KEY"

# Generate Telegram webhook secret
WEBHOOK_SECRET=$(openssl rand -hex 32)

# Get Telegram bot token from @BotFather
TELEGRAM_BOT_TOKEN="<your_bot_token_here>"
```

### 3. Create SealedSecret

```bash
# Create temporary secret file
cat <<EOF > /tmp/backend-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: lingualink-backend-secret
  namespace: lingua-app
type: Opaque
stringData:
  DATABASE_URL: "postgresql+asyncpg://lingualink:${POSTGRES_PASSWORD}@postgres.lingua-app.svc.cluster.local:5432/lingualink"
  TELEGRAM_BOT_TOKEN: "${TELEGRAM_BOT_TOKEN}"
  TELEGRAM_WEBHOOK_SECRET: "${WEBHOOK_SECRET}"
  API_KEY: "${API_KEY}"
EOF

# Generate SealedSecret
kubeseal --format yaml \
  --controller-name=sealed-secrets \
  --controller-namespace=sealed-secrets \
  < /tmp/backend-secret.yaml \
  > apps/lingua-app/backend/sealed-secret.yaml

# Clean up
rm /tmp/backend-secret.yaml
```

### 4. Enable Backend in Kustomization

```bash
# Edit apps/lingua-app/kustomization.yaml
# Uncomment the backend line:
resources:
  - postgres
  - backend  # <-- Uncomment this

# Also uncomment sealed-secret.yaml in apps/lingua-app/backend/kustomization.yaml
```

### 5. Commit and Deploy

```bash
git add apps/lingua-app/backend/
git commit -m "feat: Add backend deployment with security hardening"
git push origin main

# Wait for Flux to reconcile
flux reconcile source git gitops-platform -n flux-system
flux reconcile kustomization vps-cluster -n flux-system
```

### 6. Verify Deployment

```bash
# Check pods
kubectl get pods -n lingua-app -l app=lingualink-backend

# Check logs
kubectl logs -n lingua-app -l app=lingualink-backend --tail=50

# Check ingress
kubectl get ingress -n lingua-app lingualink-backend

# Test health endpoint
curl https://api.lingua.cachefly.site/health

# Test API (should return 401 without API key)
curl -X GET https://api.lingua.cachefly.site/api/admin/requests

# Test with API key
curl -X GET https://api.lingua.cachefly.site/api/admin/requests \
  -H "X-API-Key: YOUR_API_KEY"
```

## 🔧 Configuration

### Environment Variables (ConfigMap)

| Variable | Value | Description |
|----------|-------|-------------|
| `APP_PORT` | `8000` | Server port |
| `DEBUG` | `false` | Production mode |
| `CORS_ORIGINS` | `["https://lingua.cachefly.site"]` | Allowed origins |
| `API_KEY_ENABLED` | `true` | Enable admin API auth |
| `ENABLE_METRICS` | `true` | Enable Prometheus metrics |
| `DB_POOL_SIZE` | `10` | Connection pool size |

### Secrets (SealedSecret)

| Secret | Description |
|--------|-------------|
| `DATABASE_URL` | PostgreSQL connection string |
| `TELEGRAM_BOT_TOKEN` | Bot token from @BotFather |
| `TELEGRAM_WEBHOOK_SECRET` | Webhook validation token |
| `API_KEY` | Admin API authentication key |

## 📊 Monitoring

### Metrics

Access metrics from within the cluster:
```bash
kubectl port-forward -n lingua-app svc/lingualink-backend 8000:80
curl http://localhost:8000/metrics
```

Prometheus automatically scrapes metrics via ServiceMonitor.

### Logs

```bash
# View logs
kubectl logs -n lingua-app -l app=lingualink-backend -f

# View logs in Grafana
# Navigate to Explore > Loki > {namespace="lingua-app",app="lingualink-backend"}
```

## 🔄 Updates

### Updating the Image

```bash
# Build new version
docker build -t ghcr.io/mvulcu/lingualink-api:v1.1.0 .
docker push ghcr.io/mvulcu/lingualink-api:v1.1.0

# Update deployment.yaml
sed -i 's/v1.0.0/v1.1.0/g' apps/lingua-app/backend/deployment.yaml

# Commit and push
git add apps/lingua-app/backend/deployment.yaml
git commit -m "chore: Update backend to v1.1.0"
git push origin main
```

## 🛡️ Security Checklist

- [x] Non-root container user
- [x] Security contexts configured
- [x] No `latest` tags
- [x] Secrets encrypted with SealedSecrets
- [x] CORS configured for production
- [x] Admin API requires authentication
- [x] Telegram webhook validates secret
- [x] Metrics endpoint internal only
- [x] NetworkPolicies applied
- [x] TLS/HTTPS enabled
- [x] Security headers configured
- [x] Rate limiting enabled
- [x] Request size limits
- [x] Database user has minimal permissions

## 📝 API Endpoints

### Public (No Auth Required)
- `POST /api/quote` - Create translation request
- `POST /api/contact` - Contact form
- `POST /api/calc-price` - Price calculator

### Admin (Requires API Key)
- `GET /api/admin/requests` - List requests
- `GET /api/admin/requests/{id}` - Get request
- `PATCH /api/admin/requests/{id}` - Update request
- `POST /api/admin/projects` - Create project
- `GET /api/admin/projects` - List projects

### Webhooks
- `POST /api/telegram/webhook` - Telegram bot webhook (requires secret)

### Internal
- `GET /health` - Health check (liveness)
- `GET /ready` - Readiness check
- `GET /metrics` - Prometheus metrics (internal only)

## 🆘 Troubleshooting

### Pod not starting

```bash
kubectl describe pod -n lingua-app -l app=lingualink-backend
kubectl logs -n lingua-app -l app=lingualink-backend
```

### Database connection issues

```bash
# Test from pod
kubectl exec -it -n lingua-app deployment/lingualink-backend -- sh
# Inside pod:
nc -zv postgres.lingua-app.svc.cluster.local 5432
```

### Ingress not working

```bash
kubectl describe ingress -n lingua-app lingualink-backend
kubectl get certificate -n lingua-app lingualink-backend-tls
```

### Metrics not visible in Prometheus

```bash
kubectl get servicemonitor -n lingua-app lingualink-backend
kubectl describe networkpolicy -n lingua-app lingualink-backend-netpol
```

## 📚 Related Documentation

- [Backend Development README](/mnt/d/Projects/lingualink-backend/README.md)
- [Project Roadmap](../../PROJECT-ROADMAP.md)
- [Sealed Secrets Guide](../../apps/infra/sealed-secrets/)

---

**Last Updated**: 2025-01-13
**Phase**: 3 - Backend Kubernetes Manifests
**Status**: Ready for deployment (pending secrets generation)
