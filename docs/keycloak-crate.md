# 🧩 Crate Name

## keycloak

# 📦 Overview

The `keycloak` crate provides a comprehensive Rust client for the [Keycloak Admin REST API](https://www.keycloak.org/docs-api/26.0/rest-api/index.html).  
Its main purpose is to automate Keycloak administration tasks such as managing realms, users, roles, and groups from Rust applications.

- **Ecosystem Role:** Admin client for Keycloak; does not provide OpenID Connect integration or token validation for end-user authentication flows.
- **Maintenance:** Actively maintained (latest release: v26.4.0).
  - [Crates.io](https://crates.io/crates/keycloak): 100+ stars, regular updates, several contributors.

---

# ⚙️ Key Features

- **Admin REST API Coverage:** Full support for Keycloak Admin REST API v26.4.0.
- **Async API:** Built on `tokio` and `reqwest` for asynchronous operations.
- **Resource Management:** Create, update, delete realms, users, roles, groups, and more.
- **Feature Flags:**
  - `tags-all` (default): Enables all resource groups.
  - `rc`: Use `Arc` for deserialization.
  - `schemars`: Adds `schemars` support.
  - `multipart`: Enables multipart support for extra API methods.
  - `resource-builder`: Adds resource builder support.
- **Role & Group Management:** Manage user roles and groups programmatically.
- **Integration Examples:** Works with any async Rust framework (e.g., Axum, Actix, Rocket) via standard HTTP client.

---

# 🧠 Architecture & Design

- **Core Modules:**
  - `KeycloakAdmin`: Main entry point for admin operations.
  - `KeycloakAdminToken`: Handles authentication and token acquisition.
  - `types`: Data structures for realms, users, roles, etc.
- **Async/Await:** All operations are async, leveraging `tokio` and `reqwest`.
- **Dependencies:**
  - `reqwest` (HTTP client)
  - `serde` (serialization/deserialization)
  - Optional: `schemars`, `Arc`, multipart
- **Compatibility:**
  - Rust edition: 2021 or newer (requires Rust >= 1.87.0)
  - Async runtime: `tokio`

---

# 🔐 Integration Flow

## Authentication Mechanism

- Uses admin credentials (username/password) to acquire an access token via Keycloak's token endpoint.
- Token is used for subsequent API calls.

## Token Handling

- Token acquisition is handled by `KeycloakAdminToken::acquire`.
- Token is automatically attached to requests.

## Example Usage

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    use keycloak::{KeycloakAdmin, KeycloakAdminToken, types::*};

    // Set up connection details
    let url = std::env::var("KEYCLOAK_ADDR").unwrap_or_else(|_| "http://localhost:8080".into());
    let user = std::env::var("KEYCLOAK_USER").unwrap_or_else(|_| "admin".into());
    let password = std::env::var("KEYCLOAK_PASSWORD").unwrap_or_else(|_| "password".into());

    let client = reqwest::Client::new();
    let admin_token = KeycloakAdminToken::acquire(&url, &user, &password, &client).await?;

    let admin = KeycloakAdmin::new(&url, admin_token, client);

    // Create a realm
    admin.post(RealmRepresentation {
        realm: Some("resource".into()),
        ..Default::default()
    }).await?;

    let realm = admin.realm("resource");

    // Add a user
    let response = realm.users_post(UserRepresentation {
        username: Some("user".into()),
        ..Default::default()
    }).await?;

    // Fetch users
    let users = realm.users_get().username("user".to_string()).await?;

    // Delete the user
    let id = users.iter()
        .find(|u| u.username == Some("user".into()))
        .unwrap()
        .id
        .as_ref()
        .unwrap()
        .to_string();

    realm.users_with_user_id_delete(id.as_str()).await?;

    // Delete the realm
    realm.delete().await?;
    Ok(())
}
```

---

# 📚 References

- [Crate Documentation](https://crates.io/crates/keycloak)
- [GitHub Repository](https://github.com/keycloak/keycloak-rs)
- [Keycloak Admin REST API Docs](https://www.keycloak.org/docs-api/26.0/rest-api/index.html)

---

# 📝 Summary

- `keycloak` is the primary Rust crate for Keycloak Admin REST API integration.
- It is async, feature-rich, and actively maintained.
- Suitable for automating Keycloak administration tasks from Rust applications.
- Requires Rust >= 1.87.0 and a running Keycloak instance.

---

# 🚀 Alternative: rust-keycloak

## Overview

**rust-keycloak** is another Rust crate for interacting with Keycloak. It provides both OpenID Connect and admin API access, focusing on authentication flows and basic user management.


- **Latest Version:** 0.0.6
- **Ecosystem Role:** OpenID Connect integration and basic admin operations.
- **Maintenance:** Less active, fewer features than `keycloak` crate.


## Key Features

- **OpenID Connect:**
  - `well_known` endpoint discovery
  - `token` acquisition
  - `refresh_token` support
  - `jwt_decode` for token validation
  - `service_account` support
- **Admin API:**
  - Create, delete, update users
  - Count users
  - Get user info
  - Introspect tokens
  - Add/remove users to/from groups
  - Add realm/client roles to users

## Example Usage

```rust
let token_request = rust_keycloak::serde_json::json!({
    "grant_type": "password",
    "username": "admin",
    "password": "password",
    "realm": "realm_name",
    "client_id": "client_id",
    "redirect_uri": "",
    "code": ""
});

let tok = rust_keycloak::keycloak::OpenId::token("http://localhost:8080", &token_request).await?;
println!("Access token: {}", tok.access_token);
```

## References

   - [Crates.io: rust-keycloak](https://crates.io/crates/rust-keycloak)
   - [GitHub: tritone11/rust-keycloak](https://github.com/tritone11/rust-keycloak)


## Comparison: `keycloak` vs `rust-keycloak`

| Feature        | keycloak crate          | rust-keycloak crate        |
| -------------- | ----------------------- | -------------------------- |
| Admin REST API | Full coverage (v26.4.0) | Basic user/group ops       |
| OpenID Connect | No (admin only)         | Yes (OIDC flows supported) |
| Async support  | Yes (tokio/reqwest)     | Yes                        |
| Feature flags  | Extensive               | Minimal                    |
| Maintenance    | Actively maintained     | Less active                |
| Use case       | Admin automation        | Auth flows + basic admin   |

**Summary:**

- Use `keycloak` crate for full admin automation and advanced Keycloak management.
- Use `rust-keycloak` crate for OpenID Connect authentication flows and simple user management.

---
