# API Versioning & Evolution

## Why Version APIs?

APIs are contracts. Once external clients depend on your API, changing it can break their applications. Versioning provides a mechanism to evolve your API while maintaining backward compatibility for existing consumers.

The cost of breaking an API is proportional to the number of consumers and the difficulty of updating them. A private microservice API with two consumers can break freely. A public API with 10,000 integrations needs a careful versioning strategy.

## Versioning Strategies

### URL Path Versioning

```
GET /v1/users/123
GET /v2/users/123
```

The most common approach. The version is explicit in the URL, making it visible in logs, documentation, and debugging.

**Pros:**
- Explicit and obvious
- Easy to route at the load balancer or gateway level
- Simple to document and test
- Clients can't accidentally use the wrong version

**Cons:**
- URL "pollution" — the version isn't part of the resource identity
- Encourages large, infrequent version bumps instead of incremental evolution
- Maintaining multiple route trees increases code complexity

**Used by:** Google Cloud APIs (`/v1/`, `/v2/`), Twitter API, most public APIs.

### Header Versioning

```
GET /users/123
Accept: application/vnd.myapi+json;version=2

# or custom header
GET /users/123
X-API-Version: 2
```

The version is in the request headers, keeping URLs clean.

**Pros:**
- Clean URLs that represent true resource identity
- Supports content negotiation (different versions = different representations)
- Follows HTTP semantics more closely

**Cons:**
- Hidden — easy to forget when making requests
- Harder to test (can't just change the URL in a browser)
- Not visible in access logs by default
- Curl/Postman testing requires extra configuration

**Used by:** GitHub REST API (`Accept: application/vnd.github.v3+json`).

### Query Parameter Versioning

```
GET /users/123?version=2
GET /users/123?api-version=2024-03-15
```

**Pros:**
- Explicit and visible
- Easy to test (just add a query parameter)
- Can default to latest version if omitted

**Cons:**
- Pollutes the query string (mixes versioning with filtering/pagination)
- Caching complications (different versions = different cache keys)

**Used by:** Azure APIs (`api-version=2024-01-01`).

## Stripe's Date-Based Versioning

Stripe's approach is the most sophisticated and widely studied versioning strategy.

```
# Pin to a specific API version
Stripe-Version: 2024-03-15

# Or set it in the dashboard — all requests from your account default to that version
```

**How it works:**
1. Every breaking change gets a new dated version (e.g., `2024-03-15`)
2. Each account is pinned to the version it was created on
3. Developers can upgrade at their own pace by changing the version header or dashboard setting
4. Stripe maintains all versions simultaneously — old versions never stop working
5. Internally, Stripe uses a version compatibility layer that transforms requests/responses between versions

**Why it works:**
- No "v1 vs v2" cliff — changes are granular and incremental
- Developers upgrade on their schedule, not yours
- Version dates are self-documenting (you know when the change happened)
- Stripe's changelog shows exactly what changed in each version

**The engineering cost:** Stripe maintains version transformation code for every breaking change. This is significant engineering effort, but it's justified by the value of not breaking thousands of integrations.

## Backward Compatibility Rules

Whether or not you use formal versioning, these rules determine what is and isn't a breaking change.

### Non-Breaking Changes (safe to make anytime)

- Adding a new optional field to a response
- Adding a new optional query parameter or request field
- Adding a new endpoint
- Adding a new enum value (if clients handle unknowns gracefully)
- Adding a new HTTP method to an existing resource
- Relaxing validation (accepting more input than before)

### Breaking Changes (require a new version or migration)

- Removing or renaming a response field
- Removing or renaming an endpoint
- Changing a field's type (string to integer)
- Adding a new required field to a request
- Changing the meaning/semantics of an existing field
- Tightening validation (rejecting previously accepted input)
- Changing authentication requirements
- Changing error response formats

### Gray Areas

- Changing default pagination size (may break clients with hardcoded expectations)
- Adding a required field with a sensible default (technically breaking, usually safe)
- Changing sort order of list responses (if undocumented, fair game; if documented, breaking)

## Deprecation

Deprecation is the process of marking an API version or feature as obsolete before removing it.

**A good deprecation process:**

1. **Announce** — Communicate the deprecation timeline well in advance (6-12 months minimum for public APIs)
2. **Document** — Mark deprecated fields/endpoints in documentation and OpenAPI specs
3. **Signal** — Return deprecation headers in responses:
   ```
   Deprecation: true
   Sunset: Sat, 01 Mar 2025 00:00:00 GMT
   Link: <https://docs.api.com/migration>; rel="deprecation"
   ```
4. **Warn** — Log usage of deprecated features. Contact heavy users directly.
5. **Migrate** — Provide migration guides and tooling
6. **Remove** — Only after the sunset date and after confirming usage has dropped to acceptable levels

**Never silently remove an API.** Even if you think nobody uses it, someone does.

## Evolution Strategies

### The Expand/Contract Pattern

Used for evolving fields without breaking clients:

1. **Expand**: Add the new field alongside the old one
   ```json
   { "name": "Alice", "full_name": "Alice Smith" }
   ```
2. Migrate clients to use the new field
3. **Contract**: Remove the old field in the next version

### Additive-Only APIs

Some teams avoid versioning entirely by committing to never make breaking changes:

- Only add new fields, endpoints, and parameters
- Never remove or rename anything
- Use feature flags or capability negotiation instead of versions

This works for internal APIs and simple public APIs. It breaks down when you need to fix design mistakes or fundamentally restructure resources.

### API Gateways for Version Management

An API gateway can handle version transformation at the edge:

```
Client (v1) --> Gateway --> transforms to v3 format --> Backend (v3)
Client (v2) --> Gateway --> transforms to v3 format --> Backend (v3)
Client (v3) --> Gateway --> passes through           --> Backend (v3)
```

The backend only implements the latest version. The gateway handles backward compatibility transformations. This is essentially what Stripe does internally.

## Practical Guidelines

1. **Start without versioning.** Design carefully, follow the backward compatibility rules, and only introduce formal versioning when you actually need a breaking change.

2. **If you version, use URL path versioning** for public APIs (most developer-friendly) or **date-based versioning** if you can invest in the infrastructure.

3. **Version the API, not individual endpoints.** `/v2/users` and `/v2/orders` should be part of the same version, not independently versioned.

4. **Communicate changes obsessively.** Changelog, email, migration guides, deprecation headers. Over-communication is better than surprising your users.

5. **Set a support window.** "We support the last 3 major versions" or "We support versions for 24 months after release." Be explicit.

6. **Test backward compatibility.** Run your test suite against old API versions. If you use Stripe's approach, write transformation tests.

7. **Monitor version usage.** Track which versions clients are using. This tells you when it's safe to sunset old versions and identifies clients who need migration help.

## Real-World Examples

| Company | Strategy | Details |
|---------|----------|---------|
| **Stripe** | Date-based headers | `Stripe-Version: 2024-03-15`. All versions supported indefinitely. |
| **GitHub** | URL path + header | `/v3/` for REST. Header for minor changes. GraphQL API is unversioned (additive-only). |
| **Google Cloud** | URL path | `/v1/`, `/v2/`. Stability levels: alpha, beta, GA. |
| **Azure** | Query param dates | `api-version=2024-01-01`. Explicit in every request. |
| **Twilio** | URL path + dates | `/2010-04-01/Accounts`. Date in URL indicates the API generation. |
| **Slack** | No versioning | Additive-only approach. New features via new endpoints/fields. |
