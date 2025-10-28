# Changelog

All notable changes to this Keycloak deployment project.

## [1.0.0] - 2025-10-22

### Added
- Initial Keycloak 26.3.3 deployment using Bitnami Helm chart v25.2.0
- PostgreSQL 17.6.0 as bundled database backend
- Development configuration (values-dev.yaml)
- Comprehensive documentation (README, QUICKSTART, NOTES)
- Kind cluster support for local development

### Changed
- **Critical**: Migrated to bitnamilegacy images (Bitnami August 2025 migration)
- Set global.security.allowInsecureImages: true

### Configuration
- Keycloak: docker.io/bitnamilegacy/keycloak:26.3.3-debian-12-r0
- PostgreSQL: docker.io/bitnamilegacy/postgresql:17.6.0-debian-12-r0
- Config CLI: docker.io/bitnamilegacy/keycloak-config-cli:6.4.0-debian-12-r11
- Default admin user: user (password auto-generated)

### Known Issues
1. First deployment: 15-20 minutes (large images)
2. Security warning expected (bitnamilegacy images)
3. Password mismatch on redeploy (delete PVCs + secrets together)

### Documentation
- Complete troubleshooting guide
- Image selection rationale
- Development vs production guidance
- Maintenance procedures
