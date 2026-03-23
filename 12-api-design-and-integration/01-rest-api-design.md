# REST API Design

## What is REST?

REST (Representational State Transfer) is an architectural style for designing web APIs around *resources* — things that have identity and state. Coined by Roy Fielding in his 2000 dissertation, REST has become the dominant paradigm for web APIs because it maps naturally to HTTP and is simple to understand.

REST is not a protocol or standard — it's a set of constraints. An API that follows these constraints is called "RESTful."

## Core Principles

- **Resources** are identified by URLs: `/users/123`, `/orders/456/items`
- **HTTP methods** map to operations: GET (read), POST (create), PUT (replace), PATCH (update), DELETE (remove)
- **Stateless** — Each request contains all information needed to process it. No server-side sessions between requests.
- **Representations** — Resources can have multiple representations (JSON, XML, HTML). Clients specify preference via `Accept` header.
- **Uniform interface** — Consistent URL patterns, standard HTTP semantics, self-descriptive messages.

## Resource Modeling

The most important decision in REST API design is choosing your resources. Resources should model your domain, not your database tables.

```
# Good — nouns, hierarchical
GET    /users                    # List users
GET    /users/123                # Get user 123
POST   /users                    # Create a user
PATCH  /users/123                # Update user 123
DELETE /users/123                # Delete user 123
GET    /users/123/orders         # List orders for user 123
GET    /users/123/orders/456     # Get order 456 for user 123

# Bad — verbs, RPC-style
POST   /getUser                  # Not RESTful
POST   /createUser               # The HTTP method already says "create"
GET    /getUserOrders?userId=123  # Hierarchy not reflected in URL
```

**Guidelines for resource naming:**
- Use plural nouns: `/users`, not `/user`
- Use hierarchy to express relationships: `/users/123/orders`
- Keep URLs shallow — two levels of nesting is usually enough
- Use hyphens for multi-word resources: `/order-items`, not `/orderItems`
- Avoid actions in URLs. If you must, treat them as sub-resources: `POST /orders/123/cancel`

## HTTP Methods

| Method | Semantics | Idempotent | Safe | Example |
|--------|-----------|------------|------|---------|
| **GET** | Read a resource | Yes | Yes | `GET /users/123` |
| **POST** | Create a resource | No | No | `POST /users` |
| **PUT** | Replace a resource entirely | Yes | No | `PUT /users/123` |
| **PATCH** | Partially update a resource | Yes* | No | `PATCH /users/123` |
| **DELETE** | Remove a resource | Yes | No | `DELETE /users/123` |

*PATCH is idempotent when using JSON Merge Patch or JSON Patch. Not guaranteed in all implementations.

**Idempotent** means calling it multiple times produces the same result as calling it once. This matters for retries — safe to retry a PUT, dangerous to retry a POST without an idempotency key.

## HTTP Status Codes

Status codes are part of the API contract, not decoration. Use them correctly.

| Code | Meaning | When to use |
|------|---------|-------------|
| **200** | OK | Successful GET, PATCH, DELETE |
| **201** | Created | Successful POST that created a resource |
| **204** | No Content | Successful DELETE with no response body |
| **400** | Bad Request | Invalid input, validation failure |
| **401** | Unauthorized | Missing or invalid authentication |
| **403** | Forbidden | Authenticated but not authorized |
| **404** | Not Found | Resource doesn't exist |
| **409** | Conflict | Duplicate resource, state conflict |
| **422** | Unprocessable Entity | Valid syntax but semantic errors |
| **429** | Too Many Requests | Rate limit exceeded |
| **500** | Internal Server Error | Unexpected server failure |

**Error response design — be specific and actionable:**

```json
{
  "error": {
    "type": "validation_error",
    "message": "Request validation failed",
    "details": [
      { "field": "email", "message": "Invalid email format" },
      { "field": "age", "message": "Must be a positive integer" }
    ]
  }
}
```

## Pagination

Never return unbounded lists. There are two main approaches.

### Offset-Based Pagination

```
GET /users?page=3&per_page=20
```

Simple to implement, but has problems: skipping rows is O(N) in most databases, and inserting/deleting rows causes items to shift between pages.

### Cursor-Based Pagination

```
GET /users?after=cursor_abc123&limit=20

# Response:
{
  "data": [...],
  "pagination": {
    "next_cursor": "cursor_def456",
    "has_more": true
  }
}
```

Cursors are opaque tokens (typically base64-encoded primary keys or timestamps). They perform consistently regardless of dataset size and handle concurrent inserts/deletes correctly. Stripe, Slack, and Facebook all use cursor-based pagination.

