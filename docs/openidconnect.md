# 🔐 OpenID Connect Crate Integration with Keycloak

> 🔑 **Need pure OAuth 2.0 authorization?** Check out our [**OAuth2 documentation**](./oauth2.md) for API access tokens and service-to-service authentication.

## 📦 Overview

The `openidconnect` crate is a comprehensive, strongly-typed OpenID Connect (OIDC) client library for Rust applications. Built on top of OAuth 2.0, it provides **authentication** capabilities alongside authorization, making it the ideal choice for user login systems and identity management with Keycloak.

**Purpose in Keycloak Ecosystem:**
- Primary OpenID Connect client integration library for Rust applications
- Enables secure **user authentication** and **identity verification** with Keycloak
- Handles complete OIDC flow including ID tokens, userinfo, and discovery
- Provides type-safe abstractions for OpenID Connect protocol interactions
- **Automatic JWT validation** and **OIDC discovery** built-in

**Implementation Status:**
- **Version:** 4.0.1 (Latest stable)
- **GitHub Stars:** 500+ ⭐
- **Contributors:** 60+ active collaborators
- **Last Commit:** Actively maintained (2025)
- **Built-in OIDC Discovery:** Automatic endpoint configuration
- **ID Token Validation:** Complete JWT verification with issuer validation
- **License:** MIT/Apache-2.0 dual license

## 🚀 Supported OpenID Connect Flows

| Flow | Status | Description |
|------|--------|-------------|
| **Authorization Code Flow** | ✅ **Recommended** | Standard OIDC flow (RFC 6749 + OIDC Core). Most secure for web applications. |
| **Authorization Code + PKCE** | ✅ **Best Practice** | Enhanced security with Proof Key for Code Exchange. Ideal for SPAs and mobile apps. |
| **Implicit Flow** | ✅ **Legacy Support** | Direct ID token flow. Supported but discouraged for new applications. |
| **Hybrid Flow** | ✅ **Advanced** | Combination of code and implicit flows for complex scenarios. |
| **Client Credentials** | ✅ **Service-to-Service** | Machine-to-machine with service account authentication. |
| **Device Authorization** | ✅ **IoT/CLI** | For devices without browsers (RFC 8628). |

## 🧰 What You Can Build

Using this crate, you can easily implement:

**✅ User Authentication Systems:**
- Web applications with secure user login
- Single Sign-On (SSO) integration with Keycloak
- Multi-tenant applications with identity federation
- Mobile apps with secure authentication flows

**✅ Identity Management:**
- User profile management and claims extraction
- Role-based access control (RBAC) integration
- Social login integration (Google, GitHub, etc. via Keycloak)
- Session management and logout flows

**✅ Advanced OIDC Features:**
- Automatic OIDC discovery from `.well-known/openid-configuration`
- ID token validation with issuer and audience verification
- Userinfo endpoint integration for profile data
- Token refresh and lifecycle management
- Custom claims processing and validation

## ⚙️ Key Differences from OAuth2 Crate

| Feature | `oauth2` Crate | `openidconnect` Crate |
|---------|----------------|----------------------|
| **Protocol** | Pure OAuth 2.0 | OpenID Connect (extends OAuth 2.0) |
| **Primary Use** | Authorization (API access) | Authentication (user identity) |
| **Token Types** | Access tokens only | Access + ID tokens |
| **Discovery** | Manual endpoint configuration | Automatic OIDC discovery |
| **JWT Validation** | Manual implementation required | Built-in ID token validation |
| **User Info** | Manual API calls | Built-in userinfo endpoint support |
| **Complexity** | Simpler, more flexible | More opinionated, OIDC-compliant |

## 🔧 Core Implementation Features

**✅ Built-in OIDC Discovery:**
- **Automatic Configuration:** Fetches endpoints from `/.well-known/openid-configuration`
- **Dynamic Provider Setup:** No manual endpoint configuration needed
- **Issuer Validation:** Automatic verification of token issuer
- **JWKS Integration:** Automatic public key fetching for token validation

**✅ ID Token Management:**
- **Complete JWT Validation:** Signature, issuer, audience, expiration verification
- **Claims Extraction:** Standard and custom claims processing
- **Nonce Validation:** CSRF protection for authentication flows
- **Token Lifecycle:** Automatic validation and refresh handling

**✅ Enhanced Security:**
- **PKCE Support:** Built-in Proof Key for Code Exchange
- **State Parameter:** CSRF protection for authorization flows
- **Nonce Handling:** Replay attack prevention
- **Secure Defaults:** OIDC-compliant security configurations

## 🏗️ Architecture & Integration Patterns

### Discovery-First Approach
```rust
// Automatic OIDC discovery
let provider_metadata = CoreProviderMetadata::discover_async(
    IssuerUrl::new("https://keycloak.example.com/realms/master".to_string())?,
    async_http_client,
).await?;

// Client automatically configured with all endpoints
let client = CoreClient::from_provider_metadata(
    provider_metadata,
    ClientId::new("my-client".to_string()),
    Some(ClientSecret::new("client-secret".to_string())),
);
```

