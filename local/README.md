# Local Development and Testing

This directory contains configuration files for local development and testing with Kind (Kubernetes in Docker).

## Files

### kind-config.yaml

Kind cluster configuration for running Keycloak locally. This is a minimal configuration for development and testing purposes.

**Usage:**

```bash
# Create Kind cluster
kind create cluster --name keycloak-demo --config local/kind-config.yaml

# Delete Kind cluster
kind delete cluster --name keycloak-demo
```

## PostgreSQL Configuration

PostgreSQL is configured in `kustomize/base/postgres.yaml` using the official `postgres:15-alpine` image.

**Note:** For production use, consider using an external managed database service (AWS RDS, GCP Cloud SQL, Azure Database).

## Quick Start Options

### Option 1: Kustomize Deployment (Recommended)

**Fastest way to get started** - Deploys both Keycloak and PostgreSQL in one command:

```bash
# Step 1: Create Kind cluster
kind create cluster --name keycloak-demo --config local/kind-config.yaml

# Step 2: Deploy with Kustomize (includes Keycloak + PostgreSQL)
kubectl apply -k kustomize/base/

# Step 3: Watch pods starting
kubectl get pods -n keycloak -w
# Wait until both pods show: 1/1 Running (Ctrl+C to exit)

# Step 4: Port-forward to access Keycloak
kubectl port-forward -n keycloak svc/keycloak 8080:8080
```

Access Keycloak at: <http://localhost:8080>

**Default credentials:**
- Username: `admin`
- Password: `admin123`

### Option 2: Helm Deployment

**⚠️ Important:** The Helm chart does NOT include PostgreSQL. You must deploy it separately.

```bash
# Step 1: Create Kind cluster
kind create cluster --name keycloak-demo --config local/kind-config.yaml

# Step 2: Create namespace
kubectl create namespace keycloak

# Step 3: Deploy PostgreSQL from Kustomize base
kubectl apply -f kustomize/base/postgres.yaml

# Step 4: Wait for PostgreSQL to be ready
kubectl wait --for=condition=ready pod -l app=postgres -n keycloak --timeout=300s

# Step 5: Deploy Keycloak with Helm
# ⚠️ CRITICAL: database password MUST match postgres password (keycloak123)
helm install keycloak ./helm/keycloak \
  --namespace keycloak \
  --set ingress.enabled=false \
  --set keycloak.admin.password=admin123 \
  --set keycloak.database.password=keycloak123

# Step 6: Watch Keycloak deployment (takes 2-3 minutes)
kubectl get pods -n keycloak -w
# Wait until keycloak pods show: 1/1 Running (Ctrl+C to exit)

# Step 7: Port-forward to access Keycloak
kubectl port-forward -n keycloak svc/keycloak 8080:8080
```

Access Keycloak at: <http://localhost:8080>

**Default credentials:**
- Username: `admin`
- Password: `admin123`

### Option 3: Multi-Environment Testing

Test different environments:

```bash
# 1. Create Kind cluster
kind create cluster --name keycloak-demo --config local/kind-config.yaml

# 2. Deploy base, dev, and prod environments
kubectl apply -k kustomize/base/
kubectl apply -k kustomize/overlays/dev/
kubectl apply -k kustomize/overlays/production/

# 3. Check all deployments
kubectl get pods -A | grep keycloak

# 4. Access each environment
# Base:  kubectl port-forward -n keycloak svc/keycloak 8080:8080
# Dev:   kubectl port-forward -n keycloak-dev svc/keycloak-dev 8081:8080
# Prod:  kubectl port-forward -n keycloak-prod svc/keycloak-prod 8082:8080
```

## Verify Deployment

```bash
# Check that pods are ready (should show 1/1 for both postgres and keycloak)
kubectl get pods -n keycloak

# View Keycloak logs
kubectl logs -n keycloak -l app.kubernetes.io/name=keycloak --tail=50

# Test health endpoints (Keycloak 26.4.0 uses port 9000 for health)
kubectl exec -n keycloak deployment/keycloak -- \
  /bin/sh -c 'wget -qO- http://localhost:9000/health/ready'

# Check service endpoints
kubectl get endpoints -n keycloak
```

## Troubleshooting

### Pods in CrashLoopBackOff

**Issue:** Keycloak pods keep restarting.

**Common causes:**

1. **PostgreSQL not ready**
   ```bash
   # Check postgres status
   kubectl get pods -n keycloak -l app=postgres
   
   # If not running, check events
   kubectl describe pod -n keycloak -l app=postgres
   ```

2. **Password mismatch** (Helm deployments)
   ```bash
   # Verify passwords match
   kubectl get secret postgres-secret -n keycloak -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d
   kubectl get secret keycloak -n keycloak -o jsonpath='{.data.db-password}' | base64 -d
   # Both should be: keycloak123
   ```

3. **Check logs**
   ```bash
   kubectl logs -n keycloak -l app.kubernetes.io/name=keycloak --tail=100
   ```

### Port-Forward Not Working

```bash
# Verify pods are ready
kubectl get pods -n keycloak

# Try port-forward to specific pod
kubectl port-forward -n keycloak <pod-name> 8080:8080

# Check if port 8080 is already in use
lsof -i :8080
```

### Postgres in Wrong Namespace

```bash
# Check all namespaces
kubectl get pods -A | grep postgres

# If in wrong namespace (e.g., default), delete and redeploy
kubectl delete -f kustomize/base/postgres.yaml
kubectl apply -f kustomize/base/postgres.yaml

# Verify correct namespace
kubectl get pods -n keycloak -l app=postgres
```

## Cleanup

```bash
# Delete the Kind cluster (removes everything)
kind delete cluster --name keycloak-demo
```
