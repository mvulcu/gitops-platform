# PostgreSQL Database for LinguaLink Backend

Production-ready PostgreSQL 15 deployment using StatefulSet with persistent storage.

## Architecture

- **Image**: `postgres:15-alpine`
- **Replicas**: 1 (single instance)
- **Storage**: 10Gi PersistentVolume using K3s `local-path` storage class
- **Resources**:
  - Requests: 100m CPU, 256Mi memory
  - Limits: 500m CPU, 512Mi memory
- **Service**: ClusterIP on port 5432
- **Health Checks**: Liveness and readiness probes using `pg_isready`

---

## Deployment Steps

### 1. Generate Sealed Secret

Before deploying, you **must** create a SealedSecret for the PostgreSQL password:

```bash
# 1. Fetch the sealed-secrets public key
kubeseal --fetch-cert \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=sealed-secrets \
  > pub-sealed-secrets.pem

# 2. Create a temporary secret with your desired password
kubectl create secret generic postgres-secret \
  --namespace=lingua-app \
  --from-literal=POSTGRES_PASSWORD="your-secure-password-here" \
  --dry-run=client -o yaml > /tmp/postgres-secret.yaml

# 3. Seal the secret
kubeseal --format=yaml --cert=pub-sealed-secrets.pem \
  < /tmp/postgres-secret.yaml > sealed-secret.yaml

# 4. Clean up temporary files
rm /tmp/postgres-secret.yaml pub-sealed-secrets.pem

# 5. Enable sealed-secret.yaml in kustomization.yaml
# Uncomment the line: # - sealed-secret.yaml
```

### 2. Update Kustomization

Enable the sealed secret in `kustomization.yaml`:

```yaml
resources:
  - configmap.yaml
  - statefulset.yaml
  - service.yaml
  - sealed-secret.yaml  # Uncomment this line
```

### 3. Commit and Push

```bash
cd /mnt/d/Projects/gitops-platform
git add apps/lingua-app/postgres/
git commit -m "Add PostgreSQL StatefulSet for LinguaLink backend"
git push origin master
```

### 4. Verify Deployment

Monitor Flux reconciliation:

```bash
# Watch Flux sync
flux get kustomizations --watch

# Check lingua-app namespace
kubectl get all -n lingua-app

# Check PostgreSQL pod logs
kubectl logs -n lingua-app postgres-0 -f

# Check PVC is bound
kubectl get pvc -n lingua-app
```

Expected output:
```
NAME                              STATUS   VOLUME                                     CAPACITY
postgres-storage-postgres-0       Bound    pvc-xxxxx-xxxxx-xxxxx-xxxxx-xxxxx         10Gi
```

### 5. Test Database Connection

```bash
# Port-forward to PostgreSQL
kubectl port-forward -n lingua-app svc/postgres 5432:5432

# Connect using psql (in another terminal)
psql -h localhost -U lingualink -d lingualink
# Enter the password you set in the SealedSecret
```

---

## Configuration

### ConfigMap (configmap.yaml)

Non-sensitive configuration:

- `POSTGRES_DB`: `lingualink` (database name)
- `POSTGRES_USER`: `lingualink` (database user)
- `PGDATA`: `/var/lib/postgresql/data/pgdata` (data directory)

### Secret (sealed-secret.yaml)

Sensitive configuration (encrypted):

- `POSTGRES_PASSWORD`: Database password (sealed/encrypted)

### StatefulSet (statefulset.yaml)

Key configuration:
- Single replica (`replicas: 1`)
- PostgreSQL 15 Alpine image
- 10Gi PersistentVolume
- Resource limits to prevent OOM
- Health checks for liveness and readiness
- Automatic volume provisioning via `volumeClaimTemplates`

---

## Accessing PostgreSQL

### From Backend Pods (Same Namespace)

```
postgresql://lingualink:<password>@postgres.lingua-app.svc.cluster.local:5432/lingualink
```

Or using environment variables:

