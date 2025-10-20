# Keycloak Helm Deployment Guide

This guide explains how to deploy Keycloak to a Kubernetes cluster using Helm.

## Prerequisites

- Kubernetes cluster (v1.24+)
- Helm 3.x installed
- kubectl configured to access your cluster
- (Optional) cert-manager for TLS certificates
- (Optional) NGINX Ingress Controller

## Local Development Setup (Kind/Minikube)

If you're testing on a local Kubernetes cluster (Kind, Minikube, Docker Desktop), you need to ensure proper storage provisioning:

### Kind Cluster Setup

```bash
# Create a Kind cluster with local storage support
kind create cluster --name keycloak-demo

# Verify storage class is available
kubectl get storageclass
```

Kind comes with `standard` storage class by default. If PostgreSQL pods remain in `Pending` state, check:

```bash
# Check PVC status
kubectl get pvc -n keycloak

# Describe the PVC to see issues
kubectl describe pvc -n keycloak
```

### Quick Fix for Storage Issues

If the PostgreSQL PVC is pending, you can:

**Option 1:** Disable persistence for testing:

```bash
helm install keycloak ./helm/keycloak \
  --namespace keycloak \
  --set keycloak.admin.password=admin \
  --set postgresql.primary.persistence.enabled=false
```

**Option 2:** Use an external database (see section below).

### Minikube Setup

```bash
# Start Minikube
minikube start

# Enable storage provisioner addon
minikube addons enable storage-provisioner
minikube addons enable default-storageclass
```

## Quick Start

**⚠️ Important:** The Helm chart does NOT include PostgreSQL by default (`postgresql.enabled=false` in values.yaml). You must deploy PostgreSQL separately or use an external database.

### Option 1: Deploy with Bundled PostgreSQL (Local Testing)

```bash
# Step 1: Create namespace
kubectl create namespace keycloak

# Step 2: Deploy PostgreSQL from Kustomize base
kubectl apply -f kustomize/base/postgres.yaml

# Step 3: Wait for PostgreSQL to be ready
kubectl wait --for=condition=ready pod -l app=postgres -n keycloak --timeout=300s

# Step 4: Install Keycloak with Helm
# ⚠️ CRITICAL: database password MUST match postgres password (default: keycloak123)
helm install keycloak ./helm/keycloak \
  --namespace keycloak \
  --set keycloak.admin.password=admin123 \
  --set keycloak.database.password=keycloak123

# Step 5: Watch deployment
kubectl get pods -n keycloak -w
# Wait until keycloak pods show: 1/1 Running

# Step 6: Access Keycloak (port-forward for testing)
kubectl port-forward -n keycloak svc/keycloak 8080:8080
```

Access Keycloak at: <http://localhost:8080>  
**Default credentials:** admin / admin123

### Option 2: Deploy with External Database (Production)

```bash
# Step 1: Create namespace
kubectl create namespace keycloak

# Step 2: Install Keycloak pointing to external database
helm install keycloak ./helm/keycloak \
  --namespace keycloak \
  --set keycloak.admin.password=YourSecureAdminPassword \
  --set postgresql.enabled=false \
  --set keycloak.database.host=your-postgres-host.example.com \
  --set keycloak.database.password=YourDbPassword

# Step 3: Verify deployment
kubectl get pods -n keycloak -w
```

## Configuration Options

### Basic Configuration

Create a `values-custom.yaml` file:

```yaml
# Custom values for production deployment
replicaCount: 3

keycloak:
  admin:
    username: admin
    password: "SecurePassword123!"
  
  database:
    vendor: postgres
    host: postgres.database.svc.cluster.local
    port: 5432
    database: keycloak
    username: keycloak
    password: "DbPassword123!"

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: keycloak.yourdomain.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: keycloak-tls
      hosts:
        - keycloak.yourdomain.com

resources:
  limits:
    cpu: 2000m
    memory: 2Gi
  requests:
    cpu: 500m
    memory: 1Gi

postgresql:
  enabled: true
  auth:
    password: "PostgresPassword123!"
```

Deploy with custom values:

```bash
helm install keycloak ./helm/keycloak \
  --namespace keycloak \
  --values values-custom.yaml
```

## Advanced Configuration

### Using Existing Secrets

For production environments, it's recommended to create secrets separately:

```bash
# Create admin secret
kubectl create secret generic keycloak-admin-secret \
  --from-literal=admin-password=YourSecurePassword \
  --namespace keycloak

# Create database secret
kubectl create secret generic keycloak-db-secret \
  --from-literal=db-password=YourDbPassword \
  --namespace keycloak
```

Then reference them in values:

```yaml
keycloak:
  admin:
    existingSecret: keycloak-admin-secret
    existingSecretKey: admin-password
  
  database:
    existingSecret: keycloak-db-secret
    existingSecretKey: db-password
```

### Enable Autoscaling

```yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80
```

### High Availability Setup

```yaml
replicaCount: 3

affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
            - key: app.kubernetes.io/name
              operator: In
              values:
                - keycloak
        topologyKey: kubernetes.io/hostname

resources:
  limits:
    cpu: 4000m
    memory: 4Gi
  requests:
    cpu: 1000m
    memory: 2Gi

postgresql:
  enabled: true
  architecture: replication
  replication:
    enabled: true
  primary:
    persistence:
      enabled: true
      size: 20Gi
```

## Deployment Commands

### Install

