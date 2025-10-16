# Keycloak Kustomize Deployment Guide

This guide explains how to deploy Keycloak to a Kubernetes cluster using Kustomize.

**✅ Status:** All environments tested and working with Keycloak 26.4.0 + PostgreSQL 15

## Prerequisites

- Kubernetes cluster (v1.24+)
- kubectl with kustomize support (v1.14+)
- (Optional) cert-manager for TLS certificates
- (Optional) NGINX Ingress Controller

## Local Development Setup (Kind/Minikube)

If you're testing on a local Kubernetes cluster, ensure proper storage provisioning:

### Kind Cluster Setup

```bash
# Create a Kind cluster
kind create cluster --name keycloak-demo

# Verify storage class
kubectl get storageclass
```

Kind has a `standard` storage class by default. If PostgreSQL StatefulSet pods remain `Pending`, check PVC status:

```bash
kubectl get pvc -n keycloak-dev
kubectl describe pvc -n keycloak-dev
```

### PostgreSQL Storage Configuration

**Current Configuration:** By default, `kustomize/base/postgres.yaml` has persistent storage **disabled** for local testing (Kind/Minikube).

```yaml
# Disabled for Kind/local testing - no PVC support
# volumeMounts:
# - name: postgres-storage
#   mountPath: /var/lib/postgresql/data
```

**⚠️ Note:** This means postgres data is **ephemeral** and will be lost on pod restart.

**For Production:** Enable persistent storage:

1. Uncomment volumeMounts and volumeClaimTemplates in `postgres.yaml`
2. Ensure your cluster has a working StorageClass
3. Or use an external managed database (AWS RDS, GCP Cloud SQL, Azure Database)

### Minikube Setup

```bash
minikube start
minikube addons enable storage-provisioner
minikube addons enable default-storageclass
```

## Directory Structure

```text
kustomize/
├── base/                    # Base manifests
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── secrets.yaml
│   ├── postgres.yaml
│   └── kustomization.yaml
└── overlays/               # Environment-specific configs
    ├── dev/
    │   └── kustomization.yaml
    ├── staging/
    │   └── kustomization.yaml
    └── production/
        └── kustomization.yaml
```

## Quick Start

**🚀 Fast Deployment (Testing):**

```bash
# Deploy to base namespace (uses default passwords)
kubectl apply -k kustomize/base/

# Check status
kubectl get pods -n keycloak -w

# Access Keycloak
kubectl port-forward -n keycloak svc/keycloak 8080:8080
# Login: admin / admin123
```

### 1. Configure Secrets (Production)

**IMPORTANT:** For production, update the secrets in `kustomize/base/secrets.yaml`:

```bash
# Generate secure passwords
ADMIN_PASSWORD=$(openssl rand -base64 32)
DB_PASSWORD=$(openssl rand -base64 32)

echo "Admin Password: $ADMIN_PASSWORD"
echo "DB Password: $DB_PASSWORD"
```

Update `kustomize/base/secrets.yaml` with these passwords or use kubectl to create secrets:

```bash
kubectl create namespace keycloak

kubectl create secret generic keycloak-admin \
  --from-literal=username=admin \
  --from-literal=password=$ADMIN_PASSWORD \
  --namespace keycloak

kubectl create secret generic keycloak-db \
  --from-literal=username=keycloak \
  --from-literal=password=$DB_PASSWORD \
  --namespace keycloak

kubectl create secret generic postgres-secret \
  --from-literal=POSTGRES_PASSWORD=$DB_PASSWORD \
  --namespace keycloak
```

### 2. Review Configuration

Preview what will be deployed:

```bash
# For base deployment
kubectl kustomize kustomize/base

# For dev environment
kubectl kustomize kustomize/overlays/dev

# For staging environment
kubectl kustomize kustomize/overlays/staging

# For production environment
kubectl kustomize kustomize/overlays/production
```

### 3. Deploy

#### Development Environment

```bash
kubectl apply -k kustomize/overlays/dev
```

#### Staging Environment

```bash
kubectl apply -k kustomize/overlays/staging
```

#### Production Environment

