# OWASP Top 10

The OWASP Top 10 is the industry-standard ranking of the most critical web application security risks. Updated periodically by the Open Web Application Security Project, it reflects real-world attack data collected from hundreds of organizations. Every engineer building networked software should internalize these categories.

## A01: Broken Access Control

Users act outside their intended permissions. This includes missing authorization checks on endpoints, Insecure Direct Object References (IDOR), privilege escalation, and exposing admin functionality to regular users.

Broken access control has been the number-one risk since the 2021 revision. It is the most commonly exploited category because developers often rely on the UI to hide functionality rather than enforcing rules on the server.

```text
// BAD: any authenticated user can view any order
PROCEDURE GET_ORDER_BAD(order_id) → Result<Order>
    order ← DB_GET_ORDER(order_id)
    IF order NOT FOUND THEN RETURN Error(404 NOT FOUND)
    RETURN Ok(order)

// GOOD: verify the authenticated user owns the resource
PROCEDURE GET_ORDER(order_id, current_user) → Result<Order>
    order ← DB_GET_ORDER(order_id)
    IF order NOT FOUND THEN RETURN Error(404 NOT FOUND)
    IF order.user_id ≠ current_user.id THEN
        RETURN Error(403 FORBIDDEN)
    END IF
    RETURN Ok(order)
```

**Prevention:** Deny by default. Every endpoint must independently verify that the authenticated user is authorized for the requested action. Never rely on obscurity (unguessable IDs) as a substitute for authorization checks.

## A02: Cryptographic Failures

Storing passwords in plaintext, using broken hashing algorithms (MD5, SHA-1 for passwords), transmitting sensitive data without TLS, hardcoding encryption keys, or using weak random number generators.

**Prevention:** Use Argon2 for password hashing. Enforce TLS everywhere. Store encryption keys in a secrets manager, never in source code. Use the `ring` or `rustls` crates for cryptographic operations in Rust.

## A03: Injection

Untrusted data sent to an interpreter as part of a command or query. SQL injection remains the most common form, but command injection, LDAP injection, and template injection also fall here.

```text
// BAD: string interpolation creates SQL injection
PROCEDURE FIND_USER_BAD(pool, email) → Result<User>
    query ← "SELECT * FROM users WHERE email = '" + email + "'"
    RETURN EXECUTE_QUERY(pool, query)

// GOOD: parameterized query; the database driver handles escaping
PROCEDURE FIND_USER(pool, email) → Result<User>
    RETURN EXECUTE_QUERY(pool,
        "SELECT * FROM users WHERE email = $1",
        BIND email)
```

**Prevention:** Always use parameterized queries. Never concatenate user input into SQL, shell commands, or template strings. In Rust, `sqlx` compile-time checked queries make this the path of least resistance.

## A04: Insecure Design

Security controls missing by design, not by implementation bug. No threat modeling performed. No rate limiting on login endpoints. No account lockout after repeated failures. Business logic that does not consider abuse scenarios.

**Prevention:** Perform threat modeling during design. Document trust boundaries. Add rate limiting and abuse controls before writing the first line of code, not after the first incident.

## A05: Security Misconfiguration

Default credentials left in place. Overly permissive CORS. Stack traces exposed to users. Debug mode in production. Unnecessary services enabled. Cloud storage buckets publicly accessible.

**Prevention:** Harden all environments. Disable debug features in production. Review CORS, CSP, and firewall rules. Automate configuration checks in CI/CD.

## A06: Vulnerable and Outdated Components

Using dependencies with known vulnerabilities. Most applications pull in hundreds of transitive dependencies, any of which can introduce a critical flaw.

```bash
# Check dependencies against the RustSec advisory database
cargo audit

# Enforce dependency policies (advisories, licenses, sources)
cargo deny check
```

**Prevention:** Run `cargo audit` in every CI pipeline. Pin dependency versions. Subscribe to security advisories for your ecosystem. Maintain a Software Bill of Materials (SBOM).

## A07: Identification and Authentication Failures

Weak passwords allowed. No multi-factor authentication. Session tokens that never expire. Credential stuffing not mitigated. Password reset flows that leak information.

**Prevention:** Enforce minimum password complexity. Require MFA for privileged accounts. Expire sessions. Rate-limit authentication endpoints. See the [Authentication](02-authentication.md) document for implementation details.

