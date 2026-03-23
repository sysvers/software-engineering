# Authentication

Authentication answers the question: "Who are you?" It is the process of verifying that a user or system is who they claim to be. Every security decision downstream depends on getting authentication right.

## Session-Based Authentication

The server creates a session after login, stores it server-side (database, Redis), and sends a session ID in a cookie. Every subsequent request includes the cookie. The server looks up the session to identify the user.

```rust
use axum::{extract::Form, http::StatusCode, Extension};
use axum_extra::extract::cookie::{Cookie, CookieJar, SameSite};
use uuid::Uuid;

async fn login(
    Form(credentials): Form<LoginRequest>,
    jar: CookieJar,
    Extension(pool): Extension<PgPool>,
    Extension(session_store): Extension<RedisPool>,
) -> Result<(CookieJar, StatusCode), StatusCode> {
    let user = db::find_user_by_email(&pool, &credentials.email)
        .await
        .map_err(|_| StatusCode::UNAUTHORIZED)?;

    // Verify password using argon2
    let parsed_hash = argon2::PasswordHash::new(&user.password_hash)
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    argon2::Argon2::default()
        .verify_password(credentials.password.as_bytes(), &parsed_hash)
        .map_err(|_| StatusCode::UNAUTHORIZED)?;

    // Create session
    let session_id = Uuid::new_v4().to_string();
    session_store::set(&session_store, &session_id, user.id, Duration::from_secs(86400))
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    let cookie = Cookie::build(("session_id", session_id))
        .http_only(true)
        .secure(true)
        .same_site(SameSite::Strict)
        .max_age(time::Duration::days(1))
        .path("/")
        .build();

    Ok((jar.add(cookie), StatusCode::OK))
}
```

Key cookie properties:
- `HttpOnly` -- JavaScript cannot read the cookie, preventing XSS-based session theft
- `Secure` -- Cookie is only sent over HTTPS
- `SameSite=Strict` -- Cookie is not sent with cross-origin requests, mitigating CSRF

**When to use sessions:** Single-domain web applications where you need immediate revocation (logout, account compromise) and server-side storage is not a scalability concern.

## JWT (JSON Web Tokens)

JWTs are self-contained tokens. The server signs them; clients send them in the `Authorization` header. No server-side session storage needed. The server verifies the signature to trust the claims.

```rust
use jsonwebtoken::{encode, decode, Header, Algorithm, Validation, EncodingKey, DecodingKey};
use serde::{Deserialize, Serialize};
use chrono::Utc;

#[derive(Debug, Serialize, Deserialize)]
struct Claims {
    sub: String,       // user id
    role: String,      // user role
    exp: usize,        // expiration (UNIX timestamp)
    iat: usize,        // issued at
}

fn create_jwt(user_id: &str, role: &str, secret: &[u8]) -> Result<String, jsonwebtoken::errors::Error> {
    let now = Utc::now().timestamp() as usize;
    let claims = Claims {
        sub: user_id.to_string(),
        role: role.to_string(),
        exp: now + 3600,  // 1 hour
        iat: now,
    };
    encode(&Header::default(), &claims, &EncodingKey::from_secret(secret))
}

fn verify_jwt(token: &str, secret: &[u8]) -> Result<Claims, jsonwebtoken::errors::Error> {
    let validation = Validation::new(Algorithm::HS256);
    let token_data = decode::<Claims>(token, &DecodingKey::from_secret(secret), &validation)?;
    Ok(token_data.claims)
}
```

**When to use JWTs:** Distributed systems or microservices that need to verify identity without calling a central session store. APIs consumed by mobile apps or SPAs. Accept that revocation requires a blocklist or short-lived tokens with refresh rotation.

## Sessions vs. JWTs

| Property | Sessions | JWTs |
|----------|----------|------|
| Storage | Server-side (Redis, DB) | Client-side (token) |
| Revocation | Immediate (delete from store) | Difficult (wait for expiry or maintain blocklist) |
| Scalability | Requires shared session store | Stateless, any server can verify |
| Size | Small cookie (~36 bytes) | Larger token (~500+ bytes) |
| Security | Server controls session data | Claims are visible (base64), only signature is secret |

## OAuth 2.0 and OpenID Connect

OAuth 2.0 is an authorization framework that lets users grant third-party applications limited access to their resources without sharing credentials. OpenID Connect (OIDC) is an identity layer on top of OAuth 2.0 that adds authentication -- it tells you *who* the user is, not just *what they can access*.

**Authorization Code flow (most common for server-side apps):**

1. User clicks "Login with Google"
2. App redirects to Google's authorization endpoint with a `code_challenge` (PKCE)
3. User authenticates with Google, consents to requested scopes
4. Google redirects back with an authorization `code`
5. App exchanges `code` + `code_verifier` for tokens at Google's token endpoint
6. App receives an `id_token` (who they are) and an `access_token` (what they can access)

