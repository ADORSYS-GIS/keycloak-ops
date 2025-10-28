# Technical Guide

Comprehensive technical documentation for the Keycloak deployment.

## Table of Contents

- [Technical Guide](#technical-guide)
  - [Table of Contents](#table-of-contents)
  - [Image Configuration](#image-configuration)
    - [Why bitnamilegacy Images?](#why-bitnamilegacy-images)
    - [Security Flag Required](#security-flag-required)
    - [Current Image Versions](#current-image-versions)
  - [Configuration Details](#configuration-details)
    - [Default Credentials](#default-credentials)
    - [Database Configuration](#database-configuration)
    - [Resource Allocation](#resource-allocation)
      - [Keycloak Pod](#keycloak-pod)
      - [PostgreSQL Pod](#postgresql-pod)
    - [Persistent Storage](#persistent-storage)
  - [Architecture Decisions](#architecture-decisions)
    - [StatefulSets vs Deployments (Kustomize)](#statefulsets-vs-deployments-kustomize)
  - [Troubleshooting](#troubleshooting)
    - [Issue: Pods Stuck in ImagePullBackOff](#issue-pods-stuck-in-imagepullbackoff)
    - [Issue: Keycloak CrashLoopBackOff with Password Authentication Failed](#issue-keycloak-crashloopbackoff-with-password-authentication-failed)
    - [Issue: Keycloak Takes Long to Start](#issue-keycloak-takes-long-to-start)
    - [Issue: "Original containers have been substituted" Warning](#issue-original-containers-have-been-substituted-warning)
    - [Issue: Port Already in Use](#issue-port-already-in-use)
    - [Check Deployment Status](#check-deployment-status)
  - [Development vs Production](#development-vs-production)
    - [Development (values-dev.yaml)](#development-values-devyaml)
    - [Production Considerations](#production-considerations)

## Image Configuration

### Why bitnamilegacy Images?

This deployment uses **`bitnamilegacy`** images instead of the standard `bitnami` images:

- **Keycloak**: `docker.io/bitnamilegacy/keycloak:26.3.3-debian-12-r0`
- **PostgreSQL**: `docker.io/bitnamilegacy/postgresql:17.6.0-debian-12-r0`
- **Config CLI**: `docker.io/bitnamilegacy/keycloak-config-cli:6.4.0-debian-12-r11`

**Reasons**:

1. **Bitnami Image Migration**: As of August 28, 2025, Bitnami moved older versioned images to the `bitnamilegacy` repository
2. **Chart Compatibility**: The Bitnami Helm chart is specifically designed for these image paths and their environment variables. Official Keycloak images use different configuration patterns that are incompatible with the chart's templates
3. **Production Stability**: These legacy images are stable, tested, and fully compatible with the chart's configuration. They receive security updates and are maintained by VMware/Broadcom
4. **Integration**: The chart expects specific container structure, environment variables, and initialization scripts that only Bitnami-packaged images provide

**Note**: Despite the "legacy" name, these images are actively maintained and receive security patches. They're legacy only in the sense that they're the older naming convention.

### Security Flag Required

Because we're using `bitnamilegacy` images, you must set this flag in your values file:

```yaml
global:
  security:
    allowInsecureImages: true  # Required for bitnamilegacy images
```

**What this does**:
- Tells the Bitnami chart to skip image verification
- `bitnamilegacy` images are not recognized as official Bitnami images by the chart's security checks
- This is **safe** when using documented `bitnamilegacy` images from Docker Hub

**Security implications**:
- The images themselves are secure and maintained
- The flag only bypasses the chart's repository naming check
- Always verify image sources: `docker.io/bitnamilegacy/*`

### Current Image Versions

| Component | Image | Version |
|-----------|-------|----------|
| Keycloak | `bitnamilegacy/keycloak` | 26.3.3-debian-12-r0 |
| PostgreSQL | `bitnamilegacy/postgresql` | 17.6.0-debian-12-r0 |
| Config CLI | `bitnamilegacy/keycloak-config-cli` | 6.4.0-debian-12-r11 |

## Configuration Details

### Default Credentials

- **Admin Username**: `user`
- **Admin Password**: Auto-generated (retrieve using command below)

```bash
kubectl get secret my-keycloak -n keycloak -o jsonpath="{.data.admin-password}" | base64 -d
echo
```

**For Development** (values-dev.yaml):
- **Username**: `admin`
- **Password**: `admin123`

### Database Configuration

- **Database**: PostgreSQL 17.6.0
- **Database Name**: `bitnami_keycloak`
- **Database User**: `bn_keycloak`
- **Database Password**: Auto-generated and auto-configured
- **Connection**: Internal service (`my-keycloak-postgresql:5432`)
- **Persistence**: 8Gi PVC for data storage

### Resource Allocation

#### Keycloak Pod
- **CPU Requests**: 500m
- **CPU Limits**: 750m
- **Memory Requests**: 512Mi
- **Memory Limits**: 768Mi

#### PostgreSQL Pod
- **CPU Requests**: 100m
- **CPU Limits**: 150m
- **Memory Requests**: 128Mi
- **Memory Limits**: 192Mi

### Persistent Storage

- **PostgreSQL Data**: 8Gi PVC (auto-provisioned by kind)
- **Storage Class**: `standard` (kind's default)
- **Access Mode**: ReadWriteOnce

## Architecture Decisions

### StatefulSets vs Deployments (Kustomize)

The Kustomize deployment uses **StatefulSets** for both Keycloak and PostgreSQL instead of standard Deployments. This is an intentional architectural choice that provides significant operational benefits.

#### Why StatefulSets?

**Stable Pod Names**
- Pods are named predictably: `keycloak-0`, `keycloak-1`, `keycloak-2`
- Names never change across restarts or updates
- Easy to identify and target specific instances
- Better for debugging and monitoring

**Stable Network Identities**
- Each pod gets a predictable DNS name
- Format: `<pod-name>.<service-name>.<namespace>.svc.cluster.local`
- Example: `keycloak-0.keycloak.keycloak.svc.cluster.local`
- Enables direct pod-to-pod communication if needed

**Ordered Deployment**
- Pods are created, updated, and deleted in order (0 → 1 → 2)
- Pod `N` only starts after pod `N-1` is Running and Ready
- Prevents race conditions during initialization
- Controlled rollout with predictable behavior

**Better for Distributed Caching**
- Keycloak uses Infinispan for distributed session caching
- JDBC_PING discovery works better with stable pod identities
- Cache rebalancing is more efficient
- Reduces cache misses during pod restarts

**Persistent Storage Support**
- Each pod can have its own PVC (if needed)
- PVCs are stable and bound to specific pod ordinals
- Survives pod restarts and rescheduling
- Better for future stateful requirements

#### Comparison: Deployment vs StatefulSet

| Feature | Deployment | StatefulSet |
|---------|------------|-------------|
| Pod Names | Random (keycloak-abc123-xyz) | Ordered (keycloak-0, keycloak-1) |
| Network Identity | Unstable | Stable DNS names |
| Startup Order | Parallel (all at once) | Sequential (0 → 1 → 2) |
| PVC Binding | Shared or ephemeral | Stable per-pod binding |
| Best For | Stateless apps | Stateful apps, databases, caches |
| Scaling | Fast (parallel) | Slower (sequential) |

#### Practical Benefits

**For Development:**
```bash
# Always connect to the same pod
kubectl port-forward -n keycloak dev-keycloak-0 8080:8080

# Easy to identify in logs
kubectl logs -n keycloak dev-keycloak-0 -f
```

**For Production:**
```bash
# Connect to specific instance for debugging
kubectl exec -it -n keycloak prod-keycloak-1 -- bash

# Verify all replicas are running
kubectl get pods -n keycloak
# NAME                READY   STATUS    RESTARTS   AGE
# prod-keycloak-0     1/1     Running   0          10m
# prod-keycloak-1     1/1     Running   0          9m
# prod-keycloak-2     1/1     Running   0          8m
```

**For Cache Clustering:**
- JGroups JDBC_PING can identify cluster members by stable names
- Cache coordination is more reliable
- Better for session affinity and sticky sessions

#### Trade-offs

**Advantages:**
- ✅ Predictable and stable
- ✅ Better for stateful applications
- ✅ Easier debugging and monitoring
- ✅ Optimal for distributed caching

**Disadvantages:**
- ⚠️ Sequential scaling (slower)
- ⚠️ More complex than Deployments
- ⚠️ Requires headless service for full DNS resolution

#### When to Use StatefulSets

Use StatefulSets when your application:
- Requires stable network identities
- Uses distributed caching or clustering
- Needs ordered deployment/shutdown
- Requires persistent storage per instance
- Benefits from predictable pod names

**For Keycloak**: All of the above apply, making StatefulSet the optimal choice for both development and production deployments.

#### Implementation Notes

The Kustomize manifests use StatefulSets in both environments:

**Development** (`overlays/dev`):
- 1 replica: `dev-keycloak-0`
- Simple setup for local testing

**Production** (`overlays/prod`):
- 3 replicas: `prod-keycloak-0`, `prod-keycloak-1`, `prod-keycloak-2`
- High availability with cache clustering
- Sequential rollout ensures stability

See `keycloak-infra/kustomize/base/keycloak-statefulset.yaml` for the implementation.

## Troubleshooting

### Issue: Pods Stuck in ImagePullBackOff

**Symptom**: Pods show `ErrImagePull` or `ImagePullBackOff`

**Cause**: Large images (~800MB each) taking time to download

**Solution**: Be patient! First-time image pull takes 10-15 minutes. Monitor progress:

```bash
kubectl get events -n keycloak --sort-by='.lastTimestamp' | tail -20
```

Look for: `Pulling image "docker.io/bitnamilegacy/keycloak...`

**Timing expectations**:
- **First Deployment**: 15-20 minutes (image downloads + startup)
- **Subsequent Deploys**: 2-3 minutes (images cached locally)

### Issue: Keycloak CrashLoopBackOff with Password Authentication Failed

**Symptom**: 
```
FATAL: password authentication failed for user "bn_keycloak"
```

**Cause**: Secrets and PVCs from previous deployments have mismatched passwords. This happens when:
- PostgreSQL was initialized with one set of credentials
- The data persists in PVC
- A new deployment generates different credentials
- The new credentials don't match the database

**Solution**: Complete cleanup and fresh install:

```bash
# 1. Uninstall release
helm uninstall my-keycloak -n keycloak

# 2. Delete persistent data (important!)
kubectl delete pvc --all -n keycloak

# 3. Delete secrets
kubectl delete secret --all -n keycloak

# 4. Wait for cleanup
kubectl get all -n keycloak  # Should show no resources

# 5. Reinstall
helm install my-keycloak ./keycloak --namespace keycloak
```

### Issue: Keycloak Takes Long to Start

**Symptom**: Pod running but not ready, readiness probe failures

**Cause**: Normal behavior - Keycloak initialization takes 2-3 minutes

**What's happening**:
1. Database schema creation (~1 minute)
2. JGroups clustering setup (~30 seconds)
3. Master realm initialization (~30 seconds)
4. Quarkus application startup (~30 seconds)

**Check logs**:
```bash
kubectl logs my-keycloak-0 -n keycloak -f
```

Look for: `Keycloak 26.3.3 on JVM started in XXXs`

**Timeline**:
```
0:00 - Pod created
0:30 - Container started
1:00 - Database connection established
2:00 - Keycloak initializing
3:00 - Keycloak ready (1/1 Running)
```

### Issue: "Original containers have been substituted" Warning

**Symptom**: Helm install shows security warning about substituted images

```
Warning: original containers have been substituted
```

**Cause**: Using `bitnamilegacy` images triggers Bitnami chart's security check

**Solution**: This is expected! The warning is safe to ignore when:
- `global.security.allowInsecureImages: true` is set
- You're using the documented `bitnamilegacy` images from Docker Hub
- The images match the versions specified in this guide

### Issue: Port Already in Use

**Symptom**: Port forwarding fails with "bind: address already in use"

**Solution**: Use a different local port

```bash
# Instead of port 8080, use 9090
kubectl port-forward -n keycloak svc/my-keycloak 9090:80
# Access: http://localhost:9090

# Or use any available port
kubectl port-forward -n keycloak svc/my-keycloak 8888:80
```

### Check Deployment Status

**Quick status check**:
```bash
kubectl get pods -n keycloak
```

**Detailed pod information**:
```bash
kubectl describe pod my-keycloak-0 -n keycloak
kubectl describe pod my-keycloak-postgresql-0 -n keycloak
```

**View logs**:
```bash
# Keycloak logs
kubectl logs my-keycloak-0 -n keycloak
kubectl logs my-keycloak-0 -n keycloak -f  # Follow logs
kubectl logs my-keycloak-0 -n keycloak --tail=50  # Last 50 lines

# PostgreSQL logs
kubectl logs my-keycloak-postgresql-0 -n keycloak
```

**Check resources**:
```bash
# Check secrets
kubectl get secrets -n keycloak

# Check persistent volumes
kubectl get pvc -n keycloak

# Check services
kubectl get svc -n keycloak

# Check all resources
kubectl get all -n keycloak
```

**Check events**:
```bash
kubectl get events -n keycloak --sort-by='.lastTimestamp'
```

## Development vs Production

### Development (values-dev.yaml)

Use `values-dev.yaml` for local development:

```bash
helm install my-keycloak ./keycloak \
  -f values-dev.yaml \
  --namespace keycloak \
  --create-namespace
```

**Features**:
- Fixed admin credentials (`admin/admin123`)
- Reduced resource limits for local development
- Bundled PostgreSQL instance
- No TLS/SSL
- ClusterIP service
- Faster startup times

**Resource configuration**:
```yaml
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 750m
    memory: 768Mi
```

### Production Considerations

**Before deploying to production, you must configure**:

#### 1. Use External Database

Do not use the bundled PostgreSQL in production:

```yaml
postgresql:
  enabled: false

externalDatabase:
  host: your-postgres-server.example.com
  port: 5432
  user: keycloak
  database: keycloak
  existingSecret: keycloak-db-secret
  existingSecretPasswordKey: password
```

Create the database secret:
```bash
kubectl create secret generic keycloak-db-secret \
  --from-literal=password='your-secure-password' \
  -n keycloak
```

#### 2. Enable TLS/SSL

Always use TLS in production:

```yaml
production: true

tls:
  enabled: true
  autoGenerated:
    enabled: true  # Or provide your own certificates
```

Or use your own certificates:
```yaml
tls:
  enabled: true
  existingSecret: keycloak-tls-secret
```

#### 3. Configure Ingress

Expose Keycloak through an Ingress controller:

```yaml
ingress:
  enabled: true
  hostname: keycloak.yourdomain.com
  ingressClassName: nginx  # or your ingress controller
  tls: true
  certManager: true  # If using cert-manager
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
```

#### 4. Set Appropriate Resource Limits

Production workloads need more resources:

```yaml
resources:
  limits:
    memory: 2Gi
    cpu: 2000m
  requests:
    memory: 1Gi
    cpu: 1000m
```

#### 5. Enable Monitoring

Set up Prometheus metrics:

```yaml
metrics:
  enabled: true
  serviceMonitor:
    enabled: true
    namespace: monitoring
```

#### 6. High Availability

Run multiple replicas:

```yaml
replicaCount: 3

podDisruptionBudget:
  enabled: true
  minAvailable: 2

podAntiAffinity:
  type: hard
```

#### 7. Secure Credentials

Never use default or simple passwords:

```yaml
auth:
  adminUser: admin
  existingSecret: keycloak-admin-secret
```

Create the admin secret:
```bash
kubectl create secret generic keycloak-admin-secret \
  --from-literal=admin-password='your-very-secure-random-password' \
  -n keycloak
```

#### 8. Configure Backup Strategy

- Set up regular database backups
- Test restore procedures
- Document recovery processes
- Consider using PostgreSQL replication

#### 9. Logging and Auditing

Enable detailed logging:

```yaml
logging:
  level: INFO  # Or WARN for production
```

Configure audit logging:
```yaml
extraEnvVars:
  - name: KC_LOG_LEVEL
    value: INFO
  - name: KC_LOG_CONSOLE_COLOR
    value: "false"
```

#### 10. Network Policies

Restrict network access:

```yaml
networkPolicy:
  enabled: true
  allowExternal: false
  ingress:
    - from:
      - podSelector:
          matchLabels:
            app: your-application
```

---

## Summary

This guide covers:
- ✅ Why we use bitnamilegacy images and security considerations
- ✅ Detailed troubleshooting for common issues
- ✅ Configuration options and resource settings
- ✅ Development vs production deployment strategies
- ✅ Production hardening checklist

For basic usage and getting started, see [README.md](README.md).