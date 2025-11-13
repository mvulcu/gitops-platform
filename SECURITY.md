# Security Configuration Guide

## 🔐 Security Overview

This document describes the security measures implemented in the LinguaLink platform deployment.

## ✅ Implemented Security Measures

### 1. Admin API Authorization

**Requirement**: Admin API only under authorization.

**Implementation**:
- API key authentication for all `/api/admin/*` endpoints
- API key stored in SealedSecret
- Environment variable: `API_KEY_ENABLED=true`
- Backend middleware validates `X-API-Key` header

**Usage**:
```bash
curl -X GET https://api.lingua.cachefly.site/api/admin/requests \
  -H "X-API-Key: YOUR_API_KEY"
```

**Location**: `apps/lingua-app/backend/configmap.yaml`, `backend/deployment.yaml`

---

### 2. Telegram Webhook Security

**Requirement**: Telegram webhook with secret and validation.

**Implementation**:
- Webhook secret token stored in SealedSecret
- Backend validates `X-Telegram-Bot-Api-Secret-Token` header
- Environment variable: `TELEGRAM_WEBHOOK_SECRET`

**Setup**:
```bash
# Set webhook with secret token
curl -X POST "https://api.telegram.org/bot${BOT_TOKEN}/setWebhook" \
  -d "url=https://api.lingua.cachefly.site/api/telegram/webhook" \
  -d "secret_token=${WEBHOOK_SECRET}"
```

**Location**: `apps/lingua-app/backend/secret-template.yaml`

---

### 3. CORS Configuration

**Requirement**: Separate CORS for prod/dev.

**Implementation**:
- **Production**: Only `https://lingua.cachefly.site`
- **Development** (local): Can add `http://localhost:3000`
- Configured via `CORS_ORIGINS` environment variable in ConfigMap
- No wildcards in production

**Configuration**:
```yaml
# Production (apps/lingua-app/backend/configmap.yaml)
CORS_ORIGINS: '["https://lingua.cachefly.site"]'

# Development (local .env)
CORS_ORIGINS: '["http://localhost:3000","https://lingua.cachefly.site"]'
```

**Location**: `apps/lingua-app/backend/configmap.yaml`

---

### 4. Database User Permissions

**Requirement**: Limited permissions for DB user.

**Implementation**:
- Separate database user: `lingualink` (not `postgres`)
- User has access only to `lingualink` database
- No superuser privileges
- No access to system tables

**Database Setup**:
```sql
-- Already configured in postgres StatefulSet initialization
CREATE USER lingualink WITH PASSWORD 'secure_password';
CREATE DATABASE lingualink OWNER lingualink;
GRANT ALL PRIVILEGES ON DATABASE lingualink TO lingualink;
-- No SUPERUSER or CREATEDB privileges
```

**Location**: `apps/lingua-app/postgres/configmap.yaml`

---

### 5. Metrics Endpoint Protection

**Requirement**: /metrics only inside cluster.

**Implementation**:
- `/metrics` endpoint **not exposed** via Ingress
- NetworkPolicy restricts access to Prometheus pods only
- Only accessible from `monitoring` namespace

**NetworkPolicy**:
```yaml
# Allow Prometheus scraping
- from:
  - namespaceSelector:
      matchLabels:
        name: monitoring
    podSelector:
      matchLabels:
        app.kubernetes.io/name: prometheus
  ports:
  - protocol: TCP
    port: 8000
```

**Location**: `apps/lingua-app/backend/networkpolicy.yaml`

---

### 6. Container Security

**Requirement**: No latest tags, with securityContext.

**Implementation**:

#### Image Tags
- **Backend**: `ghcr.io/mvulcu/lingualink-api:v1.0.0` (specific version)
- **Frontend**: `ghcr.io/mvulcu/lingua-app:v1.0.0` (specific version)
- **PostgreSQL**: `postgres:15-alpine` (pinned major version)
- ❌ No `latest` tags

#### Pod Security Context
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 1000
  seccompProfile:
    type: RuntimeDefault
```

#### Container Security Context
```yaml
securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: false
  runAsNonRoot: true
  runAsUser: 1000
  capabilities:
    drop:
    - ALL
```

**Location**:
- `apps/lingua-app/backend/deployment.yaml`
- `apps/frontend/lingua-app/deployment.yaml`

---

## 🛡️ Additional Security Features

### 7. TLS/HTTPS Enforcement

- All traffic encrypted with Let's Encrypt certificates
- Automatic certificate renewal via cert-manager
- TLS 1.2+ enforced

**Domains**:
- Frontend: `https://lingua.cachefly.site`
- Backend API: `https://api.lingua.cachefly.site`

---

### 8. Security Headers

**Frontend Ingress**:
```yaml
X-Frame-Options: SAMEORIGIN
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: geolocation=(), microphone=(), camera=()
```

**Backend Ingress**:
```yaml
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Referrer-Policy: strict-origin-when-cross-origin
```