**When to use which:** Offset for small datasets or when users need "jump to page 50." Cursor for everything else.

## Filtering, Sorting, and Field Selection

```
GET /orders?status=pending&sort=-created_at&fields=id,total,status
```

- **Filtering**: Use query parameters matching field names. For complex filters, consider a filter syntax: `?filter=status:eq:pending,total:gt:100`
- **Sorting**: Prefix with `-` for descending. Multiple fields: `?sort=-created_at,name`
- **Field selection**: Let clients request only the fields they need. Reduces payload size for mobile clients.

## HATEOAS

Hypermedia As The Engine Of Application State — responses include links to related resources and available actions.

```json
{
  "id": "order_123",
  "status": "pending",
  "total": 99.50,
  "_links": {
    "self": { "href": "/orders/order_123" },
    "cancel": { "href": "/orders/order_123/cancel", "method": "POST" },
    "items": { "href": "/orders/order_123/items" },
    "customer": { "href": "/users/user_456" }
  }
}
```

HATEOAS is the most debated REST constraint. In theory, it makes APIs self-discoverable. In practice, most APIs skip it because clients are usually coded against known endpoints. GitHub's REST API uses it well — every response includes URL links to related resources.

## Rust axum Example

```rust
use axum::{
    extract::{Path, Query, State},
    http::StatusCode,
    routing::{get, post},
    Json, Router,
};
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
struct ListParams {
    after: Option<String>,
    limit: Option<u32>,
    status: Option<String>,
    sort: Option<String>,
}

#[derive(Serialize)]
struct PaginatedResponse<T: Serialize> {
    data: Vec<T>,
    pagination: Pagination,
}

#[derive(Serialize)]
struct Pagination {
    next_cursor: Option<String>,
    has_more: bool,
}

async fn list_orders(
    State(db): State<AppState>,
    Query(params): Query<ListParams>,
) -> Json<PaginatedResponse<Order>> {
    let limit = params.limit.unwrap_or(20).min(100);
    let orders = db.list_orders(params.after, limit + 1, params.status).await;

    let has_more = orders.len() > limit as usize;
    let data: Vec<Order> = orders.into_iter().take(limit as usize).collect();
    let next_cursor = if has_more { data.last().map(|o| o.id.clone()) } else { None };

    Json(PaginatedResponse {
        data,
        pagination: Pagination { next_cursor, has_more },
    })
}

async fn get_order(
    State(db): State<AppState>,
    Path(id): Path<String>,
) -> Result<Json<Order>, StatusCode> {
    db.get_order(&id)
        .await
        .map(Json)
        .ok_or(StatusCode::NOT_FOUND)
}

async fn create_order(
    State(db): State<AppState>,
    Json(input): Json<CreateOrderInput>,
) -> (StatusCode, Json<Order>) {
    let order = db.create_order(input).await;
    (StatusCode::CREATED, Json(order))
}

fn app(state: AppState) -> Router {
    Router::new()
        .route("/orders", get(list_orders).post(create_order))
        .route("/orders/{id}", get(get_order))
        .with_state(state)
}
```

## Model APIs: Stripe and Google

### Stripe

Stripe's API is widely regarded as the gold standard for REST design:
- Consistent `snake_case` naming everywhere
- Every object has an `id` field and an `object` field (e.g., `"object": "customer"`)
- Expandable fields: `GET /charges/ch_123?expand[]=customer` inlines the customer object instead of returning just the ID
- Idempotency keys: `Idempotency-Key: abc123` header makes POST requests safe to retry
- Date-based versioning: `Stripe-Version: 2024-03-15`

### Google Cloud APIs

Google's publicly available [API Design Guide](https://cloud.google.com/apis/design) standardizes:
- Resource-oriented design with standard methods (List, Get, Create, Update, Delete)
- Consistent error model across all APIs
- Standard fields: `name`, `create_time`, `update_time`, `delete_time`, `etag`
- Long-running operations pattern for async work

Both demonstrate that consistency and predictability matter more than cleverness.

## Common Mistakes

- **Inconsistent naming** — Mixing `camelCase` and `snake_case`. Pick one and enforce it.
- **Using POST for everything** — Treating REST as RPC. Use the right HTTP method.
- **Leaking internals** — Exposing database column names or internal IDs. The API is a public contract.
- **No pagination** — Works with 10 records, crashes with 10 million.
- **Poor error messages** — `{"error": "Bad Request"}` tells the client nothing.
