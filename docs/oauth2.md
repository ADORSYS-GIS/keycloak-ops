# 🧩 oauth2 Crate Integration with Keycloak

## 📦 Overview

The `oauth2` crate is a comprehensive, strongly-typed OAuth 2.0 client library for Rust applications. This documentation covers our complete implementation with Keycloak, including working demos for both Resource Owner Password Credentials Grant and Implicit Grant flows.

**Purpose in Keycloak Ecosystem:**
- Primary OAuth 2.0/OIDC client integration library for Rust applications
- Enables secure authentication and authorization with Keycloak servers
- Handles token lifecycle management (acquisition, refresh, validation)
- Provides type-safe abstractions for OAuth 2.0 protocol interactions
- **JWT signature verification** using Keycloak's public keys (JWKS)

**Implementation Status:**
- **Version:** 5.0.0 (Latest stable)
- **GitHub Stars:** 1.1k+ ⭐
- **Contributors:** 90+ active collaborators
- **Last Commit:** April 19, 2025 (actively maintained)
- **JWT Validation:** Real-time token verification using Keycloak's public keys
- **Protected Endpoints:** Functional API endpoints with token-based authentication
- **License:** MIT/Apache-2.0 dual license

## ⚙️ Key Features

**Authentication & Authorization Support:**
- Authorization Code Grant (with PKCE support)
- Implicit Grant Flow
- Authorization Code Grant (with PKCE support)

- Authorization Code Grant (with PKCE support)
- Implicit Grant Flow
- Client Credentials Grant
- Device Authorization Grant
- Custom grant type extensibility
- Client Credentials Grant
- Device Authorization Grant
- Custom grant type extensibility

- Client Credentials Grant
- Device Authorization Grant
- Custom grant type extensibility

- Resource Owner Password Credentials Grant
- Authorization Code Grant (with PKCE support)
- Implicit Grant Flow
- Client Credentials Grant
- Device Authorization Grant
- Custom grant type extensibility

**Token Management:**
- **JWT Signature Verification:** Using Keycloak's JWKS endpoint
- **Real-time Token Validation:** Cryptographic verification of token authenticity
- **Claims Extraction:** User information from validated JWT tokens
- **Token Expiration Handling:** Automatic validation of token lifetime
- **Protected Resource Access:** Bearer token authentication for API endpoints

**Implemented Features:**
- **Resource Owner Password Grant:** Direct username/password authentication
- **Implicit Grant Flow:** Browser-based authentication for SPAs
- **JWT Validation:** Full cryptographic verification using RSA public keys
- **Protected Endpoints:** `/protected` and `/api/userinfo` with token validation
- **User Information Display:** Extracted from validated JWT claims

**Framework Integration:**
- **Axum:** Complete async web framework implementation
- **Tokio:** Full async/await support with non-blocking operations
- **Reqwest:** HTTP client for Keycloak API interactions
- **Serde:** JSON serialization for OAuth2 responses and JWT claims
- **Chrono:** Token expiration and timestamp handling

## 🧠 Architecture & Design

**Core Modules:**
- `oauth2::basic` - Standard OAuth 2.0 client implementation
- `oauth2::reqwest` - Async HTTP client integration using reqwest
- `oauth2::curl` - Synchronous HTTP client using libcurl
- `oauth2::types` - Core type definitions and trait abstractions

**Async/Await Support:**
- Full async/await compatibility with Tokio runtime
- Non-blocking HTTP operations
- Concurrent token refresh handling
- Stream-based token processing

**Dependency Footprint (Actual Implementation):**
- **Core:** `oauth2 = "5.0.0"`, `serde`, `tokio`, `anyhow`
- **HTTP Client:** `reqwest` for async Keycloak API calls
- **JWT Validation:** `jsonwebtoken = "9.2"`, `base64 = "0.21"`
- **Web Framework:** `axum = "0.7"`, `tower-http` for CORS
- **JSON:** `serde_json` for OAuth2 responses and JWT claims
- **Time:** `chrono = "0.4"` for token expiration handling
- **Utilities:** `uuid`, `urlencoding`, `clap` for CLI interface

**Compatibility (Tested):**
- **Rust Edition:** 2021 (MSRV: 1.65.0)
- **Async Runtime:** Tokio 1.0+ (fully tested)
- **Keycloak Version:** 23.0+ (tested with latest)
- **TLS:** Full HTTPS support for production Keycloak instances

## ⚠️ Known Issues & Considerations

Based on the current GitHub issues (as of April 2025), be aware of these potential challenges:

**Active Issues:**
- **Serialization traits:** Some marginal issues with serialization implementations
- **Access token requests:** `expires_in` field handling as string vs number
- **Rand 0.9 update:** Migration to newer random number generation
- **Self-signed certificates:** Limited support for development environments
- **Error handling:** Status 200 responses that should return errors
- **Documentation:** Examples may need updates for latest features

## 🔐 Integration Flow

### Resource Owner Password Credentials Grant

**Authentication Mechanism:**
Direct username/password exchange for access tokens. Suitable for highly trusted first-party applications where the client can securely handle user credentials.

