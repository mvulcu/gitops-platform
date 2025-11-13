# GitOps Platform

Production-ready GitOps infrastructure repository for Kubernetes deployments using Flux CD. This repository serves as the single source of truth for all Kubernetes resources deployed on the VPS cluster.

## 🏗️ Architecture

This repository follows GitOps principles where all infrastructure and application configurations are declaratively defined in Git. Flux CD continuously monitors this repository and automatically applies changes to the cluster.

### Technology Stack

- **GitOps Engine**: Flux CD v2
- **Kubernetes**: K3s (lightweight Kubernetes)
- **Secret Management**: Sealed Secrets (Bitnami)
- **Certificate Management**: cert-manager v1.16.2 with Let's Encrypt
- **Monitoring**: kube-prometheus-stack v65.5.1 (Prometheus + Grafana)
- **Ingress**: nginx-ingress-controller
- **Image Registry**: GitHub Container Registry (GHCR)

### Related Repositories

This is part of a 3-repository architecture:

1. **ansible-infra** - Infrastructure provisioning (Ansible IaC)
2. **gitops-platform** (this repo) - Kubernetes manifests and GitOps configuration
3. **lingua-app** - Next.js application source code

## 📁 Repository Structure

```
gitops-platform/
├── apps/
│   ├── infra/                    # Infrastructure components
│   │   ├── sealed-secrets/       # Secret management (v0.27.2)
│   │   ├── cert-manager/         # TLS certificate automation (v1.16.2)
│   │   └── monitoring/           # Prometheus + Grafana stack (v65.5.1)
│   └── frontend/                 # Application deployments
│       └── lingua-app/           # LinguaLink Translations app
└── clusters/
    └── vps/                      # VPS cluster configuration
        └── kustomization.yaml    # Root kustomization
```

## 🚀 Deployed Components

### Infrastructure Layer (`apps/infra/`)

#### 1. Sealed Secrets Controller
- **Version**: v0.27.2
- **Purpose**: Secure secret management using asymmetric encryption
- **Namespace**: sealed-secrets
- **Algorithm**: RSA-4096
- **Encryption Key**: Generated on first install, backed up securely

#### 2. cert-manager
- **Version**: v1.16.2
- **Purpose**: Automated TLS certificate management
- **Namespace**: cert-manager
- **ClusterIssuer**: `letsencrypt-prod`
- **Challenge Type**: HTTP-01 with nginx ingress
- **Email**: mvulcu@users.noreply.github.com
- **Note**: Uses local manifests (964KB) due to Flux/Kustomize URL limitations

