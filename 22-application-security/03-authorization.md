# Authorization

Authorization answers the question: "What are you allowed to do?" It is enforced after authentication and determines whether a verified user can perform a specific action on a specific resource. Getting authorization wrong leads to the OWASP Top 10's number-one risk: Broken Access Control.

## Role-Based Access Control (RBAC)

Users are assigned roles. Roles have permissions. The system checks whether the user's role grants the required permission for an action. RBAC is simple, auditable, and sufficient for most applications.

```rust
use std::collections::HashSet;
use axum::http::StatusCode;

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
enum Permission {
    ReadOrders,
    CreateOrders,
    DeleteOrders,
    ManageUsers,
    ViewAnalytics,
}

#[derive(Debug, Clone)]
enum Role {
    Viewer,
    Editor,
    Admin,
}

fn permissions_for_role(role: &Role) -> HashSet<Permission> {
    match role {
        Role::Viewer => HashSet::from([Permission::ReadOrders, Permission::ViewAnalytics]),
        Role::Editor => HashSet::from([
            Permission::ReadOrders,
            Permission::CreateOrders,
            Permission::ViewAnalytics,
        ]),
        Role::Admin => HashSet::from([
            Permission::ReadOrders,
            Permission::CreateOrders,
            Permission::DeleteOrders,
            Permission::ManageUsers,
            Permission::ViewAnalytics,
        ]),
    }
}

fn authorize(user: &AuthenticatedUser, required: Permission) -> Result<(), StatusCode> {
    let perms = permissions_for_role(&user.role);
    if perms.contains(&required) {
        Ok(())
    } else {
        Err(StatusCode::FORBIDDEN)
    }
}
```

**When to use RBAC:** Your authorization rules map cleanly to a small number of roles (viewer, editor, admin). You need simple, auditable access control. Most web applications and internal tools fit this model.

**Limitation: role explosion.** When you start creating roles like `editor-but-only-for-project-X` or `viewer-with-export-permission`, RBAC is reaching its limits. This is when you should consider ABAC.

## Attribute-Based Access Control (ABAC)

Decisions based on attributes of the user, the resource, the action, and the environment. More flexible than RBAC, but more complex to implement and audit.

Example policy: "A user can edit a document if they are in the same department AND the document is not archived AND it is during business hours."

```rust
use chrono::{Utc, Weekday, Timelike};

struct PolicyContext<'a> {
    user: &'a AuthenticatedUser,
    resource: &'a Document,
    action: Action,
}

enum Action {
    Read,
    Edit,
    Delete,
}

fn evaluate_policy(ctx: &PolicyContext) -> bool {
    match ctx.action {
        Action::Edit => {
            // User must be in the same department
            let same_dept = ctx.user.department == ctx.resource.department;
            // Document must not be archived
            let not_archived = !ctx.resource.is_archived;
            // Must be during business hours (9-17 UTC, weekdays)
            let now = Utc::now();
            let is_business_hours = now.hour() >= 9
                && now.hour() < 17
                && !matches!(now.weekday(), Weekday::Sat | Weekday::Sun);

            same_dept && not_archived && is_business_hours
        }
        Action::Read => {
            // Any authenticated user in the same department can read
            ctx.user.department == ctx.resource.department
        }
        Action::Delete => {
            // Only department heads can delete
            ctx.user.is_department_head && ctx.user.department == ctx.resource.department
        }
    }
}
```

**When to use ABAC:** Authorization depends on resource attributes, environmental conditions, or relationships. RBAC leads to too many roles or cannot express your access rules. Common in healthcare (HIPAA), finance, and multi-tenant SaaS.

## Policy Engines

For complex authorization, externalize the policy from application code. Policy engines evaluate rules written in a dedicated policy language, making policies auditable, testable, and changeable without code deployments.

**Open Policy Agent (OPA):** A general-purpose policy engine. Policies written in Rego (a declarative language). The application sends a structured authorization request; OPA returns a decision.

**Cedar (by AWS):** A policy language designed specifically for authorization. Used in AWS Verified Permissions. Policies are human-readable and formally verifiable.

Example Cedar policy:
```
permit(
    principal in Role::"editor",
    action in [Action::"read", Action::"edit"],
    resource in Folder::"project-x"
) when {
    resource.is_archived == false
};
```

**When to use a policy engine:** You have more than a handful of authorization rules. Policies need to change without code deployments. You need formal auditing of who can do what. Multiple services need consistent authorization logic.

## Authorization Middleware in Axum