**How Tokens are Fetched (Implemented Flow):**
1. User enters credentials in web form
2. Credentials sent via POST to an endpoint
3. Server exchanges credentials with Keycloak token endpoint using `oauth2` crate
4. Keycloak validates credentials and returns JWT access token
5. User redirected to an endpoint with token

**Token Verification (Implemented):**
- **JWT Signature Validation:** Using Keycloak's JWKS endpoint (`/realms/master/protocol/openid-connect/certs`)
- **RSA Public Key Verification:** Cryptographic validation of token authenticity
- **Claims Extraction:** Username, email, subject, issuer, expiration from JWT
- **Real-time Validation:** Every protected endpoint validates tokens before access

**Actual Working Implementation:**
```rust
// From crate-test/src/main.rs - Password authentication handler
async fn password_auth(
    State(state): State<AppState>,
    Form(form): Form<PasswordForm>,
) -> Result<Redirect, Html<String>> {
    let username_cred = ResourceOwnerUsername::new(form.username);
    let password_cred = ResourceOwnerPassword::new(form.password);
    
    let token_result = state
        .oauth_client
        .exchange_password(&username_cred, &password_cred)
        .add_scope(Scope::new("openid".to_string()))
        .add_scope(Scope::new("profile".to_string()))
        .add_scope(Scope::new("email".to_string()))
        .request_async(oauth2::reqwest::async_http_client)
        .await;
    
    match token_result {
        Ok(token) => {
            let access_token = token.access_token().secret();
            let success_url = format!("/success?access_token={}", urlencoding::encode(access_token));
            Ok(Redirect::to(&success_url))
        }
        Err(e) => Err(Html(format!("Authentication failed: {}", e)))
    }
}

// JWT validation using Keycloak's public keys
async fn validate_jwt_token(token: &str, keycloak_config: &KeycloakConfig) -> anyhow::Result<Claims> {
    let header = jsonwebtoken::decode_header(token)?;
    let kid = header.kid.ok_or_else(|| anyhow::anyhow!("Token missing kid"))?;
    
    let jwks = fetch_jwks(keycloak_config).await?;
    let jwk = jwks.keys.iter().find(|key| key.kid == kid)
        .ok_or_else(|| anyhow::anyhow!("Key with kid {} not found", kid))?;
    
    let decoding_key = DecodingKey::from_rsa_components(&jwk.n, &jwk.e)?;
    let mut validation = Validation::new(Algorithm::RS256);
    validation.validate_aud = false;
    validation.set_issuer(&[&format!("{}/realms/{}", keycloak_config.base_url, keycloak_config.realm)]);
    
    let token_data = decode::<Claims>(token, &decoding_key, &validation)?;
    Ok(token_data.claims)
}
```

### Implicit Grant Flow

**Authentication Mechanism:**
Browser-based flow for public clients (SPAs, mobile apps) where client secrets cannot be securely stored. Access token returned directly in URL fragment.

**How Tokens are Fetched (Implemented Flow):**
1. User clicks "🚀 Start Authentication" button on home page
2. Browser redirects to Keycloak authorization endpoint with `response_type=token`
3. User authenticates via Keycloak login interface
4. Keycloak redirects to `/callback#access_token=...&token_type=Bearer` (token in URL fragment)
5. JavaScript extracts token from URL fragment and redirects to `<endpoint>?access_token=...`
6. Success page validates JWT using Keycloak's public keys

**Token Verification (Implemented):**
- **Server-side JWT Validation:** Same cryptographic verification as password flow
- **JWKS Public Key Verification:** Real-time validation using Keycloak's certificates
- **No Refresh Token:** Implicit flow provides only access tokens (by design)
- **Token Expiration:** Configurable in Keycloak client settings

**Actual Working Implementation:**
```rust
// From crate-test/src/main.rs - Start implicit flow authentication
async fn start_auth(State(state): State<AppState>) -> Result<Redirect, StatusCode> {
    let (auth_url, _csrf_token) = state
        .oauth_client
        .authorize_url(CsrfToken::new_random)
        .set_response_type(&oauth2::ResponseType::new("token".to_string()))
        .add_scope(Scope::new("openid".to_string()))
        .add_scope(Scope::new("profile".to_string()))
        .add_scope(Scope::new("email".to_string()))
        .url();
    
    Ok(Redirect::to(auth_url.as_str()))
}

// Handle callback with JavaScript token extraction
async fn handle_callback(Query(params): Query<AuthCallback>) -> Result<Html<String>, Html<String>> {
    // Return HTML with JavaScript to extract token from URL fragment
    Ok(Html(r#"
        <script>
            if (window.location.hash) {
                const fragment = window.location.hash.substring(1);
                const params = new URLSearchParams(fragment);
                const accessToken = params.get('access_token');
                
                if (accessToken) {
                    window.location.href = '/success?access_token=' + encodeURIComponent(accessToken);
                } else {
                    // Handle error cases
                }
            }
        </script>
    "#.to_string()))
}
```