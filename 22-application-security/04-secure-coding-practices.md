# Secure Coding Practices

Security is not a feature bolted on at the end. It is a property of how you write every line of code. These practices prevent the most common classes of web application vulnerabilities: injection, cross-site scripting, cross-site request forgery, and misconfiguration.

## Input Validation

Validate all input at the boundary. Use strong types to make invalid states unrepresentable. Reject unexpected input before it reaches business logic.

```rust
use validator::Validate;
use serde::Deserialize;
use axum::{http::StatusCode, Json};

#[derive(Debug, Deserialize, Validate)]
struct CreateUserRequest {
    #[validate(email)]
    email: String,

    #[validate(length(min = 8, max = 128))]
    password: String,

    #[validate(length(min = 1, max = 100))]
    name: String,
}

async fn create_user(
    Json(payload): Json<CreateUserRequest>,
) -> Result<Json<User>, (StatusCode, String)> {
    payload.validate().map_err(|e| {
        (StatusCode::BAD_REQUEST, format!("Validation error: {}", e))
    })?;
    // proceed with validated data
    todo!()
}
```

Validation rules:
- **Length limits** on all string inputs to prevent denial of service and buffer abuse
- **Format validation** (email, URL, phone) using well-tested libraries, not hand-written regex
- **Range checks** on numeric inputs
- **Allowlists over denylists** -- define what is permitted rather than trying to block what is dangerous

Rust's type system adds a second layer: use newtypes to enforce invariants at compile time.

```rust
struct EmailAddress(String);

impl EmailAddress {
    fn parse(s: &str) -> Result<Self, ValidationError> {
        // Validate email format
        if !s.contains('@') || s.len() > 254 {
            return Err(ValidationError::InvalidEmail);
        }
        Ok(Self(s.to_lowercase()))
    }
}
```

## Parameterized Queries

Never concatenate user input into SQL strings. This is the single most important rule for preventing injection. Parameterized queries separate code from data.

```rust
use sqlx::PgPool;

// BAD: string interpolation creates SQL injection
async fn search_bad(pool: &PgPool, term: &str) -> Result<Vec<Product>, sqlx::Error> {
    let query = format!("SELECT * FROM products WHERE name LIKE '%{}%'", term);
    sqlx::query_as::<_, Product>(&query).fetch_all(pool).await
}

// GOOD: parameterized query
async fn search(pool: &PgPool, term: &str) -> Result<Vec<Product>, sqlx::Error> {
    sqlx::query_as::<_, Product>("SELECT * FROM products WHERE name LIKE $1")
        .bind(format!("%{}%", term))
        .fetch_all(pool)
        .await
}
```

With `sqlx`, you can go further and use compile-time checked queries that verify your SQL against the actual database schema:

```rust
let products = sqlx::query_as!(
    Product,
    "SELECT id, name, price FROM products WHERE name LIKE $1",
    format!("%{}%", term)
)
.fetch_all(pool)
.await?;
```

This catches SQL syntax errors and type mismatches at compile time, not at runtime.

## Cross-Site Scripting (XSS) Prevention

An attacker injects malicious JavaScript into a page viewed by other users. Three types:
- **Stored XSS** -- Malicious script saved in the database, served to every user who views the page
- **Reflected XSS** -- Malicious script in a URL parameter, reflected back in the response
- **DOM-based XSS** -- Malicious script manipulates the DOM client-side

```rust
use axum::response::Html;

// BAD: raw user input injected into HTML
async fn profile_bad(name: &str) -> Html<String> {
    Html(format!("<h1>Hello, {}</h1>", name))
    // If name is: <script>alert('xss')</script>, it executes
}

// GOOD: use a template engine that escapes by default (Askama)
#[derive(askama::Template)]
#[template(path = "profile.html")]
struct ProfileTemplate<'a> {
    name: &'a str,  // Askama auto-escapes this in the template
}
```

Prevention checklist:
- Escape all output rendered in HTML. Template engines (Tera, Askama) escape by default.
- Set `Content-Security-Policy` headers to restrict script sources.
- Use `HttpOnly` cookies so JavaScript cannot steal session tokens.
- Sanitize HTML if you must accept rich text input (use a library like `ammonia` for Rust).

## Cross-Site Request Forgery (CSRF) Prevention

An attacker tricks a user's browser into making a request to your site while the user is authenticated. The browser automatically includes cookies, so the request looks legitimate.

Prevention strategies (use at least one, preferably combine):

1. **SameSite=Strict cookies** -- The browser does not send the cookie with cross-origin requests. This is the simplest defense and should be the default.

2. **CSRF tokens** -- Generate a random token per session, embed it in forms, and verify it on the server.

```rust
use axum::{extract::Form, http::StatusCode, Extension};

#[derive(Deserialize)]
struct TransferForm {
    to_account: String,
    amount: f64,
    csrf_token: String,
}

async fn transfer(
    Form(form): Form<TransferForm>,
    Extension(session): Extension<Session>,
) -> Result<StatusCode, StatusCode> {
    // Verify CSRF token matches the one stored in the session
    if form.csrf_token != session.csrf_token {
        return Err(StatusCode::FORBIDDEN);
    }
    // Process transfer
    todo!()
}
```

3. **Custom request headers** -- Require a custom header (e.g., `X-Requested-With`) that browsers do not include in cross-origin form submissions or simple requests.

## Security Headers