Enforce authorization as middleware so that individual handlers do not need to repeat checks.

```rust
use axum::{
    extract::Request,
    http::StatusCode,
    middleware::{self, Next},
    response::Response,
    Extension,
};

async fn require_permission(
    required: Permission,
) -> impl Fn(Request, Next) -> futures::future::BoxFuture<'static, Result<Response, StatusCode>> {
    move |request: Request, next: Next| {
        let required = required.clone();
        Box::pin(async move {
            let user = request
                .extensions()
                .get::<AuthenticatedUser>()
                .ok_or(StatusCode::UNAUTHORIZED)?;

            authorize(user, required)?;
            Ok(next.run(request).await)
        })
    }
}

// Usage in router
let app = Router::new()
    .route("/orders", get(list_orders))
    .route_layer(middleware::from_fn(move |req, next| {
        require_permission(Permission::ReadOrders)(req, next)
    }))
    .route("/admin/users", get(list_users))
    .route_layer(middleware::from_fn(move |req, next| {
        require_permission(Permission::ManageUsers)(req, next)
    }));
```

This pattern ensures that authorization is enforced at the routing layer, not scattered across individual handler functions. Missing an authorization check on a new endpoint is a common source of access control vulnerabilities.

## Resource-Based Authorization

Sometimes authorization depends not just on the role, but on the relationship between the user and the specific resource. This is common for multi-tenant systems and collaborative applications.

```rust
async fn update_project(
    Path(project_id): Path<u64>,
    Extension(user): Extension<AuthenticatedUser>,
    Extension(pool): Extension<PgPool>,
    Json(payload): Json<UpdateProject>,
) -> Result<Json<Project>, StatusCode> {
    let project = db::get_project(&pool, project_id)
        .await
        .map_err(|_| StatusCode::NOT_FOUND)?;

    // Check: user must be a member of this project with edit permission
    let membership = db::get_project_membership(&pool, project_id, user.id)
        .await
        .map_err(|_| StatusCode::FORBIDDEN)?;

    if membership.role < ProjectRole::Editor {
        return Err(StatusCode::FORBIDDEN);
    }

    let updated = db::update_project(&pool, project_id, payload)
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    Ok(Json(updated))
}
```

## Real-World Examples

### Broken Access Control in Practice

In 2019, a vulnerability in First American Financial's website exposed 885 million records containing bank account numbers, Social Security numbers, and mortgage documents. The flaw was a simple IDOR: incrementing a document ID in the URL returned any customer's documents with no authorization check. This is the most basic form of broken access control and remains depressingly common.

### Principle of Least Privilege

Google's BeyondCorp model treats every network as untrusted and requires authorization for every request, even from internal services. This zero-trust approach means that a compromised internal service cannot automatically access other internal services. Authorization is not just for user-facing endpoints -- service-to-service communication needs it too.

## RBAC vs. ABAC Trade-offs

| Aspect | RBAC | ABAC |
|--------|------|------|
| Complexity | Low | High |
| Flexibility | Limited to role definitions | Unlimited attribute combinations |
| Auditability | Easy (list roles and permissions) | Harder (policies can interact) |
| Performance | Fast (set membership check) | Variable (policy evaluation) |
| Maintenance | Role explosion risk | Policy sprawl risk |
| Best for | Most applications | Complex, multi-tenant, regulated systems |

## Common Mistakes

- **Relying on the UI to enforce authorization.** Attackers bypass the UI entirely and call your API directly. Every endpoint must independently verify authorization.
- **Overly broad CORS policies.** Setting `Access-Control-Allow-Origin: *` on an API that uses cookies or returns sensitive data allows any website to make authenticated requests on behalf of your users.
- **Checking authorization once at login.** Authorization must be checked on every request. A user's permissions can change between requests (role revoked, resource transferred, account suspended).
- **Not logging authorization failures.** Failed authorization attempts are a signal of probing or misconfiguration. Log them and alert on anomalous patterns.
- **Confusing authentication with authorization.** "The user is logged in" does not mean "the user can do this." These are separate concerns that must be implemented separately.

## Key Takeaways

1. Deny by default. Every endpoint must explicitly grant access, not implicitly allow it.
2. RBAC handles most use cases. Reach for ABAC or a policy engine only when RBAC cannot express your rules.
3. Enforce authorization in middleware or at the routing layer, not scattered across handler functions.
4. Authorization applies to service-to-service communication, not just user-facing endpoints.
5. Log and monitor authorization failures. They are early indicators of attacks or misconfigurations.
