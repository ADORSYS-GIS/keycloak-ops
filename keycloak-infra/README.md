# Keycloak Deployment with Bitnami Helm Charts

Complete production-ready Keycloak deployment on Kubernetes using Bitnami Helm charts with PostgreSQL database.

## 📋 Table of Contents

- [Keycloak Deployment with Bitnami Helm Charts](#keycloak-deployment-with-bitnami-helm-charts)
  - [📋 Table of Contents](#-table-of-contents)
  - [Overview](#overview)
  - [Why Bitnami Charts?](#why-bitnami-charts)
  - [Prerequisites](#prerequisites)
  - [Project Structure](#project-structure)
  - [Deployment Options](#deployment-options)
    - [Option 1: Helm Deployment (Recommended)](#option-1-helm-deployment-recommended)
    - [Option 2: Kustomize Deployment](#option-2-kustomize-deployment)
  - [Quick Start (Helm)](#quick-start-helm)
  - [Verify It Works](#verify-it-works)
  - [Next Steps](#next-steps)
  - [Documentation](#documentation)

## Overview

This project provides a complete Keycloak Identity and Access Management (IAM) solution deployed on Kubernetes using:

- **Keycloak 26.3.3** - Modern authentication and authorization server
- **PostgreSQL 17.6.0** - Production-ready database backend
- **Bitnami Helm Charts** - Battle-tested, enterprise-ready Helm charts
- **Kind Cluster** - Local Kubernetes cluster for development and testing

⏱️ **Deployment Time**:

- First deployment: 15-20 minutes (includes image downloads)
- Subsequent deployments: 2-3 minutes (images cached)

## Why Bitnami Charts?

Bitnami Helm charts provide production-ready Kubernetes deployments with minimal configuration:

**Key Benefits**:

- ✅ **Production-Ready**: Pre-configured security, health checks, and secret management
- ✅ **Zero Configuration**: PostgreSQL integration and networking configured automatically
- ✅ **Enterprise Support**: Actively maintained by VMware/Broadcom with regular updates
- ✅ **Time Savings**: Deploy in 30 minutes vs 2-3 days writing custom charts
- ✅ **Comprehensive Features**: Metrics, ingress, TLS, HA, and backup capabilities included

**Estimated effort**: 40+ hours to build custom charts vs 2 hours with Bitnami

## Prerequisites

- **Docker** - For running kind cluster
- **kubectl** (v1.23+)
- **Helm** (v3.8.0+)
- **kind** - Kubernetes in Docker (for local development)
- **4GB+ RAM** available for the cluster

## Project Structure

```plaintext
keycloak-infra/
├── README.md                          # Project overview and quick start
├── TECHNICAL_GUIDE.md                 # Detailed troubleshooting and configuration
├── CHANGELOG.md                       # Version history
└── keycloak-infra/
    ├── keycloak/                      # Helm deployment
    │   ├── Chart.yaml                 # Helm chart metadata
    │   ├── Chart.lock                 # Dependency lock file
    │   ├── values.yaml                # Default configuration
    │   ├── values-dev.yaml            # Development overrides
    │   ├── charts/                    # Dependency charts (PostgreSQL, common)
    │   └── templates/                 # Kubernetes manifests
    └── kustomize/                     # Kustomize deployment (alternative)
        ├── base/                      # Base manifests
        ├── overlays/dev/              # Development overlay
        ├── overlays/prod/             # Production overlay
        └── README.md                  # Kustomize documentation
```

## Deployment Options

Choose between **Helm** (recommended for production) or **Kustomize** (simpler, GitOps-friendly):

### Option 1: Helm Deployment (Recommended)

Best for: Production deployments, complex configurations, dependency management

### Option 2: Kustomize Deployment

Best for: Simple deployments, GitOps workflows, learning Kubernetes

See [kustomize/README.md](keycloak-infra/kustomize/README.md) for Kustomize instructions.

## Quick Start (Helm)

Get Keycloak running in 3 commands:

```bash
# 1. Create cluster
kind create cluster --name keycloak-demo

# 2. Deploy Keycloak
cd keycloak-infra
helm install my-keycloak ./keycloak --namespace keycloak --create-namespace

# 3. Wait and watch (pods will take 15-20 minutes on first run)
kubectl get pods -n keycloak -w
```

**Access Keycloak**:

```bash
# Get admin credentials
echo "Username: user"
kubectl get secret my-keycloak -n keycloak -o jsonpath="{.data.admin-password}" | base64 -d
echo

# Port forward to access
kubectl port-forward -n keycloak svc/my-keycloak 8080:80
```

Open browser: **http://localhost:8080**

**Clean Up**:

```bash
# Delete cluster
kind delete cluster --name keycloak-demo
```

## Verify It Works

Once pods are running (1/1 Ready status):

1. Access the Keycloak admin console at **http://localhost:8080**
2. Click **"Administration Console"**
3. Login with credentials from above
4. You should see the Keycloak admin dashboard
5. **Test**: Navigate to **Clients** → You'll see default clients
6. **Test**: Navigate to **Users** → Click "Add user" to test user creation

## Next Steps

1. **Create a Realm**: Admin Console → Create Realm → "my-app"
2. **Add a Client**: Configure OAuth2/OIDC for your application
3. **Add Users**: Create test users in your realm
4. **Customize**: Change themes, configure login flows
5. **Development**: Use `values-dev.yaml` for fixed credentials (`admin/admin123`)

## Documentation

- **[TECHNICAL_GUIDE.md](docs/TECHNICAL_GUIDE.md)** - Comprehensive troubleshooting, configuration details, and production deployment guide
- **[keycloak-infra/kustomize/README.md](keycloak-infra/kustomize/README.md)** - Kustomize deployment guide
- **[CHANGELOG.md](docs/CHANGELOG.md)** - Version history and changes
- **[Bitnami Keycloak Chart](https://github.com/bitnami/charts/tree/main/bitnami/keycloak)** - Official Bitnami chart documentation

---

**Quick Troubleshooting**:

- Pods stuck? Images are large (~800MB), first pull takes 15-20 minutes
- CrashLoopBackOff? Clean up PVCs and secrets, then reinstall
- Port 8080 in use? Use `kubectl port-forward -n keycloak svc/my-keycloak 9090:80`

For detailed troubleshooting, see [TECHNICAL_GUIDE.md](docs/TECHNICAL_GUIDE.md).
