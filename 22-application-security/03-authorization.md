# Authorization

Authorization answers the question: "What are you allowed to do?" It is enforced after authentication and determines whether a verified user can perform a specific action on a specific resource. Getting authorization wrong leads to the OWASP Top 10's number-one risk: Broken Access Control.

## Role-Based Access Control (RBAC)

Users are assigned roles. Roles have permissions. The system checks whether the user's role grants the required permission for an action. RBAC is simple, auditable, and sufficient for most applications.

```text
ENUMERATION Permission
    ReadOrders, CreateOrders, DeleteOrders, ManageUsers, ViewAnalytics

ENUMERATION Role
    Viewer, Editor, Admin

PROCEDURE PERMISSIONS_FOR_ROLE(role) → Set<Permission>
    MATCH role
        CASE Viewer  → RETURN {ReadOrders, ViewAnalytics}
        CASE Editor  → RETURN {ReadOrders, CreateOrders, ViewAnalytics}
        CASE Admin   → RETURN {ReadOrders, CreateOrders, DeleteOrders,
                                ManageUsers, ViewAnalytics}

PROCEDURE AUTHORIZE(user, required_permission) → Result
    perms ← PERMISSIONS_FOR_ROLE(user.role)
    IF required_permission IN perms THEN
        RETURN Ok
    ELSE
        RETURN Error(403 FORBIDDEN)
    END IF
```

**When to use RBAC:** Your authorization rules map cleanly to a small number of roles (viewer, editor, admin). You need simple, auditable access control. Most web applications and internal tools fit this model.

**Limitation: role explosion.** When you start creating roles like `editor-but-only-for-project-X` or `viewer-with-export-permission`, RBAC is reaching its limits. This is when you should consider ABAC.

## Attribute-Based Access Control (ABAC)

Decisions based on attributes of the user, the resource, the action, and the environment. More flexible than RBAC, but more complex to implement and audit.

Example policy: "A user can edit a document if they are in the same department AND the document is not archived AND it is during business hours."

```text
STRUCTURE PolicyContext
    user : AuthenticatedUser
    resource : Document
    action : Action

ENUMERATION Action
    Read, Edit, Delete

PROCEDURE EVALUATE_POLICY(ctx) → Boolean
    MATCH ctx.action
        CASE Edit:
            // User must be in the same department
            same_dept ← ctx.user.department = ctx.resource.department
            // Document must not be archived
            not_archived ← NOT ctx.resource.is_archived
            // Must be during business hours (9-17 UTC, weekdays)
            now ← CURRENT_UTC_TIME()
            is_business_hours ← now.hour ≥ 9
                AND now.hour < 17
                AND now.weekday NOT IN {Saturday, Sunday}
            RETURN same_dept AND not_archived AND is_business_hours

        CASE Read:
            // Any authenticated user in the same department can read
            RETURN ctx.user.department = ctx.resource.department

        CASE Delete:
            // Only department heads can delete
            RETURN ctx.user.is_department_head
                AND ctx.user.department = ctx.resource.department
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

```text
PROCEDURE REQUIRE_PERMISSION(required_permission) → Middleware
    RETURN MIDDLEWARE(request, next):
        user ← request.EXTENSIONS().GET(AuthenticatedUser)
        IF user IS NULL THEN RETURN Error(401 UNAUTHORIZED)

        AUTHORIZE(user, required_permission)
        IF authorization FAILS THEN RETURN Error(403 FORBIDDEN)

        RETURN next.RUN(request)

// Usage in router
app ← Router
    .ROUTE("/orders", GET(LIST_ORDERS))
    .ADD_LAYER(REQUIRE_PERMISSION(ReadOrders))
    .ROUTE("/admin/users", GET(LIST_USERS))
    .ADD_LAYER(REQUIRE_PERMISSION(ManageUsers))
```

This pattern ensures that authorization is enforced at the routing layer, not scattered across individual handler functions. Missing an authorization check on a new endpoint is a common source of access control vulnerabilities.

## Resource-Based Authorization

Sometimes authorization depends not just on the role, but on the relationship between the user and the specific resource. This is common for multi-tenant systems and collaborative applications.

```text
PROCEDURE UPDATE_PROJECT(project_id, user, pool, payload) → Result<Project>
    project ← DB_GET_PROJECT(pool, project_id)
    IF project NOT FOUND THEN RETURN Error(404 NOT FOUND)

    // Check: user must be a member of this project with edit permission
    membership ← DB_GET_PROJECT_MEMBERSHIP(pool, project_id, user.id)
    IF membership NOT FOUND THEN RETURN Error(403 FORBIDDEN)

    IF membership.role < ProjectRole.Editor THEN
        RETURN Error(403 FORBIDDEN)
    END IF

    updated ← DB_UPDATE_PROJECT(pool, project_id, payload)
    RETURN Ok(updated)
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
