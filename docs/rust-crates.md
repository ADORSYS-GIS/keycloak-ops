# Rust Crates for Keycloak - Quick Reference Table

## Complete Crates Comparison Table

| Crate | Category / Use | Purpose | Integration Role with Keycloak | Pros / Strengths | Limitations / Notes |
|-------|---|---|---|---|---|
| **jsonwebtoken** | JWT Validation / Decoding | Create, decode, and verify JSON Web Tokens (JWTs) using RSA, HMAC, or ECDSA. | Validates Keycloak's access tokens or ID tokens on resource servers. | Mature, actively maintained, supports all JOSE algorithms. | Manual key rotation; you must fetch JWKS and handle caching. |
| **reqwest** | HTTP Client | Async/sync HTTP client for making external web requests. | Used to fetch Keycloak's OIDC discovery document and JWKS (public keys). | Easy to use, integrates with tokio, supports async, TLS, and redirects. | Larger binary size; slower than lightweight clients for small apps. |
| **serde / serde_json** | Data Parsing / Serialization | Define Rust structs to serialize/deserialize JSON data. | Used to parse Keycloak discovery metadata, user claims, or configuration. | Essential for all Keycloak JSON payloads; stable and widely used. | None significant — it's the standard. |
| **jwt-simple** | Simplified JWT Handling | A lightweight, non-opinionated JWT library for signing and verifying tokens. | Alternative to jsonwebtoken for simpler token workflows. | Very small dependency footprint; simple API. | Doesn't support all advanced JOSE algorithms like RSA+PS. |
| **async-oidc-jwt-validator** | Token Validation / Discovery | Validates tokens from any OIDC provider with JWKS fetching & caching. | Validates and caches Keycloak's signing keys automatically. | Great for backend services needing scalable validation. | Slightly lower ecosystem adoption. |
| **axum-keycloak-auth** | Axum Middleware | Provides authentication/authorization middleware for Axum. | Secures Axum routes using Keycloak-issued JWTs; validates roles and claims. | Automatic OIDC discovery; role extraction; minimal setup. | Framework-specific (Axum only). |
| **actix-web-middleware-keycloak-auth** | Actix Middleware | Middleware for Keycloak JWT validation in Actix Web. | Simplifies protecting routes with Keycloak roles/claims. | Native Actix integration; handles passthrough behavior. | Less flexible for custom claim extraction. |
| **leptos-keycloak-auth** | Full-stack Auth (Leptos) | Integrates Keycloak authentication with Leptos frontend + SSR. | Handles login, logout, token refresh, and session management. | Ideal for full-stack web apps needing OIDC flows. | Adds client-side dependencies. |
| **actix-web** | Web Framework Core | Core framework for building web APIs (like Actix). | Base for implementing manual JWT validation. | Mature, stable, performant for backend APIs. | Needs manual Keycloak logic if not using middleware. |
| **josekit** | Low-Level JOSE Standards | Implements JWS/JWE/JWK primitives. | Provides cryptographic primitives for verifying Keycloak tokens directly. | Fine-grained control; supports all JOSE operations. | More complex; low-level API. |
| **loco-keycloak-auth** | Framework Integration | Adds Keycloak authentication layer to Loco apps. | Enables config-based protection of routes in Loco.rs. | Declarative setup, minimal boilerplate. | Tightly coupled to Loco framework. |
| **warp-jwt** | Web Auth Middleware | JWT validation middleware for the Warp web framework. | Can validate tokens issued by Keycloak with correct configuration. | Simple integration for Warp; lightweight. | Limited advanced OIDC discovery support. |
| **axum-login** | Session/Auth Framework | Framework-agnostic login management crate for Axum. | Can integrate with Keycloak's ID tokens for session-based apps. | Manages user sessions easily. | Doesn't fetch or validate tokens automatically. |
| **tracing / tracing-subscriber** | Logging & Observability | Structured logging and event tracing. | Useful for debugging OIDC requests, token validation, and API calls. | Modern, async-friendly logging ecosystem. | None significant. |
| **url / urlencoding** | URL Handling | URL parsing and manipulation. | Used to build OIDC redirect URIs, token endpoints, etc. | Lightweight, fast. | Must ensure correct URL encoding for query params. |

---

## Simplified Category View

### JWT & Token Handling

| Crate | Purpose | Best For | Complexity |
|-------|---------|----------|-----------|
| **jsonwebtoken** | Full-featured JWT validation | Production token validation | Medium |
| **jwt-simple** | Lightweight JWT handling | Simple workflows | Low |
| **async-oidc-jwt-validator** | Auto-discovery & caching | Microservices | Low |
| **josekit** | Low-level JOSE operations | Custom cryptography | High |