```yaml
env:
- name: DATABASE_URL
  value: postgresql+asyncpg://lingualink:$(POSTGRES_PASSWORD)@postgres:5432/lingualink
- name: POSTGRES_PASSWORD
  valueFrom:
    secretKeyRef:
      name: postgres-secret
      key: POSTGRES_PASSWORD
```

### From Other Namespaces

Use the full DNS name: `postgres.lingua-app.svc.cluster.local:5432`

---

## Backup and Recovery

### Manual Backup

```bash
# Create a backup
kubectl exec -n lingua-app postgres-0 -- \
  pg_dump -U lingualink lingualink > backup-$(date +%Y%m%d).sql

# Or use pg_dumpall for all databases
kubectl exec -n lingua-app postgres-0 -- \
  pg_dumpall -U lingualink > backup-all-$(date +%Y%m%d).sql
```

### Restore from Backup

```bash
# Copy backup to pod
kubectl cp backup-20250113.sql lingua-app/postgres-0:/tmp/backup.sql

# Restore
kubectl exec -n lingua-app postgres-0 -- \
  psql -U lingualink lingualink < /tmp/backup.sql
```

---

## Troubleshooting

### Pod Not Starting

```bash
# Check pod events
kubectl describe pod -n lingua-app postgres-0

# Check logs
kubectl logs -n lingua-app postgres-0

# Common issues:
# - PVC not bound (check storage class)
# - Secret not found (ensure sealed-secret.yaml is created)
# - Resource limits too low (adjust in statefulset.yaml)
```

### PVC Issues

```bash
# Check PVC status
kubectl get pvc -n lingua-app

# Check StorageClass
kubectl get storageclass

# For K3s, ensure local-path provisioner is running
kubectl get pods -n kube-system | grep local-path
```

### Connection Issues

```bash
# Test connection from a debug pod
kubectl run -n lingua-app psql-test --rm -it --restart=Never \
  --image=postgres:15-alpine -- \
  psql -h postgres -U lingualink -d lingualink

# Check service and endpoints
kubectl get svc -n lingua-app postgres
kubectl get endpoints -n lingua-app postgres
```

### Password Issues

```bash
# Verify secret exists
kubectl get secret -n lingua-app postgres-secret

# Check secret data (base64 encoded)
kubectl get secret -n lingua-app postgres-secret -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d
```

---

## Monitoring

PostgreSQL metrics are not yet exported to Prometheus. This will be added in Phase 6.

TODO:
- Add postgres_exporter sidecar
- Create ServiceMonitor for Prometheus scraping
- Create Grafana dashboard for PostgreSQL metrics

---

## Scaling Considerations

This deployment uses a **single replica** StatefulSet. For production high availability:

1. **Option 1**: Use a managed PostgreSQL service (AWS RDS, GCP Cloud SQL, etc.)
2. **Option 2**: Deploy PostgreSQL with replication (requires Patroni/Stolon/pgpool)
3. **Option 3**: Use a PostgreSQL operator (Zalando, Crunchy Data, etc.)

For the current VPS setup with limited resources, a single instance is sufficient.

---

## Security

- ✅ Password stored as SealedSecret (encrypted at rest)
- ✅ ClusterIP service (not exposed externally)
- ✅ Resource limits to prevent DoS
- ✅ Health checks for automatic recovery
- ⚠️ No SSL/TLS for internal cluster traffic (consider adding for production)
- ⚠️ No network policies (consider adding for production)

---

## Next Steps

After PostgreSQL is deployed and verified:

1. ✅ Phase 2 complete: PostgreSQL in Kubernetes
2. 🔜 Phase 3: Create backend deployment manifests
3. 🔜 Phase 4: Integrate with Flux GitOps
4. 🔜 Phase 5: Set up CI/CD pipeline
5. 🔜 Phase 6: Add monitoring and dashboards
6. 🔜 Phase 7: Automate database migrations

---

**Created**: 2025-01-13
**Last Updated**: 2025-01-13
**Version**: 1.0.0