Apply security headers as middleware to every response. These headers instruct the browser to enable security features.

```rust
use axum::{middleware, http::{Request, HeaderValue}, response::Response};

async fn security_headers<B>(request: Request<B>, next: middleware::Next<B>) -> Response {
    let mut response = next.run(request).await;
    let headers = response.headers_mut();

    // Prevent MIME type sniffing
    headers.insert("X-Content-Type-Options", HeaderValue::from_static("nosniff"));

    // Prevent clickjacking by disallowing framing
    headers.insert("X-Frame-Options", HeaderValue::from_static("DENY"));

    // Disable the XSS auditor (it can introduce vulnerabilities)
    headers.insert("X-XSS-Protection", HeaderValue::from_static("0"));

    // Enforce HTTPS for 1 year, including subdomains
    headers.insert(
        "Strict-Transport-Security",
        HeaderValue::from_static("max-age=31536000; includeSubDomains"),
    );

    // Restrict content sources
    headers.insert(
        "Content-Security-Policy",
        HeaderValue::from_static("default-src 'self'; script-src 'self'; style-src 'self'"),
    );

    // Control referrer information sent to other origins
    headers.insert(
        "Referrer-Policy",
        HeaderValue::from_static("strict-origin-when-cross-origin"),
    );

    response
}
```

Header explanations:
- **`X-Content-Type-Options: nosniff`** -- Prevents the browser from guessing the content type, which can lead to script execution
- **`X-Frame-Options: DENY`** -- Prevents your pages from being embedded in iframes, blocking clickjacking
- **`Strict-Transport-Security`** -- Tells the browser to only connect via HTTPS, even if the user types `http://`
- **`Content-Security-Policy`** -- The most powerful security header. Restricts which sources can load scripts, styles, images, and other resources. Start strict (`'self'` only) and relax as needed.
- **`Referrer-Policy`** -- Controls how much URL information is shared when navigating to other sites

## Secure Defaults

Design systems so that the secure path is the easy path. Insecurity should require explicit opt-in.

Principles:
- **Deny by default.** New endpoints should require authentication and authorization until explicitly marked as public.
- **Fail closed.** If an authorization check fails or throws an error, deny access. Never default to allowing.
- **Minimize surface area.** Disable unused features, endpoints, and debug tooling in production.
- **Environment-aware configuration.** Separate development and production configurations. Debug mode, verbose error messages, and permissive CORS belong in development only.

```rust
#[derive(Clone)]
struct SecurityConfig {
    cors_origins: Vec<String>,
    debug_errors: bool,
    require_https: bool,
}

impl SecurityConfig {
    fn production() -> Self {
        Self {
            cors_origins: vec!["https://app.example.com".to_string()],
            debug_errors: false,
            require_https: true,
        }
    }

    fn development() -> Self {
        Self {
            cors_origins: vec!["http://localhost:3000".to_string()],
            debug_errors: true,
            require_https: false,
        }
    }
}
```

## Error Handling

Never expose internal details in error responses. Stack traces, database error messages, and internal paths give attackers information about your system.

```rust
// BAD: exposes internal details
async fn get_user_bad(pool: &PgPool, id: u64) -> Result<Json<User>, String> {
    db::get_user(pool, id)
        .await
        .map(Json)
        .map_err(|e| format!("Database error: {}", e))  // Leaks DB info
}

// GOOD: generic error to client, detailed error to logs
async fn get_user(pool: &PgPool, id: u64) -> Result<Json<User>, StatusCode> {
    db::get_user(pool, id)
        .await
        .map(Json)
        .map_err(|e| {
            tracing::error!("Failed to fetch user {}: {}", id, e);
            StatusCode::INTERNAL_SERVER_ERROR
        })
}
```

## Logging Sensitive Data

Never log passwords, credit card numbers, tokens, or personal data. Logs are often stored with weaker access controls than production databases and may be shipped to third-party aggregation services.

```rust
// BAD: logs the password
tracing::info!("Login attempt: email={}, password={}", email, password);

// GOOD: log the event without sensitive data
tracing::info!("Login attempt: email={}", email);

// GOOD: redact sensitive fields in structs
#[derive(Debug)]
struct AuditableRequest {
    email: String,
    password: String, // should never appear in logs
}

impl std::fmt::Display for AuditableRequest {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "AuditableRequest {{ email: {}, password: [REDACTED] }}", self.email)
    }
}
```

## Common Mistakes

- **Overly broad CORS policies.** `Access-Control-Allow-Origin: *` on an API with cookies or sensitive data allows any website to make authenticated requests. Lock CORS to known frontend origins.
- **Exposing stack traces in production.** Set `debug_errors: false` and return generic error messages to clients.
- **Logging sensitive data.** Sanitize all log output. Never log passwords, tokens, or PII.
- **Not validating content types.** Accept only expected content types (`application/json`) and reject everything else.
- **Using `unwrap()` in request handlers.** A panic in a handler crashes the task and may leak information in the error response. Use proper error handling with `Result`.

## Key Takeaways

1. Validate all input at the boundary. Use the `validator` crate and Rust's type system to reject bad data early.
2. Use parameterized queries for all database access. Never concatenate user input into SQL.
3. Apply security headers as middleware. Start with a strict CSP and relax as needed.
4. Make security the default. Insecurity should require explicit opt-in, not the other way around.
5. Never expose internal error details to clients. Log them for debugging; return generic messages to users.
