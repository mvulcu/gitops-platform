# GitOps Platform - ADHD App Deployment Guide

**Repository**: https://github.com/mvulcu/gitops-platform
**Application**: ADHD Diagnostic Web App
**Last Updated**: 2025-11-27

---

## 🎯 Overview

This GitOps repository manages the Kubernetes manifests for the ADHD diagnostic web application deployed on K3s cluster. FluxCD monitors this repository and automatically applies changes to the cluster.

---

## 📋 Prerequisites

Before deploying, ensure you have:

1. ✅ **K3s cluster running** with FluxCD installed
2. ✅ **kubectl configured** to access the cluster
3. ✅ **Sealed Secrets controller** installed in cluster
4. ✅ **cert-manager** installed for SSL/TLS
5. ✅ **GitHub Container Registry (GHCR) access** for pulling images
6. ✅ **SSH Deploy Key** added to this repository for FluxCD

---

## 🚀 Initial Deployment

### Step 1: Verify ansible-infra FluxCD Configuration

Ensure `/mnt/d/Projects/ADHD/ansible-infra/roles/gitops/tasks/main.yml` has the correct GitRepository:

```yaml
- name: Create GitRepository for gitops-platform
  kubernetes.core.k8s:
    state: present
    kubeconfig: /etc/rancher/k3s/k3s.yaml
    definition:
      apiVersion: source.toolkit.fluxcd.io/v1
      kind: GitRepository
      metadata:
        name: gitops-platform
        namespace: flux-system
      spec:
        interval: 1m
        url: "ssh://git@github.com/mvulcu/gitops-platform"
        ref:
          branch: master
        secretRef:
          name: flux-github-credentials

- name: Create Kustomization for vps cluster
  kubernetes.core.k8s:
    state: present
    kubeconfig: /etc/rancher/k3s/k3s.yaml
    definition:
      apiVersion: kustomize.toolkit.fluxcd.io/v1
      kind: Kustomization
      metadata:
        name: vps-cluster
        namespace: flux-system
      spec:
        interval: 1m
        prune: true
        sourceRef:
          kind: GitRepository
          name: gitops-platform
        path: "./clusters/vps"
        timeout: 2m
```

### Step 2: Deploy GitOps Configuration

```bash
cd /mnt/d/Projects/ADHD/ansible-infra

# Run GitOps playbook
ansible-playbook -i inventory/hosts.ini playbooks/gitops.yml
```

This will:
- Install FluxCD
- Generate SSH key for Git access
- Create GitRepository pointing to mvulcu/gitops-platform
- Create Kustomization for clusters/vps

### Step 3: Add Deploy Key to GitHub

1. The Ansible playbook will display the public SSH key
2. Go to: https://github.com/mvulcu/gitops-platform/settings/keys
3. Click **Add deploy key**
4. Title: `FluxCD VPS`
5. Key: Paste the public key from Ansible output
6. ✅ Check **Allow write access** (for image automation)
7. Click **Add key**

### Step 4: Create GHCR Pull Secret

```bash
kubectl create secret docker-registry ghcr-secret \
  --namespace=adhd \
  --docker-server=ghcr.io \
  --docker-username=mvulcu \
  --docker-password=YOUR_GITHUB_TOKEN \
  --docker-email=your-email@example.com

# Also create in flux-system namespace for ImageRepository
kubectl create secret docker-registry ghcr-secret \
  --namespace=flux-system \
  --docker-server=ghcr.io \
  --docker-username=mvulcu \
  --docker-password=YOUR_GITHUB_TOKEN \
  --docker-email=your-email@example.com
```

### Step 5: Create Sealed Secrets

1. **Get the public key:**
   ```bash
   kubeseal --fetch-cert \
     --controller-name=sealed-secrets \
     --controller-namespace=sealed-secrets \
     > pub-sealed-secrets.pem
   ```

2. **Create plaintext secret (DO NOT commit):**
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

4. **Clean up and commit:**
   ```bash
   rm /tmp/adhd-secrets.yaml
   git add apps/adhd-app/sealed-secret.yaml
   git commit -m "chore: add sealed secrets for ADHD app"
   git push origin master
   ```

### Step 6: Verify Deployment

```bash
# Wait for FluxCD to reconcile (1-2 minutes)
watch flux get all -A

# Check application
kubectl get pods -n adhd
kubectl get deployment -n adhd
kubectl get ingress -n adhd

# Check logs
kubectl logs -n adhd -l app=mywebadhd --tail=50

# Test health endpoint
curl https://www.cachefly.site/api/health
```