**PKCE (Proof Key for Code Exchange):** Prevents authorization code interception attacks. The client generates a random `code_verifier`, sends a hashed `code_challenge` in the authorization request, and proves possession of the verifier when exchanging the code for tokens. Required for public clients (SPAs, mobile apps), recommended for all clients.

**When to use OAuth 2.0 / OIDC:** Social login ("Login with Google/GitHub"). APIs that third parties will integrate with. When you do not want to store or manage user passwords.

### Real-World Incident: GitHub OAuth Token Theft (2022)

Attackers stole OAuth tokens issued to Heroku and Travis CI that were stored as GitHub integrations. These tokens granted access to private repositories of dozens of organizations, including npm. The incident showed that OAuth token storage, scope limitation, and token rotation are not optional. Tokens should have the minimum scope necessary and be rotated regularly.

## Password Hashing with Argon2

Never store passwords in plaintext or use fast hashing algorithms (MD5, SHA-256). Use a memory-hard, slow-by-design algorithm like Argon2, the winner of the 2015 Password Hashing Competition.

```rust
use argon2::{
    password_hash::{rand_core::OsRng, PasswordHash, PasswordHasher, PasswordVerifier, SaltString},
    Argon2,
};

fn hash_password(password: &str) -> Result<String, argon2::password_hash::Error> {
    let salt = SaltString::generate(&mut OsRng);
    let argon2 = Argon2::default();
    let hash = argon2.hash_password(password.as_bytes(), &salt)?;
    Ok(hash.to_string())
}

fn verify_password(password: &str, hash: &str) -> Result<bool, argon2::password_hash::Error> {
    let parsed_hash = PasswordHash::new(hash)?;
    Ok(Argon2::default()
        .verify_password(password.as_bytes(), &parsed_hash)
        .is_ok())
}
```

Why Argon2 over bcrypt or scrypt: Argon2 is memory-hard, making GPU-based cracking significantly more expensive. It offers tunable parameters for memory, iterations, and parallelism. The default parameters in the `argon2` crate are suitable for most applications.

## Multi-Factor Authentication (MFA)

MFA requires users to present two or more independent authentication factors:
- **Something you know** -- password, PIN
- **Something you have** -- TOTP app, hardware key (YubiKey), phone
- **Something you are** -- biometrics (fingerprint, face)

**TOTP (Time-based One-Time Password):** The most common MFA implementation. The server and user share a secret key. Both compute a 6-digit code based on the current time (30-second windows). The `totp-rs` crate implements this in Rust.

```rust
use totp_rs::{Algorithm, TOTP, Secret};

fn generate_totp_secret() -> TOTP {
    let secret = Secret::generate_secret();
    TOTP::new(
        Algorithm::SHA1,
        6,      // digits
        1,      // skew (allow 1 step before/after)
        30,     // step in seconds
        secret.to_bytes().unwrap(),
    ).unwrap()
}

fn verify_totp(totp: &TOTP, code: &str) -> bool {
    totp.check_current(code).unwrap_or(false)
}
```

**WebAuthn / Passkeys:** The modern standard for phishing-resistant authentication. Uses public-key cryptography tied to a device. The private key never leaves the authenticator (hardware key or platform authenticator). Eliminates password reuse, phishing, and credential stuffing entirely. Adoption is accelerating as browsers and operating systems add native support.

## Common Mistakes

- **Not rate-limiting authentication endpoints.** Without rate limiting, attackers can attempt millions of passwords per hour. Rate limit login, password reset, and MFA verification endpoints. Use exponential backoff or account lockout after repeated failures.
- **Rolling your own authentication or cryptography.** Custom password hashing, homebrew token schemes, and self-designed encryption are almost always broken. Use battle-tested libraries.
- **Storing secrets in code or environment variables without encryption.** Use a secrets manager (HashiCorp Vault, AWS Secrets Manager). Add `.env` to `.gitignore` on day one.
- **Session tokens that never expire.** Always set a maximum session lifetime. Implement idle timeout in addition to absolute timeout.
- **Using JWTs with the `none` algorithm.** Always validate the algorithm in token verification. The `jsonwebtoken` crate rejects `none` by default.

## Key Takeaways

1. Sessions are simpler and easier to revoke; JWTs are stateless and scale better. Choose based on your architecture.
2. Always use Argon2 for password hashing. Never MD5, SHA-256, or even bcrypt for new projects.
3. MFA is not optional for any application handling sensitive data. Prefer WebAuthn/passkeys where possible.
4. OAuth 2.0 with PKCE is the standard for delegated authentication. Minimize token scopes and rotate tokens.
5. Every authentication decision has security implications. The defaults in your framework matter -- understand them before shipping.