```bash
kubectl apply -k kustomize/overlays/production
```

#### Base Deployment (no overlay)

```bash
kubectl apply -k kustomize/base
```

## Environment Configurations

### Base

- **Namespace:** keycloak
- **Replicas:** 2
- **Hostname:** keycloak.example.com
- **Resources:** Standard (500m CPU, 1Gi RAM)
- **Postgres:** postgres-service
- **Credentials:** admin / admin123 (default - **change for production**)

### Development

- **Namespace:** keycloak-dev
- **Replicas:** 1
- **Hostname:** keycloak-dev.example.com
- **Resources:** Standard (500m CPU, 1Gi RAM)
- **Postgres:** postgres-service-dev
- **Credentials:** admin / admin123 (default - **change for production**)

### Staging

- **Namespace:** keycloak-staging
- **Replicas:** 2
- **Hostname:** keycloak-staging.example.com
- **Resources:** Standard (500m CPU, 1Gi RAM)
- **Postgres:** postgres-service-staging
- **TLS:** Let's Encrypt Staging
- **Credentials:** admin / admin123 (default - **change for production**)

### Production

- **Namespace:** keycloak-prod
- **Replicas:** 3
- **Hostname:** keycloak.example.com
- **Resources:** Enhanced (1000m CPU, 2Gi RAM)
- **Postgres:** postgres-service-prod
- **TLS:** Let's Encrypt Production
- **Credentials:** admin / admin123 (default - **MUST change for production**)

## Customization

### Update Ingress Hostname

Edit the overlay kustomization.yaml:

```yaml
patches:
  - target:
      kind: Ingress
      name: keycloak
    patch: |-
      - op: replace
        path: /spec/rules/0/host
        value: your-domain.com
```

### Change Replica Count

```yaml
patches:
  - target:
      kind: Deployment
      name: keycloak
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 5
```

### Update Resource Limits

```yaml
patches:
  - target:
      kind: Deployment
      name: keycloak
    patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/resources/limits/cpu
        value: 4000m
      - op: replace
        path: /spec/template/spec/containers/0/resources/limits/memory
        value: 4Gi
```

### Use External Database

Remove postgres.yaml from base kustomization and update database environment variables:

```yaml
# In deployment.yaml
- name: KC_DB_URL_HOST
  value: "your-external-db-host"
- name: KC_DB_URL_PORT
  value: "5432"
```

## Advanced Usage

### Create Custom Overlay

Create a new overlay directory:

```bash
mkdir -p kustomize/overlays/custom
```

Create `kustomize/overlays/custom/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: keycloak-custom

bases:
  - ../../base

commonLabels:
  environment: custom

patches:
  - target:
      kind: Deployment
      name: keycloak
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 4
  - target:
      kind: Ingress
      name: keycloak
    patch: |-
      - op: replace
        path: /spec/rules/0/host
        value: keycloak-custom.example.com
```

Deploy:

```bash
kubectl apply -k kustomize/overlays/custom
```

### Using ConfigMapGenerator

Add configuration via ConfigMap:

```yaml
configMapGenerator:
  - name: keycloak-config
    literals:
      - KC_LOG_LEVEL=INFO
      - KC_CACHE=ispn
      - KC_CACHE_STACK=kubernetes

patches:
  - target:
      kind: Deployment
      name: keycloak
    patch: |-
      - op: add
        path: /spec/template/spec/containers/0/envFrom
        value:
          - configMapRef:
              name: keycloak-config
```

### Using SecretGenerator

Generate secrets from files:

```yaml
secretGenerator:
  - name: keycloak-tls
    files:
      - tls.crt
      - tls.key
    type: kubernetes.io/tls
```

## Verification

### Check Deployment Status

```bash
# Check all keycloak namespaces
kubectl get pods -A | grep keycloak

# For base environment
kubectl get all -n keycloak

# For dev environment
kubectl get all -n keycloak-dev

# For staging environment
kubectl get all -n keycloak-staging

# For production environment
kubectl get all -n keycloak-prod
```

### Verify Health Endpoints

Keycloak 26.4.0 health endpoints run on port 9000 (management interface):

