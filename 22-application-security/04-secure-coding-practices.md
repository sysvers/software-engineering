# Secure Coding Practices

Security is not a feature bolted on at the end. It is a property of how you write every line of code. These practices prevent the most common classes of web application vulnerabilities: injection, cross-site scripting, cross-site request forgery, and misconfiguration.

## Input Validation

Validate all input at the boundary. Use strong types to make invalid states unrepresentable. Reject unexpected input before it reaches business logic.

```text
STRUCTURE CreateUserRequest
    email : String          // must be valid email format
    password : String       // length between 8 and 128
    name : String           // length between 1 and 100

PROCEDURE CREATE_USER(payload) → Result<User>
    IF NOT VALIDATE(payload) THEN
        RETURN Error(400 BAD REQUEST, "Validation error: " + errors)
    END IF
    // proceed with validated data
    ...
```

Validation rules:
- **Length limits** on all string inputs to prevent denial of service and buffer abuse
- **Format validation** (email, URL, phone) using well-tested libraries, not hand-written regex
- **Range checks** on numeric inputs
- **Allowlists over denylists** -- define what is permitted rather than trying to block what is dangerous

Rust's type system adds a second layer: use newtypes to enforce invariants at compile time.

```text
STRUCTURE EmailAddress
    value : String

PROCEDURE EmailAddress.PARSE(s) → Result<EmailAddress>
    // Validate email format
    IF s DOES NOT CONTAIN '@' OR LENGTH(s) > 254 THEN
        RETURN Error(InvalidEmail)
    END IF
    RETURN Ok(NEW EmailAddress(LOWERCASE(s)))
```

## Parameterized Queries

Never concatenate user input into SQL strings. This is the single most important rule for preventing injection. Parameterized queries separate code from data.

```text
// BAD: string interpolation creates SQL injection
PROCEDURE SEARCH_BAD(pool, term) → Result<List<Product>>
    query ← "SELECT * FROM products WHERE name LIKE '%" + term + "%'"
    RETURN EXECUTE_QUERY(pool, query)

// GOOD: parameterized query
PROCEDURE SEARCH(pool, term) → Result<List<Product>>
    RETURN EXECUTE_QUERY(pool,
        "SELECT * FROM products WHERE name LIKE $1",
        BIND "%" + term + "%")
```

With `sqlx`, you can go further and use compile-time checked queries that verify your SQL against the actual database schema:

```text
// Compile-time checked query -- catches SQL errors and type mismatches at build time
products ← EXECUTE_CHECKED_QUERY(pool,
    "SELECT id, name, price FROM products WHERE name LIKE $1",
    BIND "%" + term + "%")
```

This catches SQL syntax errors and type mismatches at compile time, not at runtime.

## Cross-Site Scripting (XSS) Prevention

An attacker injects malicious JavaScript into a page viewed by other users. Three types:
- **Stored XSS** -- Malicious script saved in the database, served to every user who views the page
- **Reflected XSS** -- Malicious script in a URL parameter, reflected back in the response
- **DOM-based XSS** -- Malicious script manipulates the DOM client-side

```text
// BAD: raw user input injected into HTML
PROCEDURE PROFILE_BAD(name) → HTML
    RETURN HTML("<h1>Hello, " + name + "</h1>")
    // If name is: <script>alert('xss')</script>, it executes

// GOOD: use a template engine that escapes by default
STRUCTURE ProfileTemplate
    name : String   // Template engine auto-escapes this in the template
// Render using template file "profile.html" with auto-escaping
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

```text
STRUCTURE TransferForm
    to_account : String
    amount : Float
    csrf_token : String

PROCEDURE TRANSFER(form, session) → Result<StatusCode>
    // Verify CSRF token matches the one stored in the session
    IF form.csrf_token ≠ session.csrf_token THEN
        RETURN Error(403 FORBIDDEN)
    END IF
    // Process transfer
    ...
```

3. **Custom request headers** -- Require a custom header (e.g., `X-Requested-With`) that browsers do not include in cross-origin form submissions or simple requests.

## Security Headers

Apply security headers as middleware to every response. These headers instruct the browser to enable security features.

```text
PROCEDURE SECURITY_HEADERS(request, next) → Response
    response ← next.RUN(request)
    headers ← response.HEADERS()

    // Prevent MIME type sniffing
    SET headers["X-Content-Type-Options"] ← "nosniff"

    // Prevent clickjacking by disallowing framing
    SET headers["X-Frame-Options"] ← "DENY"

    // Disable the XSS auditor (it can introduce vulnerabilities)
    SET headers["X-XSS-Protection"] ← "0"

    // Enforce HTTPS for 1 year, including subdomains
    SET headers["Strict-Transport-Security"] ← "max-age=31536000; includeSubDomains"

    // Restrict content sources
    SET headers["Content-Security-Policy"] ← "default-src 'self'; script-src 'self'; style-src 'self'"

    // Control referrer information sent to other origins
    SET headers["Referrer-Policy"] ← "strict-origin-when-cross-origin"

    RETURN response
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

```text
STRUCTURE SecurityConfig
    cors_origins : List<String>
    debug_errors : Boolean
    require_https : Boolean

PROCEDURE SecurityConfig.PRODUCTION() → SecurityConfig
    RETURN SecurityConfig {
        cors_origins ← ["https://app.example.com"],
        debug_errors ← FALSE,
        require_https ← TRUE
    }

PROCEDURE SecurityConfig.DEVELOPMENT() → SecurityConfig
    RETURN SecurityConfig {
        cors_origins ← ["http://localhost:3000"],
        debug_errors ← TRUE,
        require_https ← FALSE
    }
```

## Error Handling

Never expose internal details in error responses. Stack traces, database error messages, and internal paths give attackers information about your system.

```text
// BAD: exposes internal details
PROCEDURE GET_USER_BAD(pool, id) → Result<User>
    result ← DB_GET_USER(pool, id)
    IF result IS ERROR THEN
        RETURN Error("Database error: " + error_details)  // Leaks DB info
    END IF
    RETURN Ok(result)

// GOOD: generic error to client, detailed error to logs
PROCEDURE GET_USER(pool, id) → Result<User>
    result ← DB_GET_USER(pool, id)
    IF result IS ERROR THEN
        LOG ERROR "Failed to fetch user " + id + ": " + error_details
        RETURN Error(500 INTERNAL SERVER ERROR)
    END IF
    RETURN Ok(result)
```

## Logging Sensitive Data

Never log passwords, credit card numbers, tokens, or personal data. Logs are often stored with weaker access controls than production databases and may be shipped to third-party aggregation services.

```text
// BAD: logs the password
LOG INFO "Login attempt: email=" + email + ", password=" + password

// GOOD: log the event without sensitive data
LOG INFO "Login attempt: email=" + email

// GOOD: redact sensitive fields in structs
STRUCTURE AuditableRequest
    email : String
    password : String   // should never appear in logs

PROCEDURE AuditableRequest.TO_STRING() → String
    RETURN "AuditableRequest { email: " + self.email + ", password: [REDACTED] }"
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
