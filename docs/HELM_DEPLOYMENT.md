# Keycloak Helm Deployment Guide

## New Template Files Required

To complete the Helm chart setup, create the following new template files:

### 1. NetworkPolicy Template

**File:** `helm/keycloak/templates/networkpolicy.yaml`

This template provides network-level security by restricting ingress and egress traffic.

### 2. ConfigMap Template

**File:** `helm/keycloak/templates/configmap.yaml`

This template allows mounting custom configuration, themes, or realm definitions.

See the "Additional Template Files" section at the end of this document for complete file contents.

## Quick Start Commands

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

### 1. Create Namespace

```bash
kubectl create namespace keycloak
```

### 2. Deploy with Default Values (includes PostgreSQL)

```bash
helm install keycloak ./helm/keycloak \
  --namespace keycloak \
  --set keycloak.admin.password=YourSecureAdminPassword
```

### 3. Deploy without Embedded PostgreSQL (using external database)

```bash
helm install keycloak ./helm/keycloak \
  --namespace keycloak \
  --set keycloak.admin.password=YourSecureAdminPassword \
  --set postgresql.enabled=false \
  --set keycloak.database.host=your-postgres-host \
  --set keycloak.database.password=YourDbPassword
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

### Pods Not Starting

```bash
# Describe pod
kubectl describe pod -n keycloak <pod-name>

# Check logs
kubectl logs -n keycloak <pod-name>
```

### Database Connection Issues

```bash
# Test database connectivity
kubectl run -it --rm debug --image=postgres:15-alpine --restart=Never -- \
  psql -h <db-host> -U keycloak -d keycloak
```

### Ingress Not Working

```bash
# Check ingress controller
kubectl get pods -n ingress-nginx

# Check ingress events
kubectl describe ingress keycloak -n keycloak
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

---

## Additional Template Files

Create these new template files to complete the Helm chart:

### NetworkPolicy Template

**File:** `helm/keycloak/templates/networkpolicy.yaml`

```yaml
{{- if .Values.networkPolicy.enabled -}}
# NetworkPolicy restricts network access to Keycloak pods
# This implements a least-privilege network security model
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: {{ include "keycloak.fullname" . }}
  labels:
    {{- include "keycloak.labels" . | nindent 4 }}
spec:
  # Apply this policy to Keycloak pods
  podSelector:
    matchLabels:
      {{- include "keycloak.selectorLabels" . | nindent 6 }}
  
  # Define allowed traffic types
  policyTypes:
    - Ingress
    - Egress
  
  # Ingress rules: who can connect to Keycloak
  ingress:
    # Allow traffic from ingress controller
    {{- if .Values.networkPolicy.ingress.enabled }}
    - from:
        {{- if .Values.networkPolicy.ingress.namespaceSelector }}
        - namespaceSelector:
            {{- toYaml .Values.networkPolicy.ingress.namespaceSelector | nindent 12 }}
        {{- end }}
        {{- if .Values.networkPolicy.ingress.podSelector }}
        - podSelector:
            {{- toYaml .Values.networkPolicy.ingress.podSelector | nindent 12 }}
        {{- end }}
      ports:
        - protocol: TCP
          port: 8080
        - protocol: TCP
          port: 8443
    {{- end }}
    
    # Allow inter-pod communication for clustering (JGroups)
    - from:
        - podSelector:
            matchLabels:
              {{- include "keycloak.selectorLabels" . | nindent 14 }}
      ports:
        - protocol: TCP
          port: 7800
        - protocol: TCP
          port: 8080
  
  # Egress rules: where Keycloak can connect to
  egress:
    # Allow DNS resolution
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
        - podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    
    # Allow connection to PostgreSQL database
    {{- if .Values.networkPolicy.egress.database.enabled }}
    - to:
        {{- if .Values.networkPolicy.egress.database.namespaceSelector }}
        - namespaceSelector:
            {{- toYaml .Values.networkPolicy.egress.database.namespaceSelector | nindent 12 }}
        {{- end }}
        {{- if .Values.networkPolicy.egress.database.podSelector }}
        - podSelector:
            {{- toYaml .Values.networkPolicy.egress.database.podSelector | nindent 12 }}
        {{- end }}
      ports:
        - protocol: TCP
          port: {{ include "keycloak.dbPort" . }}
    {{- end }}
    
    # Allow HTTPS for external integrations (LDAP, OIDC providers, etc.)
    {{- if .Values.networkPolicy.egress.external.enabled }}
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: TCP
          port: 443
        - protocol: TCP
          port: 636  # LDAPS
    {{- end }}
    
    # Additional custom egress rules
    {{- with .Values.networkPolicy.egress.custom }}
    {{- toYaml . | nindent 4 }}
    {{- end }}
{{- end }}
```

### ConfigMap Template

**File:** `helm/keycloak/templates/configmap.yaml`

```yaml
{{- if .Values.configMap.enabled -}}
# ConfigMap for custom Keycloak configuration
# Use this to provide custom themes, providers, or realm configurations
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "keycloak.fullname" . }}-config
  labels:
    {{- include "keycloak.labels" . | nindent 4 }}
data:
  {{- with .Values.configMap.data }}
  {{- toYaml . | nindent 2 }}
  {{- end }}
{{- end }}
```

### Usage Examples

#### Enable NetworkPolicy

In your `values.yaml` or custom values file:

```yaml
networkPolicy:
  enabled: true
  ingress:
    enabled: true
    namespaceSelector:
      matchLabels:
        name: ingress-nginx
  egress:
    database:
      enabled: true
      podSelector:
        matchLabels:
          app: postgres
    external:
      enabled: true
```

#### Use ConfigMap for Custom Realm

In your `values.yaml` or custom values file:

```yaml
configMap:
  enabled: true
  data:
    realm.json: |
      {
        "realm": "myrealm",
        "enabled": true,
        "displayName": "My Realm",
        "loginTheme": "base"
      }
  mountPath: /opt/keycloak/data/import
```

Then update the deployment args to import the realm:

```yaml
keycloak:
  extraEnv:
    - name: KEYCLOAK_IMPORT
      value: /opt/keycloak/data/import/realm.json
```