```bash
# Test from inside pod (curl not available in keycloak image)
kubectl exec -n keycloak deployment/keycloak -- \
  /bin/sh -c 'wget -qO- http://localhost:9000/health/ready'

# Or check probe status
kubectl get pods -n keycloak -o wide
# Should show 1/1 READY status
```

### View Logs

```bash
kubectl logs -n keycloak-prod -l app=keycloak --tail=100 -f
```

### Access Keycloak

#### Via Port Forward

```bash
kubectl port-forward -n keycloak-prod svc/keycloak-prod 8080:8080
```

Access: <http://localhost:8080>

#### Via Ingress

Access the configured domain (e.g., <https://keycloak.example.com>)

### Get Admin Credentials

```bash
# Username
kubectl get secret keycloak-admin-prod -n keycloak-prod \
  -o jsonpath="{.data.username}" | base64 --decode

# Password
kubectl get secret keycloak-admin-prod -n keycloak-prod \
  -o jsonpath="{.data.password}" | base64 --decode
```

## Update Deployment

### Update Image Version

Edit `kustomize/base/kustomization.yaml`:

```yaml
images:
  - name: quay.io/keycloak/keycloak
    newTag: "26.0.0"
```

Apply changes:

```bash
kubectl apply -k kustomize/overlays/production
```

### Scale Deployment

```bash
kubectl scale deployment keycloak-prod \
  --replicas=5 \
  --namespace keycloak-prod
```

## Rollback

### Using kubectl

```bash
kubectl rollout undo deployment/keycloak-prod -n keycloak-prod
```

### View Rollout History

```bash
kubectl rollout history deployment/keycloak-prod -n keycloak-prod
```

### Rollback to Specific Revision

```bash
kubectl rollout undo deployment/keycloak-prod \
  --to-revision=2 \
  -n keycloak-prod
```

## Delete Deployment

```bash
# Delete specific environment
kubectl delete -k kustomize/overlays/production

# Delete base deployment
kubectl delete -k kustomize/base
```

## Troubleshooting

### Check Pod Status

```bash
kubectl get pods -n keycloak-prod
kubectl describe pod <pod-name> -n keycloak-prod
```

### View Events

```bash
kubectl get events -n keycloak-prod --sort-by='.lastTimestamp'
```

### Debug Pod Issues

```bash
# Get detailed pod information
kubectl describe pod -n keycloak-prod <pod-name>

# Check logs
kubectl logs -n keycloak-prod <pod-name> --previous

# Execute commands in pod
kubectl exec -it -n keycloak-prod <pod-name> -- /bin/bash
```

### Database Connection Issues

```bash
# Test database connectivity from Keycloak pod
kubectl exec -it -n keycloak-prod <keycloak-pod> -- \
  curl -v telnet://postgres-service-prod:5432
```

## Production Best Practices

1. **Secrets Management**
   - Use external secret management (e.g., Sealed Secrets, External Secrets Operator)
   - Never commit actual passwords to Git

2. **Resource Management**
   - Set appropriate resource requests and limits
   - Use HorizontalPodAutoscaler for auto-scaling

3. **High Availability**
   - Run multiple replicas (minimum 3 for production)
   - Use pod anti-affinity rules

4. **Monitoring**
   - Enable metrics endpoint
   - Integrate with Prometheus/Grafana
   - Set up alerts

5. **Backup**
   - Regular database backups
   - Test restore procedures
   - Document backup strategy

6. **Security**
   - Enable TLS/HTTPS
   - Use network policies
   - Regular security updates
   - Implement RBAC

## GitOps Integration

### ArgoCD

Create an Application manifest:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: keycloak-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/keycloak-ops
    targetRevision: main
    path: kustomize/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: keycloak-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Flux CD

```bash
flux create kustomization keycloak-prod \
  --source=GitRepository/keycloak-ops \
  --path="./kustomize/overlays/production" \
  --prune=true \
  --interval=5m
```

## Further Resources

- [Kustomize Documentation](https://kustomize.io/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Keycloak Documentation](https://www.keycloak.org/documentation)