---

### 9. Rate Limiting

- **Frontend**: 50 requests/second per IP
- **Backend**: 100 requests/second per IP
- Configured via nginx ingress annotations

**Location**:
- `apps/frontend/lingua-app/ingress.yaml`
- `apps/lingua-app/backend/ingress.yaml`

---

### 10. Request Size Limits

- **Frontend**: 5MB max request body
- **Backend**: 10MB max request body (for file uploads)

---

### 11. Network Policies

#### Backend Ingress Rules:
- Allow from ingress-controller
- Allow from Prometheus (metrics)
- Allow from same namespace

#### Backend Egress Rules:
- Allow DNS (port 53)
- Allow PostgreSQL (port 5432)
- Allow HTTPS (port 443 for Telegram API)

**Location**: `apps/lingua-app/backend/networkpolicy.yaml`

---

### 12. Secrets Management

- All secrets encrypted with Bitnami SealedSecrets
- Secrets never committed to git in plain text
- Sealed secrets can only be decrypted in the cluster

**Secrets**:
- `postgres-secret`: Database password
- `lingualink-backend-secret`: API keys, tokens, connection strings
- `ghcr-creds`: GitHub Container Registry credentials

---

## 🔧 Security Configuration Checklist

Use this checklist before deploying to production:

### Backend Security
- [x] API_KEY_ENABLED=true in ConfigMap
- [x] Strong API key generated (32+ characters)
- [x] TELEGRAM_WEBHOOK_SECRET generated and configured
- [x] CORS_ORIGINS limited to production domain
- [x] DEBUG=false in production
- [x] Specific image tag (not latest)
- [x] SecurityContext configured
- [x] NetworkPolicy applied
- [x] Metrics not exposed externally

### Frontend Security
- [x] Specific image tag (not latest)
- [x] SecurityContext configured
- [x] Security headers configured
- [x] Rate limiting enabled
- [x] TLS certificate configured

### Database Security
- [x] Limited user permissions
- [x] Strong password (SealedSecret)
- [x] Network access restricted
- [x] Persistent storage encrypted

### Infrastructure Security
- [x] All secrets sealed
- [x] TLS/HTTPS everywhere
- [x] Ingress rate limiting
- [x] Resource limits set
- [x] PodDisruptionBudget configured

---

## 🚨 Security Incident Response

### If API Key is Compromised

1. Generate new API key:
```bash
NEW_API_KEY=$(openssl rand -hex 32)
```

2. Update SealedSecret:
```bash
# Create new secret with updated API_KEY
# Regenerate sealed secret
# Commit and push
```

3. Restart backend pods:
```bash
kubectl rollout restart deployment/lingualink-backend -n lingua-app
```

4. Notify API consumers of new key

---

### If Database Password is Compromised

1. Connect to PostgreSQL:
```bash
kubectl exec -it -n lingua-app postgres-0 -- psql -U lingualink
```

2. Change password:
```sql
ALTER USER lingualink WITH PASSWORD 'new_secure_password';
```

3. Update both secrets:
   - `postgres-secret`: POSTGRES_PASSWORD
   - `lingualink-backend-secret`: DATABASE_URL

4. Restart pods:
```bash
kubectl rollout restart statefulset/postgres -n lingua-app
kubectl rollout restart deployment/lingualink-backend -n lingua-app
```

---

### If Telegram Webhook Secret is Compromised

1. Generate new secret:
```bash
NEW_WEBHOOK_SECRET=$(openssl rand -hex 32)
```

2. Update Telegram webhook:
```bash
curl -X POST "https://api.telegram.org/bot${BOT_TOKEN}/setWebhook" \
  -d "url=https://api.lingua.cachefly.site/api/telegram/webhook" \
  -d "secret_token=${NEW_WEBHOOK_SECRET}"
```

3. Update SealedSecret with new TELEGRAM_WEBHOOK_SECRET

4. Restart backend:
```bash
kubectl rollout restart deployment/lingualink-backend -n lingua-app
```

---

## 📚 Security Best Practices

### 1. Regular Updates
- Monitor CVEs for dependencies
- Update base images regularly
- Keep Kubernetes cluster updated

### 2. Secret Rotation
- Rotate API keys every 90 days
- Rotate database password every 180 days
- Rotate webhook secrets every 180 days

### 3. Monitoring
- Monitor failed authentication attempts
- Alert on unusual API usage patterns
- Review access logs regularly

### 4. Backup & Recovery
- Regular database backups
- Test restoration procedures
- Document recovery steps

### 5. Audit Logging
- Enable Kubernetes audit logging
- Log all admin API requests
- Retain logs for compliance

---

## 📞 Security Contacts

**Security Issues**: Report to repository security advisories
**Infrastructure Team**: maria@lingualink.com

---

**Last Updated**: 2025-01-13
**Review Date**: 2025-04-13 (quarterly review)
**Version**: 1.0