## A08: Software and Data Integrity Failures

Unsigned updates. Deserialization of untrusted data without validation. CI/CD pipelines without integrity verification. Auto-update mechanisms that do not verify signatures.

**Prevention:** Sign all artifacts. Verify checksums on downloads. Use `serde` with strict deserialization (deny unknown fields). Lock CI/CD pipeline dependencies.

## A09: Security Logging and Monitoring Failures

No logging of authentication events. No alerting on suspicious activity. Breaches go undetected for months. The median time to detect a breach is 204 days (IBM 2024).

**Prevention:** Log all authentication events (login, logout, failure, MFA). Alert on anomalies (impossible travel, brute force attempts). Ship logs to a centralized, tamper-resistant system. Test your detection with red team exercises.

## A10: Server-Side Request Forgery (SSRF)

The application fetches a URL supplied by the user without validating the destination, allowing access to internal services, cloud metadata endpoints (169.254.169.254), or other restricted resources.

```text
// BAD: fetches any URL the user provides
PROCEDURE FETCH_URL_BAD(url) → Result<String>
    RETURN HTTP_GET(url).TEXT()

// GOOD: validate against an allowlist of permitted hosts
PROCEDURE FETCH_URL(url) → Result<String>
    parsed ← PARSE_URL(url)
    IF parsed IS INVALID THEN RETURN Error(InvalidUrl)
    host ← parsed.HOST()
    IF host IS NULL THEN RETURN Error(InvalidUrl)

    allowed_hosts ← ["api.example.com", "cdn.example.com"]
    IF host NOT IN allowed_hosts THEN
        RETURN Error(ForbiddenHost)
    END IF

    // Also block private IP ranges
    body ← HTTP_GET(url).TEXT()
    RETURN Ok(body)
```

**Prevention:** Validate and sanitize all user-supplied URLs. Use allowlists for permitted destinations. Block requests to private IP ranges and cloud metadata endpoints. Use network-level controls as a second layer.

## Real-World Incidents

### Equifax Breach (2017)

Equifax failed to patch a known vulnerability in Apache Struts (CVE-2017-5638) for over two months after the patch was available. Attackers exploited it to steal personal data of 147 million people, including Social Security numbers. The breach cost Equifax over $1.4 billion in settlements and remediation. Root cause: no dependency scanning or patch management process. This maps directly to A06 (Vulnerable and Outdated Components). Running `cargo audit` in CI prevents the Rust equivalent.

### Log4Shell (2021)

A remote code execution vulnerability in Log4j (CVE-2021-44228) affected virtually every Java application using the library. Attackers executed arbitrary code by sending a crafted string that the logger evaluated as a JNDI lookup. Most affected organizations did not even know they depended on Log4j transitively. This maps to both A06 (Vulnerable Components) and A08 (Software and Data Integrity Failures). The incident drove widespread adoption of SBOMs and dependency scanning.

### Heartbleed (2014)

A buffer over-read in OpenSSL allowed attackers to read up to 64KB of server memory per request, potentially exposing private keys, session tokens, and user credentials. The bug existed for two years before discovery. This class of vulnerability -- memory safety bugs -- is eliminated by Rust's ownership system. You cannot read beyond buffer bounds in safe Rust. Heartbleed maps to A02 (Cryptographic Failures) and demonstrates why memory-safe languages matter for security-critical infrastructure.

### Capital One Breach (2019)

An SSRF vulnerability in a misconfigured WAF allowed an attacker to access AWS metadata credentials and exfiltrate 106 million customer records from S3 buckets. This maps to A10 (SSRF) and A05 (Security Misconfiguration). The attacker used the metadata endpoint (169.254.169.254) to obtain temporary AWS credentials with excessive permissions.

## Key Takeaways

1. The OWASP Top 10 is a starting point, not a complete threat model. It covers the most common risks but your application may face domain-specific threats.
2. Rust eliminates memory safety vulnerabilities (buffer overflows, use-after-free), but every other category on this list still applies to Rust applications.
3. Most of these vulnerabilities are prevented by straightforward engineering discipline: parameterized queries, authorization checks, dependency scanning, and secure defaults.
4. The cost of prevention is orders of magnitude lower than the cost of exploitation. Equifax spent $1.4 billion on a breach that a dependency scan would have caught.
