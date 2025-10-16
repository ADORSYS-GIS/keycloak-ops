# Security & Production Checklist

## 🔴 Critical - Required for Production

### 1. Secure Credentials
**Current:** Default passwords (`admin123`, `keycloak123`) in [kustomize/base/secrets.yaml](cci:7://file:///home/guy_ghis/projects/keycloak_projects/keycloak-ops/kustomize/base/secrets.yaml:0:0-0:0)

**Fix:** Use [./create-secrets.sh](cci:7://file:///home/guy_ghis/projects/keycloak_projects/keycloak-ops/create-secrets.sh:0:0-0:0) or external secret management

### 2. Persistent Database  
**Current:** Ephemeral storage (data lost on restart)

**Fix:** Enable PVCs or use managed database (RDS, Cloud SQL)

### 3. TLS/HTTPS
**Current:** HTTP only

**Fix:** Install cert-manager and configure ingress TLS

### 4. Production Mode
**Current:** Using `start-dev` command

**Fix:** Change to `start --optimized` in deployment.yaml

## ✅ Already Configured

- Health probes on port 9000
- Non-root containers  
- Security capabilities dropped
- Pod anti-affinity
- Prometheus metrics

## Production Checklist

- [ ] Change all default passwords
- [ ] Enable persistent storage
- [ ] Configure TLS/HTTPS
- [ ] Switch to production mode
- [ ] Set up monitoring
- [ ] Configure backups
- [ ] Test disaster recovery

See full security guide at https://www.keycloak.org/docs/latest/server_installation/