### HTTP & Network

| Crate | Purpose | Best For | Complexity |
|-------|---------|----------|-----------|
| **reqwest** | Async HTTP client | API calls & discovery | Low |

### Data Serialization

| Crate | Purpose | Best For | Complexity |
|-------|---------|----------|-----------|
| **serde / serde_json** | JSON serialization | All JSON operations | Low |

### Web Frameworks & Middleware

| Crate | Purpose | Best For | Complexity |
|-------|---------|----------|-----------|
| **axum-keycloak-auth** | Axum authentication | Axum web apps | Low |
| **actix-web-middleware-keycloak-auth** | Actix authentication | Actix web apps | Low |
| **warp-jwt** | Warp authentication | Warp microservices | Low |
| **actix-web** | Core web framework | Backend APIs | Medium |

### Full-Stack Solutions

| Crate | Purpose | Best For | Complexity |
|-------|---------|----------|-----------|
| **leptos-keycloak-auth** | Full-stack auth | Leptos SSR apps | Medium |
| **loco-keycloak-auth** | Loco framework auth | Loco web apps | Low |
| **axum-login** | Session management | Session-based apps | Medium |

### Utilities & Observability

| Crate | Purpose | Best For | Complexity |
|-------|---------|----------|-----------|
| **tracing / tracing-subscriber** | Structured logging | Debugging & monitoring | Low |
| **url / urlencoding** | URL handling | OIDC URI construction | Low |

---

## Feature Comparison Matrix

| Feature | jsonwebtoken | jwt-simple | async-oidc-jwt-validator | josekit | reqwest | serde_json |
|---------|---|---|---|---|---|---|
| **JWT Validation** | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| **Async Support** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **OIDC Discovery** | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |
| **JWKS Caching** | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |
| **All JOSE Algorithms** | ✅ | ⚠️ | ✅ | ✅ | ❌ | ❌ |
| **Low Dependency** | ⚠️ | ✅ | ⚠️ | ⚠️ | ⚠️ | ✅ |
| **Production Ready** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

| Feature | axum-keycloak-auth | actix-web-middleware-keycloak-auth | leptos-keycloak-auth | loco-keycloak-auth | warp-jwt |
|---------|---|---|---|---|---|
| **Framework Integration** | Axum | Actix | Leptos | Loco | Warp |
| **Automatic Discovery** | ✅ | ⚠️ | ✅ | ✅ | ❌ |
| **Role Extraction** | ✅ | ✅ | ✅ | ✅ | ⚠️ |
| **Session Management** | ❌ | ❌ | ✅ | ⚠️ | ❌ |
| **Minimal Setup** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Production Ready** | ✅ | ✅ | ✅ | ✅ | ✅ |

---

## Use Case Selection Table

### Backend API Protection

| Need | Recommended Crate | Reason |
|------|-------------------|--------|
| Token validation | `jsonwebtoken` | Mature, all algorithms |
| HTTP requests | `reqwest` | Async, reliable |
| JSON parsing | `serde_json` | Standard, fast |
| Logging | `tracing` | Structured, async-friendly |

### Axum Web Framework

| Need | Recommended Crate | Reason |
|------|-------------------|--------|
| Authentication | `axum-keycloak-auth` | Built-in middleware |
| Sessions | `axum-login` | Session management |
| Logging | `tracing` | Framework-native |

### Actix Web Framework

| Need | Recommended Crate | Reason |
|------|-------------------|--------|
| Authentication | `actix-web-middleware-keycloak-auth` | Native integration |
| Core framework | `actix-web` | Mature, performant |
| Logging | `tracing` | Async-friendly |

### Full-Stack Leptos App

| Need | Recommended Crate | Reason |
|------|-------------------|--------|
| Full-stack auth | `leptos-keycloak-auth` | Handles OIDC flows |
| JSON parsing | `serde_json` | Standard serialization |
| Logging | `tracing` | Observability |

### Microservices

| Need | Recommended Crate | Reason |
|------|-------------------|--------|
| Token validation | `async-oidc-jwt-validator` | Auto-discovery & caching |
| HTTP requests | `reqwest` | Lightweight, async |
| Logging | `tracing` | Distributed tracing |

### Custom Integration

| Need | Recommended Crate | Reason |
|------|-------------------|--------|
| HTTP client | `reqwest` | Flexible, async |
| Token validation | `jsonwebtoken` | Full control |
| JOSE operations | `josekit` | Low-level primitives |
| JSON parsing | `serde_json` | Standard |
| Logging | `tracing` | Comprehensive |

---

## Maturity & Adoption Levels