### ID Token Validation
```rust
// Automatic ID token validation with all security checks
let id_token_claims: CoreIdTokenClaims = client
    .exchange_code(authorization_code)
    .set_pkce_verifier(pkce_verifier)
    .request_async(async_http_client)
    .await?
    .id_token()
    .ok_or("No ID token")?
    .claims(&client.id_token_verifier(), &nonce)?;

// Extract user information
let user_id = id_token_claims.subject().as_str();
let email = id_token_claims.email().map(|e| e.as_str());
```

## 🎯 Use Case Scenarios

**✅ Web Application Login:**
- User clicks "Login with Keycloak"
- Redirect to Keycloak authorization endpoint
- User authenticates and consents
- Receive authorization code + exchange for tokens
- Validate ID token and extract user identity
- Create application session

**✅ API Gateway Authentication:**
- Validate incoming ID tokens from client applications
- Extract user claims for authorization decisions
- Integrate with Keycloak for user management
- Handle token refresh and session management

**✅ Microservices Identity:**
- Service-to-service authentication with client credentials
- User context propagation between services
- Centralized identity management with Keycloak
- Token validation and claims-based authorization

## 🔒 Security Best Practices

**✅ OIDC Compliance:**
- Always use **Authorization Code + PKCE** for public clients
- Implement proper **nonce validation** for replay protection
- Use **state parameter** for CSRF protection
- Validate **issuer, audience, and expiration** in ID tokens

**✅ Token Management:**
- Store tokens securely (HttpOnly cookies for web apps)
- Implement proper **token refresh** logic
- Handle **token expiration** gracefully
- Use **short-lived access tokens** with refresh tokens

**✅ Keycloak Integration:**
- Configure **proper redirect URIs** in Keycloak client settings
- Use **confidential clients** for server-side applications
- Enable **PKCE** in Keycloak client configuration
- Implement **proper logout** with Keycloak session termination

## 🚀 Getting Started

### Basic Setup
```rust
use openidconnect::{
    core::{CoreClient, CoreProviderMetadata, CoreResponseType},
    reqwest::async_http_client,
    AuthenticationFlow, AuthorizationCode, ClientId, ClientSecret,
    CsrfToken, IssuerUrl, Nonce, PkceCodeChallenge, RedirectUrl, Scope,
};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Discover OIDC configuration
    let provider_metadata = CoreProviderMetadata::discover_async(
        IssuerUrl::new("https://keycloak.example.com/realms/master".to_string())?,
        async_http_client,
    ).await?;

    // Create client
    let client = CoreClient::from_provider_metadata(
        provider_metadata,
        ClientId::new("my-app".to_string()),
        Some(ClientSecret::new("client-secret".to_string())),
    )
    .set_redirect_uri(RedirectUrl::new("http://localhost:3000/callback".to_string())?);

    // Generate PKCE challenge
    let (pkce_challenge, pkce_verifier) = PkceCodeChallenge::new_random_sha256();

    // Generate authorization URL
    let (auth_url, csrf_token, nonce) = client
        .authorize_url(
            AuthenticationFlow::<CoreResponseType>::AuthorizationCode,
            CsrfToken::new_random,
            Nonce::new_random,
        )
        .add_scope(Scope::new("openid".to_string()))
        .add_scope(Scope::new("email".to_string()))
        .add_scope(Scope::new("profile".to_string()))
        .set_pkce_challenge(pkce_challenge)
        .url();

    println!("Visit this URL: {}", auth_url);
    
    Ok(())
}
```

## 💡 Advanced Features

**✅ Custom Claims Processing:**
```rust
// Define custom claims structure
#[derive(Debug, Deserialize)]
struct CustomClaims {
    roles: Option<Vec<String>>,
    department: Option<String>,
    employee_id: Option<String>,
}

// Extract custom claims from ID token
let custom_claims: CustomClaims = id_token_claims
    .additional_claims()
    .clone()
    .try_into()?;
```

**✅ Userinfo Endpoint Integration:**
```rust
// Fetch additional user information
let userinfo_claims = client
    .user_info(access_token, Some(&id_token_claims.subject()))?
    .request_async(async_http_client)
    .await?;

let full_name = userinfo_claims.name()
    .and_then(|name| name.get(None))
    .map(|name| name.as_str());
```

**✅ Token Refresh:**
```rust
// Refresh expired access token
if let Some(refresh_token) = token_response.refresh_token() {
    let refreshed_token = client
        .exchange_refresh_token(refresh_token)
        .request_async(async_http_client)
        .await?;
}
```

## 🎯 Conclusion

The `openidconnect` crate provides a **production-ready** OIDC implementation that significantly simplifies user authentication with Keycloak. Key advantages:

**✅ Strengths:**
- **Complete OIDC compliance** with automatic discovery and validation
- **Built-in security features** (PKCE, nonce, state validation)
- **Type-safe implementation** with excellent async support
- **Automatic JWT validation** eliminates manual security implementation
- **Comprehensive documentation** and active maintenance