---

## 🔄 Day-to-Day Operations

### Automatic Updates

**GitOps workflow is fully automated:**

```
Developer commits code
        ↓
GitHub Actions builds & pushes image
        ↓
FluxCD ImageRepository detects new image
        ↓
FluxCD ImageUpdateAutomation updates deployment.yaml
        ↓
FluxCD Kustomization applies changes
        ↓
✅ Zero-downtime rolling update
```

**No manual intervention required!**

### Manual Operations

```bash
# Force reconciliation
flux reconcile source git gitops-platform
flux reconcile kustomization vps-cluster

# Restart application
kubectl rollout restart deployment/mywebadhd -n adhd

# Rollback
kubectl rollout undo deployment/mywebadhd -n adhd

# View history
kubectl rollout history deployment/mywebadhd -n adhd
```

### Updating Secrets

```bash
# 1. Create new sealed secret (see Step 5)
# 2. Commit and push
# 3. Restart deployment
kubectl rollout restart deployment/mywebadhd -n adhd
```

---

## 📊 Monitoring

### FluxCD Status

```bash
# Overall status
flux get all -A

# Git source status
kubectl get gitrepository -n flux-system

# Kustomization status
kubectl get kustomization -n flux-system

# Image automation status
kubectl get imagerepository,imagepolicy,imageupdateautomation -n flux-system

# View logs
flux logs --all-namespaces --follow
```

### Application Status

```bash
# Pods
kubectl get pods -n adhd -w

# HPA (auto-scaling)
kubectl get hpa -n adhd

# Ingress
kubectl get ingress -n adhd

# Logs (live)
kubectl logs -n adhd -l app=mywebadhd -f --tail=100

# Events
kubectl get events -n adhd --sort-by='.lastTimestamp'
```

---

## 🐛 Troubleshooting

### FluxCD not reconciling

```bash
# Check if GitRepository is accessible
kubectl describe gitrepository gitops-platform -n flux-system

# Check SSH credentials
kubectl get secret flux-github-credentials -n flux-system

# Force reconciliation
flux reconcile source git gitops-platform --with-source
```

### Images not updating

```bash
# Check ImageRepository
kubectl describe imagerepository mywebadhd -n flux-system

# Check for GHCR authentication
kubectl get secret ghcr-secret -n flux-system

# Force image scan
flux reconcile image repository mywebadhd
```

### Pods not starting

```bash
# Check pod status
kubectl describe pod -n adhd -l app=mywebadhd

# Check secrets
kubectl get secret adhd-secrets -n adhd

# Re-seal secrets if needed
```

### Certificate issues

```bash
# Check cert-manager
kubectl get certificate -n adhd
kubectl describe certificate cachefly-tls -n adhd

# Check cert-manager logs
kubectl logs -n cert-manager -l app=cert-manager
```

---

## 🔧 Configuration Files

### Key Files

| File | Purpose |
|------|---------|
| `clusters/vps/kustomization.yaml` | Entry point for cluster |
| `apps/adhd-app/deployment.yaml` | Application deployment |
| `apps/adhd-app/sealed-secret.yaml` | Encrypted secrets |
| `apps/adhd-app/image-automation.yaml` | Auto-update config |
| `apps/adhd-app/ingress.yaml` | HTTP/HTTPS routing |

### Modifying Configuration

1. Edit files in `apps/adhd-app/`
2. Test locally: `kubectl kustomize apps/adhd-app`
3. Commit and push to Git
4. FluxCD will apply changes automatically (1-2 minutes)

---

## 📚 Related Repositories

- **Application Code**: https://github.com/mvulcu/testadhd
- **Infrastructure (Ansible)**: https://github.com/mvulcu/ansible-infra
- **GitOps Platform**: https://github.com/mvulcu/gitops-platform

---

## 🌐 URLs

- **Production**: https://www.cachefly.site
- **Alt**: https://cachefly.site
- **Health**: https://www.cachefly.site/api/health

---

## ✅ Success Criteria

After deployment, verify:

- [ ] `kubectl get pods -n adhd` shows 2+ running pods
- [ ] `kubectl get hpa -n adhd` shows HPA configured
- [ ] `curl https://www.cachefly.site/api/health` returns 200 OK
- [ ] `flux get all -A` shows all resources ready
- [ ] SSL certificate is valid (check browser)
- [ ] Image updates automatically (test with dummy commit)

---

**Managed by**: FluxCD v2
**Deployment**: Zero-downtime rolling updates
**Recovery Time**: < 5 minutes (automatic healing)
