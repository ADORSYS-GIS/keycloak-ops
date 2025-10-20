# Keycloak Helm Chart

A Helm chart for deploying Keycloak on Kubernetes with PostgreSQL support.

## Requirements

- Kubernetes 1.24+
- Helm 3.18+

## Features

- Production-ready Keycloak deployment
- Built-in PostgreSQL database (optional)
- High availability with multiple replicas
- Ingress support with TLS
- Health checks and startup probes
- Resource management and autoscaling
- Security best practices

## Installation

**⚠️ Important:** This Helm chart does NOT include PostgreSQL by default (`postgresql.enabled=false` in values.yaml). You must deploy PostgreSQL separately or use an external database.

### Quick Install (with Bundled PostgreSQL)

For local testing with the bundled PostgreSQL from Kustomize:

```bash
# Step 1: Create namespace
kubectl create namespace keycloak

# Step 2: Deploy PostgreSQL from Kustomize base
kubectl apply -f ../../kustomize/base/postgres.yaml

# Step 3: Wait for PostgreSQL to be ready
kubectl wait --for=condition=ready pod -l app=postgres -n keycloak --timeout=300s

# Step 4: Install Keycloak with Helm
# ⚠️ CRITICAL: database password MUST match postgres password (keycloak123)
helm install keycloak . \
  --namespace keycloak \
  --set keycloak.admin.password=admin123 \
  --set keycloak.database.password=keycloak123

# Step 5: Watch deployment
kubectl get pods -n keycloak -w
```

**Default credentials:**

- Username: `admin`
- Password: `admin123`

Access via port-forward: `kubectl port-forward -n keycloak svc/keycloak 8080:8080`

### Install with External Database (Production)

```bash
helm install keycloak . \
  --namespace keycloak \
  --create-namespace \
  --set keycloak.admin.password=YourSecurePassword \
  --set postgresql.enabled=false \
  --set keycloak.database.host=your-postgres-host.example.com \
  --set keycloak.database.password=YourDbPassword
```

### Install with Custom Values

```bash
helm install keycloak . \
  --namespace keycloak \
  --create-namespace \
  --values values-custom.yaml
```

## Configuration

See [values.yaml](values.yaml) for all configuration options.

### Key Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicaCount` | Number of Keycloak replicas | `2` |
| `image.repository` | Keycloak image repository | `quay.io/keycloak/keycloak` |
| `image.tag` | Keycloak image tag | `26.4.0` |
| `keycloak.admin.username` | Admin username | `admin` |
| `keycloak.admin.password` | Admin password | _Required_ |
| `keycloak.database.vendor` | Database vendor | `postgres` |
| `keycloak.database.host` | Database host | `postgres-service` |
| `keycloak.database.password` | Database password | _Required_ |
| `postgresql.enabled` | Deploy PostgreSQL subchart | `false` ⚠️ |
| `ingress.enabled` | Enable ingress | `true` |
| `ingress.hosts[0].host` | Ingress hostname | `keycloak.example.com` |

**Note:** `postgresql.enabled=false` by default. Deploy PostgreSQL separately using `kubectl apply -f ../../kustomize/base/postgres.yaml` or use an external database.

## Upgrading

```bash
helm upgrade keycloak ./helm/keycloak \
  --namespace keycloak \
  --values values-custom.yaml
```

## Uninstalling

```bash
helm uninstall keycloak --namespace keycloak
```

## Documentation

For detailed deployment instructions, see:

- [Helm Deployment Guide](../../docs/HELM_DEPLOYMENT.md)
- [Official Keycloak Documentation](https://www.keycloak.org/documentation)

## License

Apache 2.0