**⚠️ Considerations:**
- **More opinionated** than the oauth2 crate (less flexibility)
- **OIDC-specific** - not suitable for pure OAuth 2.0 use cases
- **Larger dependency footprint** due to comprehensive feature set
- **Learning curve** for developers new to OIDC concepts

## ⚠️ Known Issues & Limitations

Based on the active GitHub issues, here are the main categories of known limitations:

### 🔐 **Security & Token Validation (7 issues)**
- **Device Code Flow Nonce:** Nonce not included in ID_TOKEN from server ([#219](https://github.com/ramosbugs/openidconnect-rs/issues/219))
- **ID Token Verification:** Questions around proper verification methods ([#218](https://github.com/ramosbugs/openidconnect-rs/issues/218))
- **Access Token Hash:** Automatic verification not implemented ([#192](https://github.com/ramosbugs/openidconnect-rs/issues/192))
- **Nonce Security:** Security considerations for nonce usage ([#203](https://github.com/ramosbugs/openidconnect-rs/issues/203))
- **Token Response:** Nonce not always included in responses ([#186](https://github.com/ramosbugs/openidconnect-rs/issues/186))
- **Timing Attack Vulnerability:** RSA crate security concern ([#181](https://github.com/ramosbugs/openidconnect-rs/issues/181))
- **Microsoft OIDC:** Strict validation issues with Microsoft's implementation ([#162](https://github.com/ramosbugs/openidconnect-rs/issues/162))

### 🔧 **Provider Compatibility (8 issues)**
- **GitHub OIDC:** Discovery metadata lacks authorization endpoint ([#201](https://github.com/ramosbugs/openidconnect-rs/issues/201))
- **OAuth2 Server:** Non-conforming discovery protocol support ([#197](https://github.com/ramosbugs/openidconnect-rs/issues/197))
- **Google Integration:** Invalid grant type issues ([#176](https://github.com/ramosbugs/openidconnect-rs/issues/176))
- **Auth0 Compliance:** Spec compliance issues ([#150](https://github.com/ramosbugs/openidconnect-rs/issues/150))
- **WebFinger Support:** Missing WebFinger discovery ([#142](https://github.com/ramosbugs/openidconnect-rs/issues/142))
- **Non-OIDC Providers:** Examples needed for non-standard providers ([#133](https://github.com/ramosbugs/openidconnect-rs/issues/133))
- **Introspection Endpoint:** Not discovered automatically ([#131](https://github.com/ramosbugs/openidconnect-rs/issues/131))
- **JWKS Decoupling:** Separate fetching of OpenID configuration and JWKS ([#124](https://github.com/ramosbugs/openidconnect-rs/issues/124))

### 📚 **API & Usability (6 issues)**
- **HTTP/3 Support:** Missing HTTP/3 support ([#217](https://github.com/ramosbugs/openidconnect-rs/issues/217))
- **CoreResponseMode:** Usage guidance needed ([#206](https://github.com/ramosbugs/openidconnect-rs/issues/206))
- **Claims Access:** Raw claims access without struct ([#169](https://github.com/ramosbugs/openidconnect-rs/issues/169))
- **Custom Claims:** ID token custom claims handling ([#162](https://github.com/ramosbugs/openidconnect-rs/issues/162))
- **User Info Verifier:** Method to add to Client ([#171](https://github.com/ramosbugs/openidconnect-rs/issues/171))
- **String Types:** Use of `IntoString` vs `ToString` ([#164](https://github.com/ramosbugs/openidconnect-rs/issues/164))

### 🛠️ **Implementation & Features (4 issues)**
- **Logout Tokens:** Support for parsing logout tokens ([#189](https://github.com/ramosbugs/openidconnect-rs/issues/189))
- **JWT Implementation:** More comprehensive JWT support ([#161](https://github.com/ramosbugs/openidconnect-rs/issues/161))
- **Signing Keys Interoperability:** Improved signing keys handling ([#172](https://github.com/ramosbugs/openidconnect-rs/issues/172))
- **Axum/Tower Middleware:** Code grant flow middleware ([#118](https://github.com/ramosbugs/openidconnect-rs/issues/118))

### 📖 **Documentation & Examples (3 issues)**
- **Usage Documentation:** More comprehensive usage examples needed ([#151](https://github.com/ramosbugs/openidconnect-rs/issues/151))
- **Okta Use Cases:** Better understanding of Okta integration ([#125](https://github.com/ramosbugs/openidconnect-rs/issues/125))
- **LocalizedClaim Examples:** Usage examples needed ([#118](https://github.com/ramosbugs/openidconnect-rs/issues/118))


## 📚 References & Further Reading

**Official Documentation:**
- [openidconnect crate documentation](https://docs.rs/openidconnect/latest/openidconnect/)
- [OpenID Connect Core 1.0](https://auth0.com/docs/authenticate/protocols/openid-connect-protocol)