# Technical Notes

## Image Selection Rationale

### Why bitnamilegacy?

As of August 28, 2025, Bitnami restructured their image repository:
- Standard versioned images moved to `docker.io/bitnamilegacy`
- New hardened images available in main `docker.io/bitnami` (latest tag only)
- Free tier users get limited access to hardened images

**Our Choice**: bitnamilegacy
- **Compatibility**: Bitnami Helm chart designed for Bitnami directory structure
- **Stability**: Tested and proven images
- **No Migration Needed**: Works out-of-the-box with existing chart
- **Full Features**: All Bitnami customizations and scripts included

### Why NOT Official Keycloak Images?

Attempted using `quay.io/keycloak/keycloak` but encountered:
1. **Path Incompatibility**: Uses `/opt/keycloak` vs `/opt/bitnami/keycloak`
2. **Init Container Failures**: Chart's prepare-write-dirs fails
3. **Environment Variables**: Different var names and formats
4. **Volume Mounts**: Incompatible mount paths
5. **Database Config**: Different connection parameter format

### Why NOT Official PostgreSQL Images?

Similar issues with `postgres:17-alpine`:
1. **Data Directory**: Uses `/var/lib/postgresql` vs `/bitnami/postgresql`
2. **Initialization Scripts**: Different script locations and execution
3. **Health Checks**: Incompatible probe commands
4. **User Management**: Different authentication configuration

## Secret Management

### Auto-Generated Secrets

The Helm chart creates:
- `my-keycloak`: Contains admin password
- `my-keycloak-postgresql`: Contains DB passwords

**Critical**: Both secrets must be created in same deployment for password sync.

### Secret Mismatch Problem

**Scenario**: 
1. Deploy → Creates secret A with password X
2. Uninstall (but PVC persists)
3. Redeploy → Creates secret B with password Y
4. PostgreSQL uses OLD database (password X) but Keycloak has password Y
5. Result: Authentication failed

**Solution**: Always delete PVCs and secrets together:
```bash
helm uninstall my-keycloak -n keycloak
kubectl delete pvc --all -n keycloak
kubectl delete secret --all -n keycloak
```

## Startup Sequence

### PostgreSQL (my-keycloak-postgresql-0)
1. Image pull: 10-15 min (first time)
2. Init container: 5s
3. PostgreSQL start: 10s
4. Database initialization: 20s
5. Ready: ~1 min after image pulled

### Keycloak (my-keycloak-0)
1. Image pull: 10-15 min (first time)
2. Init container (prepare-write-dirs): 5s
3. Wait for PostgreSQL: Variable
4. Database schema creation: 60s
5. JGroups clustering setup: 30s
6. Quarkus application startup: 30s
7. Master realm initialization: 30s
8. Ready: ~3 min after PostgreSQL ready

**Total First Deploy**: ~18 min  
**Subsequent Deploys**: ~3 min

## Resource Usage

### Actual Resource Consumption (Observed)

**Keycloak Pod:**
- CPU: 200-400m (idle), 500-1000m (startup)
- Memory: 600-800Mi (steady state)
- Ephemeral Storage: ~100Mi

**PostgreSQL Pod:**
- CPU: 50-100m (idle), 150m (startup)
- Memory: 150-200Mi
- PersistentVolume: ~500Mi (initial data)

### Recommended Limits

**Development:**
- Keycloak: 500m CPU, 512Mi memory
- PostgreSQL: 100m CPU, 128Mi memory

**Production:**
- Keycloak: 1000-2000m CPU, 2-4Gi memory
- PostgreSQL: External managed database recommended

## Network Architecture

```
┌─────────────────────────────────────────────┐
│  External Access                            │
│  (kubectl port-forward or Ingress)          │
└──────────────────────┬──────────────────────┘
                   │
                   ▼
         ┌─────────────────┐
         │  my-keycloak    │ Service (ClusterIP)
         │  Port: 80       │
         └────────┬────────┘
                  │
                  ▼
         ┌─────────────────┐
         │  my-keycloak-0  │ Pod
         │  Port: 8080     │
         └────────┬────────┘
                  │
                  │ JDBC Connection
                  ▼
         ┌──────────────────────────┐
         │  my-keycloak-postgresql  │ Service
         │  Port: 5432              │
         └────────┬─────────────────┘
                  │
                  ▼
         ┌───────────────────────────┐
         │  my-keycloak-postgresql-0 │ Pod
         │  Port: 5432               │
         │  PVC: data-my-keycloak-   │
         │       postgresql-0        │
         └───────────────────────────┘
```

## Chart Dependencies

From `Chart.yaml`:
```yaml
dependencies:
- name: postgresql
  version: 16.x.x
  repository: oci://registry-1.docker.io/bitnamicharts
  condition: postgresql.enabled

- name: common
  version: 2.x.x  
  repository: oci://registry-1.docker.io/bitnamicharts
```

**Important**: Chart.lock pins exact versions. Update with:
```bash
helm dependency update ./keycloak
```

## Production Considerations

### High Availability
- Use `replicaCount: 3` minimum
- Configure pod anti-affinity
- Enable PodDisruptionBudget

### External Database
- Use managed PostgreSQL (RDS, Cloud SQL, Azure Database)
- Enable SSL/TLS for DB connections
- Configure connection pooling
- Set up automated backups

### Monitoring
- Enable Prometheus metrics
- Set up Grafana dashboards  
- Configure alerts for:
  - High CPU/memory usage
  - Failed login attempts
  - Database connection failures
  - Certificate expiration

### Security Hardening
- Enable TLS/SSL (production mode)
- Use cert-manager for certificate management
- Implement network policies
- Regular security scanning
- Keep images updated

## Troubleshooting Common Issues

### Issue: Database Schema Already Exists

**Error**: `Liquibase update failed: ERROR: relation "keycloak_role" already exists`

**Cause**: Database from previous deployment wasn't cleaned

**Solution**:
```bash
helm uninstall my-keycloak -n keycloak
kubectl delete pvc --all -n keycloak
helm install my-keycloak ./keycloak --namespace keycloak
```

### Issue: JGroups JDBC_PING Duplicate Key Error

**Error**: `duplicate key value violates unique constraint "constraint_jgroups_ping"`

**Impact**: This is a **WARNING**, not a fatal error. Keycloak will still function correctly.

**Explanation**: Happens when Keycloak pod restarts and tries to re-register in the JGroups cluster table. Safe to ignore.

### Issue: Slow First Startup

**Normal Behavior**: 
- First deployment: 15-20 minutes
- Most time spent pulling images (~800MB each)
- Database schema creation adds 1-2 minutes

**Not Normal**:
- If stuck for >30 minutes, check:
  - Network connectivity to Docker Hub
  - Sufficient node resources
  - Storage provisioner working

## Maintenance Tasks

### Backup Database

```bash
# Dump database
kubectl exec my-keycloak-postgresql-0 -n keycloak -- \
  pg_dump -U bn_keycloak bitnami_keycloak > backup-$(date +%Y%m%d).sql
```

### Restore Database

```bash
# Restore from backup
kubectl exec -i my-keycloak-postgresql-0 -n keycloak -- \
  psql -U bn_keycloak bitnami_keycloak < backup-YYYYMMDD.sql
```

### Update Chart Dependencies

```bash
cd keycloak
helm dependency update
helm upgrade my-keycloak . -n keycloak
```

### View Helm Release History

```bash
helm history my-keycloak -n keycloak
```

### Rollback to Previous Version

```bash
helm rollback my-keycloak <revision-number> -n keycloak
