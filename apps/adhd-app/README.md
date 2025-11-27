# ADHD App - GitOps Manifests

**Application**: mywebadhd - ADHD Diagnostic Web Application
**Technology**: Next.js 14, Node.js 20
**Container Registry**: GitHub Container Registry (GHCR)
**Image**: `ghcr.io/mvulcu/testadhd`

---

## 📁 Structure

```
apps/adhd-app/
├── README.md                    # This file
├── namespace.yaml               # Kubernetes namespace
├── deployment.yaml              # Application deployment
├── service.yaml                 # ClusterIP service
├── ingress.yaml                 # Nginx ingress with SSL
├── hpa.yaml                     # Horizontal Pod Autoscaler (2-5 replicas)
├── pdb.yaml                     # Pod Disruption Budget
├── sealed-secret.yaml           # Encrypted secrets (SealedSecret)
├── image-automation.yaml        # FluxCD image update automation
└── kustomization.yaml           # Kustomize config
```

---

## 🚀 Deployment Flow

```
Developer commits code → GitHub Actions
    ↓
Build & Push Docker image to GHCR
    ↓
FluxCD ImageRepository detects new image
    ↓
FluxCD ImageUpdateAutomation updates deployment.yaml
    ↓
FluxCD Kustomization applies changes to cluster
    ↓
✅ Zero-downtime rolling update
```

**Total time: ~3-5 minutes from commit to production**

---

## 🔒 Secrets Management

### Creating Sealed Secrets

1. **Get public key from sealed-secrets controller:**
   ```bash
   kubeseal --fetch-cert --controller-name=sealed-secrets \
     --controller-namespace=sealed-secrets > pub-sealed-secrets.pem
   ```

2. **Create regular secret (DO NOT commit):**
   ```bash
   kubectl create secret generic adhd-secrets \
     --namespace=adhd \
     --from-literal=MONGODB_URI="mongodb://adhd_user:PASSWORD@HOST:27017/mywebadhd?authSource=mywebadhd" \
     --from-literal=NEXT_PUBLIC_SITE_URL="https://www.cachefly.site" \
     --from-literal=CLAUDE_API_KEY="your-key-here" \
     --dry-run=client -o yaml > /tmp/adhd-secrets.yaml
   ```

3. **Seal the secret:**
   ```bash
   kubeseal --format=yaml --cert=pub-sealed-secrets.pem \
     < /tmp/adhd-secrets.yaml > apps/adhd-app/sealed-secret.yaml
   ```

4. **Delete plaintext:**
   ```bash
   rm /tmp/adhd-secrets.yaml
   ```

5. **Commit sealed secret:**
   ```bash
   git add apps/adhd-app/sealed-secret.yaml
   git commit -m "chore: update sealed secrets"
   git push
   ```

---

## 🧪 Testing Locally

```bash
# Test kustomize build
kubectl kustomize apps/adhd-app

# Dry-run apply
kubectl apply -k apps/adhd-app --dry-run=client

# Validate with kustomize
kustomize build apps/adhd-app | kubectl apply --dry-run=client -f -
```

---

## 📊 Monitoring

### Check Application Status

```bash
# Check pods
kubectl get pods -n adhd

# Check deployment
kubectl get deployment -n adhd

# Check HPA (auto-scaling)
kubectl get hpa -n adhd

# View logs
kubectl logs -n adhd -l app=mywebadhd --tail=50 -f

# Check ingress
kubectl get ingress -n adhd
```

### Check FluxCD Automation

```bash
# Check image repository
kubectl get imagerepository -n flux-system

# Check image policy
kubectl get imagepolicy -n flux-system

# Check image update automation
kubectl get imageupdateautomation -n flux-system

# View FluxCD logs
flux logs --all-namespaces --follow
```

---

## 🔄 Manual Operations

### Update Image Manually

```bash
# Restart deployment (pulls latest image)
kubectl rollout restart deployment/mywebadhd -n adhd

# Check rollout status
kubectl rollout status deployment/mywebadhd -n adhd
```

### Rollback

```bash
# Rollback to previous version
kubectl rollout undo deployment/mywebadhd -n adhd

# Check rollout history
kubectl rollout history deployment/mywebadhd -n adhd
```

### Scale Manually

```bash
# Scale to specific replicas
kubectl scale deployment/mywebadhd -n adhd --replicas=3

# HPA will override manual scaling after a while
```

---

## 🔧 Configuration

### Environment Variables

Set via `sealed-secret.yaml`:

| Variable | Description | Required |
|----------|-------------|----------|
| `MONGODB_URI` | MongoDB connection string | ✅ Yes |
| `NEXT_PUBLIC_SITE_URL` | Public site URL | ✅ Yes |
| `CLAUDE_API_KEY` | Claude AI API key | ⚠️ Optional |

### Resource Limits

```yaml
requests:
  cpu: 500m
  memory: 512Mi
limits:
  cpu: 1000m
  memory: 1Gi
```

### Auto-Scaling

- **Min replicas**: 2
- **Max replicas**: 5
- **CPU target**: 70%
- **Memory target**: 80%

---

## 🐛 Troubleshooting

### Pods not starting

```bash
# Check pod status
kubectl describe pod -n adhd -l app=mywebadhd

# Check events
kubectl get events -n adhd --sort-by='.lastTimestamp'
```

### Image not updating

```bash
# Check ImageRepository
kubectl describe imagerepository mywebadhd -n flux-system

# Check ImagePolicy
kubectl describe imagepolicy mywebadhd -n flux-system

# Force reconciliation
flux reconcile image repository mywebadhd
flux reconcile image update mywebadhd
```

### Secrets not decrypting

```bash
# Check sealed-secrets controller
kubectl get pods -n sealed-secrets

# Check sealed secret
kubectl describe sealedsecret adhd-secrets -n adhd

# Re-seal if needed (see Secrets Management section)
```

---

## 📚 Related Documentation

- [FluxCD Documentation](https://fluxcd.io/docs/)
- [Kustomize Documentation](https://kustomize.io/)
- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
- [Kubernetes HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)

---

## 🎯 Production URLs

- **Main**: https://www.cachefly.site
- **Alt**: https://cachefly.site
- **Health Check**: https://www.cachefly.site/api/health

---

**Managed by**: FluxCD GitOps
**Last Updated**: 2025-11-27