```bash
helm install keycloak ./helm/keycloak \
  --namespace keycloak \
  --create-namespace \
  --values values-custom.yaml
```

### Upgrade

```bash
helm upgrade keycloak ./helm/keycloak \
  --namespace keycloak \
  --values values-custom.yaml
```

### Rollback

```bash
helm rollback keycloak <revision> --namespace keycloak
```

### Uninstall

```bash
helm uninstall keycloak --namespace keycloak
```

## Verification

### Check Deployment Status

```bash
# Check pods
kubectl get pods -n keycloak

# Check services
kubectl get svc -n keycloak

# Check ingress
kubectl get ingress -n keycloak

# View logs
kubectl logs -n keycloak -l app.kubernetes.io/name=keycloak
```

### Access Keycloak

#### Via Port Forward (for testing)

```bash
kubectl port-forward -n keycloak svc/keycloak 8080:8080
```

Then access: <http://localhost:8080>

#### Via Ingress

Access the URL configured in your ingress (e.g., <https://keycloak.yourdomain.com>)

### Get Admin Password

```bash
kubectl get secret keycloak -n keycloak \
  -o jsonpath="{.data.admin-password}" | base64 --decode
```

## Troubleshooting

### Keycloak Pods in CrashLoopBackOff

**Symptom:** Keycloak pods keep restarting and show `CrashLoopBackOff` status.

**Common causes:**

1. **PostgreSQL not deployed or not ready**
   
   If using the bundled postgres approach:
   ```bash
   # Check if postgres is running
   kubectl get pods -n keycloak -l app=postgres
   
   # If not found, deploy it
   kubectl apply -f kustomize/base/postgres.yaml
   
   # Wait for it to be ready
   kubectl wait --for=condition=ready pod -l app=postgres -n keycloak --timeout=300s
   ```

2. **Password mismatch between Keycloak and PostgreSQL**
   
   The database password in Helm (`--set keycloak.database.password`) **MUST** match the password in PostgreSQL.
   
   For bundled postgres from `kustomize/base/postgres.yaml`, the default password is `keycloak123`.
   
   ```bash
   # Check the postgres password
   kubectl get secret postgres-secret -n keycloak -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d
   
   # Check the Helm-managed password
   kubectl get secret keycloak -n keycloak -o jsonpath='{.data.db-password}' | base64 -d
   
   # If they don't match, reinstall with correct password
   helm uninstall keycloak -n keycloak
   helm install keycloak ./helm/keycloak \
     --namespace keycloak \
     --set keycloak.admin.password=admin123 \
     --set keycloak.database.password=keycloak123
   ```

3. **Database connection refused**
   
   Check if Keycloak can reach the database:
   ```bash
   # Test connection from Keycloak pod
   kubectl exec -n keycloak <keycloak-pod> -- \
     nc -zv postgres-service 5432
   ```

### Pods Not Starting

```bash
# Describe pod to see events
kubectl describe pod -n keycloak <pod-name>

# Check container logs
kubectl logs -n keycloak <pod-name>

# Check previous container logs (if pod restarted)
kubectl logs -n keycloak <pod-name> --previous
```

### Database Connection Issues

```bash
# Verify postgres is running
kubectl get pods -n keycloak -l app=postgres

# Check postgres service
kubectl get svc postgres-service -n keycloak

# Test database connectivity
kubectl run -it --rm debug --image=postgres:15-alpine --restart=Never -- \
  psql -h postgres-service.keycloak.svc.cluster.local -U keycloak -d keycloak
```

### Port-Forward Connection Refused

**Symptom:** `kubectl port-forward` fails with "connection refused"

```bash
# Verify pods are actually ready (1/1)
kubectl get pods -n keycloak

# Check service endpoints
kubectl get endpoints keycloak -n keycloak

# Try port-forward directly to pod instead of service
kubectl port-forward -n keycloak <pod-name> 8080:8080

# Check if Keycloak is listening on port 8080
kubectl exec -n keycloak <pod-name> -- netstat -tlnp | grep 8080
```

### Ingress Not Working

```bash
# Check ingress controller
kubectl get pods -n ingress-nginx

# Check ingress events
kubectl describe ingress keycloak -n keycloak

# Verify ingress class
kubectl get ingressclass
```

## Production Checklist

- [ ] Use external managed PostgreSQL database
- [ ] Create secrets separately (don't use values.yaml for passwords)
- [ ] Enable TLS with cert-manager
- [ ] Configure resource limits appropriately
- [ ] Enable autoscaling for high availability
- [ ] Set up monitoring and alerting
- [ ] Configure backup for database
- [ ] Use pod anti-affinity for spreading pods
- [ ] Review and configure security contexts
- [ ] Set up log aggregation

## Monitoring

Add Prometheus annotations to scrape metrics:

```yaml
podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"
  prometheus.io/path: "/metrics"
```

## Backup and Restore

### Backup PostgreSQL Database

```bash
kubectl exec -n keycloak <postgres-pod> -- \
  pg_dump -U keycloak keycloak > keycloak-backup.sql
```

### Restore PostgreSQL Database

```bash
kubectl exec -i -n keycloak <postgres-pod> -- \
  psql -U keycloak keycloak < keycloak-backup.sql
```

## Further Resources

- [Keycloak Official Documentation](https://www.keycloak.org/documentation)
- [Keycloak on Kubernetes Guide](https://www.keycloak.org/operator/installation)
- [Helm Documentation](https://helm.sh/docs/)