### ⭐⭐⭐⭐⭐ Highly Mature & Widely Adopted
- `jsonwebtoken` - Industry standard for JWT
- `reqwest` - De facto HTTP client
- `serde_json` - Standard serialization
- `tracing` - Modern logging standard
- `url` - URL handling standard

### ⭐⭐⭐⭐ Mature & Well-Adopted
- `jwt-simple` - Lightweight alternative
- `async-oidc-jwt-validator` - Growing adoption
- `josekit` - Comprehensive JOSE support
- `axum-keycloak-auth` - Axum ecosystem
- `actix-web-middleware-keycloak-auth` - Actix ecosystem
- `warp-jwt` - Warp ecosystem

### ⭐⭐⭐ Established & Stable
- `leptos-keycloak-auth` - Full-stack solution
- `loco-keycloak-auth` - Loco framework
- `axum-login` - Session management

---

## Performance Characteristics

| Crate | Memory Footprint | Speed | Notes |
|-------|------------------|-------|-------|
| `jsonwebtoken` | Low | Fast | Cached keys recommended |
| `jwt-simple` | Very Low | Very Fast | Minimal overhead |
| `async-oidc-jwt-validator` | Medium | Fast | Includes caching |
| `josekit` | Low | Medium | Fine-grained control |
| `reqwest` | Medium | Medium | Connection pooling helps |
| `serde_json` | Low | Fast | Industry standard |
| `tracing` | Low | Fast | Negligible overhead |
| `url` | Very Low | Very Fast | Lightweight parsing |

---

## Decision Tree

```
Start: Need Keycloak integration?
│
├─ Need JWT validation?
│  ├─ Simple workflow? → jwt-simple
│  ├─ Production, all algorithms? → jsonwebtoken
│  ├─ Auto-discovery & caching? → async-oidc-jwt-validator
│  └─ Low-level JOSE? → josekit
│
├─ Need web framework integration?
│  ├─ Using Axum? → axum-keycloak-auth
│  ├─ Using Actix? → actix-web-middleware-keycloak-auth
│  ├─ Using Warp? → warp-jwt
│  ├─ Using Leptos? → leptos-keycloak-auth
│  └─ Using Loco? → loco-keycloak-auth
│
├─ Need HTTP client?
│  └─ → reqwest (async/sync)
│
├─ Need JSON parsing?
│  └─ → serde_json
│
├─ Need logging?
│  └─ → tracing / tracing-subscriber
│
└─ Need URL handling?
   └─ → url / urlencoding
```

---

## Quick Start Stacks

### Stack 1: Minimal Backend API
```toml
[dependencies]
reqwest = { version = "0.12", features = ["json"] }
jsonwebtoken = "9.0"
serde_json = "1.0"
tokio = { version = "1", features = ["full"] }
```

### Stack 2: Axum Web App
```toml
[dependencies]
axum = "0.7"
axum-keycloak-auth = "0.1"
tokio = { version = "1", features = ["full"] }
tracing = "0.1"
tracing-subscriber = "0.3"
```

### Stack 3: Actix Web App
```toml
[dependencies]
actix-web = "4.0"
actix-web-middleware-keycloak-auth = "0.1"
tracing = "0.1"
tracing-subscriber = "0.3"
```

### Stack 4: Full-Stack Leptos
```toml
[dependencies]
leptos = "0.5"
leptos-keycloak-auth = "0.1"
tokio = { version = "1", features = ["full"] }
```

### Stack 5: Microservices
```toml
[dependencies]
async-oidc-jwt-validator = "0.1"
reqwest = { version = "0.12", features = ["json"] }
tracing = "0.1"
tracing-subscriber = "0.3"
```

---

## Troubleshooting Guide

| Problem | Likely Cause | Solution |
|---------|-------------|----------|
| Token validation fails | Invalid key format | Use `jsonwebtoken` with correct algorithm |
| JWKS not cached | Manual caching needed | Use `async-oidc-jwt-validator` |
| Large binary size | Too many dependencies | Use `jwt-simple` instead of `jsonwebtoken` |
| URL encoding issues | Incorrect encoding | Use `urlencoding::encode()` |
| Async runtime errors | Missing tokio features | Add `tokio = { version = "1", features = ["full"] }` |
| Logging not appearing | Tracing not initialized | Call `tracing_subscriber::fmt::init()` |

---

## Conclusion

Choose based on your needs:

- **Simple token validation** → `jsonwebtoken` + `reqwest`
- **Framework integration** → Framework-specific middleware crate
- **Microservices** → `async-oidc-jwt-validator`
- **Full-stack app** → `leptos-keycloak-auth`
- **Custom logic** → `josekit` + `jsonwebtoken`

All crates are production-ready. Combine them based on your architecture!