#### 3. Monitoring Stack (kube-prometheus-stack)
- **Chart Version**: 65.5.1
- **Purpose**: Complete observability solution
- **Namespace**: monitoring
- **Components**:
  - Prometheus Operator (7-day retention)
  - Grafana (accessible at https://grafana.cachefly.site)
  - Alertmanager
  - Node Exporter
  - kube-state-metrics
- **Grafana Access**: Changed from default during first login (secure)
- **TLS**: Automated via cert-manager

### Application Layer (`apps/frontend/`)

#### lingua-app
- **Image**: ghcr.io/mvulcu/lingua-app:latest
- **Namespace**: lingua-app
- **URL**: https://lingua.cachefly.site
- **Replicas**: 1
- **Resources**:
  - Requests: 128Mi memory, 100m CPU
  - Limits: 512Mi memory, 500m CPU
- **Port**: 3000 (Node.js/Next.js)
- **Health Checks**: Liveness and readiness probes configured
- **TLS**: Automated Let's Encrypt certificate
- **Registry**: Private GHCR with sealed secret authentication

## 🔧 Setup and Installation

### Prerequisites

1. **K3s cluster** running on VPS (provisioned via ansible-infra)
2. **Flux CD** installed and bootstrapped
3. **kubectl** configured with cluster access
4. **kubeseal** CLI for creating sealed secrets

### Initial Bootstrap

If Flux is not yet installed, bootstrap it with:

```bash
# Install Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# Bootstrap Flux to your cluster
flux bootstrap github \
  --owner=mvulcu \
  --repository=gitops-platform \
  --branch=master \
  --path=./clusters/vps \
  --personal
```

### Verify Deployment

Check that all Flux components are running:

```bash
# Check Flux system
flux check

# Monitor Flux reconciliation
flux get all

# Check kustomizations
flux get kustomizations -A

# Check helm releases
flux get helmreleases -A
```

## 🔐 Working with Sealed Secrets

### Creating Sealed Secrets

1. **Get the public certificate from the cluster:**

```bash
kubeseal --fetch-cert \
  --controller-name=sealed-secrets \
  --controller-namespace=sealed-secrets \
  > pub-sealed-secrets.pem
```

2. **Create a regular Kubernetes secret file:**

```bash
kubectl create secret generic my-secret \
  --from-literal=username=admin \
  --from-literal=password=supersecret \
  --dry-run=client \
  -o yaml > secret.yaml
```

3. **Seal the secret:**

```bash
kubeseal --format=yaml --cert=pub-sealed-secrets.pem \
  < secret.yaml > sealed-secret.yaml
```

4. **Commit the sealed secret to Git:**

```bash
# sealed-secret.yaml is safe to commit
git add sealed-secret.yaml
git commit -m "Add sealed secret"
git push
```

### GHCR Authentication Example

The lingua-app uses a sealed secret for GHCR authentication:

```bash
# Create docker-registry secret
kubectl create secret docker-registry ghcr-creds \
  --docker-server=ghcr.io \
  --docker-username=mvulcu \
  --docker-password=$GITHUB_PAT \
  --namespace=lingua-app \
  --dry-run=client -o yaml | \
kubeseal --format=yaml --cert=pub-sealed-secrets.pem \
  > apps/frontend/lingua-app/sealed-secret.yaml
```

## 📊 Accessing Services

### Grafana Monitoring

- **URL**: https://grafana.cachefly.site
- **Username**: admin
- **Password**: Changed during first login (not default)

**Available Dashboards**:
- Kubernetes Cluster Monitoring
- Node Exporter metrics
- Prometheus metrics
- Application performance metrics

### Prometheus

Access Prometheus via port-forward:

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
```

Then open: http://localhost:9090

### lingua-app Application

- **URL**: https://lingua.cachefly.site
- **Status**: Production deployment
- **TLS**: Automated Let's Encrypt certificate

## 🔄 Adding New Applications

### 1. Create Application Directory

```bash
mkdir -p apps/frontend/my-app
```

### 2. Create Kubernetes Manifests

Create the following files in `apps/frontend/my-app/`:

- `namespace.yaml` - Application namespace
- `deployment.yaml` - Deployment configuration
- `service.yaml` - Service definition
- `ingress.yaml` - Ingress with TLS
- `kustomization.yaml` - Kustomize configuration

### 3. Configure TLS Certificate

Add cert-manager annotation to your ingress:

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - my-app.cachefly.site
    secretName: my-app-tls
```

### 4. Update Kustomization

Add your app to `apps/frontend/kustomization.yaml`:

```yaml
resources:
  - lingua-app
  - my-app  # Add this line
```

### 5. Commit and Push

```bash
git add apps/frontend/my-app
git commit -m "Add my-app deployment"
git push
```

Flux will automatically detect and apply changes within 1 minute.

## 🛠️ Troubleshooting

### Check Flux Sync Status

```bash
# Check all Flux resources
flux get all -A

# Check specific kustomization
flux get kustomizations -n flux-system

# Check logs
flux logs --all-namespaces
```

### Reconcile Manually

Force immediate reconciliation:

```bash
# Reconcile everything
flux reconcile kustomization flux-system --with-source

# Reconcile specific kustomization
flux reconcile kustomization vps -n flux-system
```

### Certificate Issues

Check cert-manager certificates:

```bash
# List certificates
kubectl get certificates -A

# Check certificate details
kubectl describe certificate lingua-app-tls -n lingua-app

# Check cert-manager logs
kubectl logs -n cert-manager -l app=cert-manager -f
```

### Sealed Secrets Issues

Check sealed secrets controller:

```bash
# Check controller status
kubectl get pods -n sealed-secrets

# Check controller logs
kubectl logs -n sealed-secrets -l app.kubernetes.io/name=sealed-secrets -f

# Verify sealed secret
kubectl get sealedsecrets -A
```

### Pod Not Starting

Debug pod issues:

```bash
# Check pod status
kubectl get pods -n lingua-app

# Describe pod
kubectl describe pod -n lingua-app <pod-name>

# Check logs
kubectl logs -n lingua-app <pod-name>

# Check events
kubectl get events -n lingua-app --sort-by='.lastTimestamp'
```

## 🔍 Monitoring and Alerts

### View Metrics

```bash
# Check resource usage
kubectl top nodes
kubectl top pods -A

# View Prometheus targets
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
# Open http://localhost:9090/targets
```

### Access Alertmanager

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-alertmanager 9093:9093
# Open http://localhost:9093
```

## 🔒 Security Best Practices

1. **Never commit plain Kubernetes secrets** - Always use sealed secrets
2. **Rotate secrets regularly** - Update credentials periodically
3. **Use RBAC** - Apply principle of least privilege
4. **Enable TLS everywhere** - All public services use HTTPS
5. **Keep components updated** - Regularly update chart versions
6. **Backup encryption keys** - Securely backup sealed-secrets key
7. **Monitor access logs** - Review Grafana and Prometheus regularly

## 📝 Maintenance

### Update Components

#### Update cert-manager

Edit `apps/infra/cert-manager/cert-manager.yaml` with new version manifests from:
https://github.com/cert-manager/cert-manager/releases

#### Update Monitoring Stack

Edit `apps/infra/monitoring/helmrelease.yaml`:

```yaml
spec:
  chart:
    spec:
      version: 65.5.1  # Update this version
```

#### Update Application Image

Edit `apps/frontend/lingua-app/deployment.yaml`:

```yaml
spec:
  template:
    spec:
      containers:
      - image: ghcr.io/mvulcu/lingua-app:v1.2.3  # Update tag
```

Or use Flux Image Automation (not yet configured).

### Backup Strategy

**Critical components to backup:**
1. Sealed Secrets encryption key: `/var/lib/rancher/k3s/server/manifests/sealed-secrets-key.yaml`
2. Let's Encrypt certificates: Stored in cluster, backed by cert-manager
3. Persistent volumes: If using databases
4. Git repository: This repo itself is the backup

## 📚 Additional Resources

- [Flux CD Documentation](https://fluxcd.io/docs/)
- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
- [cert-manager Docs](https://cert-manager.io/docs/)
- [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [Kustomize](https://kustomize.io/)

## 🤝 Contributing

When adding new features or applications:

1. Create a feature branch
2. Test changes in a non-production environment if possible
3. Use `flux diff` to preview changes
4. Commit with descriptive messages
5. Monitor Flux reconciliation after merge

## 📄 License

This infrastructure configuration is maintained by mvulcu.

---

**Status**: Production
**Cluster**: VPS at 141.95.19.212
**Last Updated**: 2025-01-13
**Flux Version**: v2.x
