# Manifest RBAC — Reference

Stack-neutral specification. Map entities and APIs to any language, ORM, and framework.

---

## Permission Key Format

```
<resource>:<action>
```

| Key | Typical use |
|-----|-------------|
| `task:read` | List/view tasks |
| `task:create` | Create tasks |
| `role:manage` | Role/group CRUD (tenant) |
| `platform_user:read` | Cross-tenant user list |
| `platform_role:manage` | Platform role editor |
| `auth:read` | Session / me endpoint |
| `auth:delete` | Logout |

Common actions: `read`, `create`, `update`, `delete`, `manage`, `upload`, plus domain verbs (`approve`, `issue`, `request`).

---

## Wildcard Matching (implement once, share server + client)

```
matches_grant(grant_resource, grant_action, required_resource, required_action):
  if grant_resource in ('*', required_resource)
     and grant_action in ('*', required_action):
    return true
  if grant_resource == required_resource and grant_action == 'manage':
    return true
  return false
```

Evaluation order: collect all grants → apply **DENY** first → then **ALLOW**.

```
permission_granted(held_keys[], required_key):
  parse required_key → (resource, action)
  for each held key:
    parse → (gr, ga)
    if matches_grant(gr, ga, resource, action): return true
  return false
```

---

## Tenant vs Platform

| | Tenant | Platform |
|---|--------|----------|
| Resource prefix | Normal names | `platform_*` |
| Role tenant_id | Set | `NULL` |
| Users | Company owner, staff | Platform operators |
| Role API | `/access/roles` | `/access/platform/roles` |

**Shared HTTP paths:** same URL may map to different resources per scope. When a platform admin hits a shared path, alias tenant resource → platform resource:

```
tenant_resource → platform_resource
dashboard       → platform_overview
user            → platform_user
site            → platform_site
role            → platform_role
setting         → platform_setting
auth            → platform_auth
```

---

## Route Manifest Schema

```json
{
  "tenant_routes": [
    {
      "method": "GET",
      "path": "/tasks",
      "resource": "task",
      "action": "read",
      "description": "List tasks"
    }
  ],
  "platform_routes": [],
  "exempt_routes": [
    {
      "method": "POST",
      "path": "/auth/login",
      "category": "public",
      "description": "Staff login"
    },
    {
      "method": "POST",
      "path": "/portal/auth/login",
      "category": "portal",
      "description": "Secondary portal login"
    }
  ],
  "default_tenant_roles": {
    "owner": { "permissions": "*", "is_system": true, "description": "Full access" },
    "staff": {
      "permissions": ["task:read", "auth:read"],
      "is_system": true,
      "description": "Default staff"
    }
  },
  "default_platform_roles": {}
}
```

Exempt categories: `public` (no auth), `portal` (secondary auth). Never assignable to roles.

---

## Entity Relationship (ORM-agnostic)

```
iam_resource
  name (PK)

iam_permission
  id (PK)
  resource → iam_resource.name
  action
  effect (ALLOW | DENY)
  UNIQUE (resource, action, effect)

iam_role
  id (PK)
  tenant_id (nullable — NULL = platform role)
  name
  description
  is_system
  external_id (optional legacy slug)

iam_role_permission
  role_id → iam_role
  permission_id → iam_permission
  PK (role_id, permission_id)

iam_role_binding
  id (PK)
  role_id → iam_role
  subject_type (USER | GROUP)
  subject_id
  tenant_id (nullable)

iam_direct_permission_binding
  id (PK)
  permission_id → iam_permission
  subject_type (USER)
  subject_id
  tenant_id (nullable)

iam_group (optional)
  id, tenant_id, name, scope_refs (e.g. site ids)

iam_group_member
  group_id, user_id
  PK (group_id, user_id)
```

---

## Grant Resolution Algorithm

```
get_effective_permission_keys(principal, scope):

  if principal is tenant_owner:
    return all tenant manifest keys

  grants = []

  # Direct bindings
  grants += direct_bindings(principal.id, tenant_id_for_scope)

  # Role bindings (user + groups)
  group_ids = group_memberships(principal.id)  # tenant scope only
  subject_ids = [principal.id] + group_ids
  for binding in role_bindings(subject_ids, tenant_id_for_scope):
    grants += permissions_of_role(binding.role_id)

  return dedupe(evaluate_allow_deny(grants))


check_permission(principal, resource, action):

  if tenant_owner_bypass(principal): return true

  scope = platform if principal.is_platform_admin else tenant
  if scope == platform and not resource.startswith('platform_'):
    resource = TENANT_TO_PLATFORM.get(resource, resource)

  keys = get_effective_permission_keys(principal, scope)
  return permission_granted(keys, permission_key(resource, action))
```

---

## User Grants Response Shape

```json
{
  "user_id": "uuid",
  "direct": ["wage:update"],
  "inherited": ["task:read", "attendance:read"],
  "effective": ["wage:update", "task:read", "attendance:read"],
  "read_only": false
}
```

`read_only: true` for protected super-users (owner, platform super-admin).

---

## Access API Contract

### Catalog

`GET /access/permissions` (tenant) or `GET /access/platform/permissions` (platform admin)

```json
{
  "permissions": [
    {
      "key": "task:read",
      "resource": "task",
      "action": "read",
      "description": "List tasks",
      "routes": [{ "method": "GET", "path": "/tasks" }]
    }
  ],
  "exempt_routes": [
    { "method": "POST", "path": "/auth/login", "category": "public" }
  ],
  "summary": { "permission_count": 42, "exempt_route_count": 12 }
}
```

### Roles

| Method | Path | Body |
|--------|------|------|
| GET | `/access/roles` | — |
| POST | `/access/roles` | `{ name, description, permissions: string[] }` |
| PATCH | `/access/roles/:id` | partial |
| DELETE | `/access/roles/:id` | — |

Platform: same under `/access/platform/roles`. Validate `platform_*` prefix on write.

### Current user

`GET /access/me` → `{ permissions: string[] }` (effective keys)

### Direct grants

`GET/PUT /access/users/:id/permissions` → user grants response shape above.

---

## Write Validation

```
normalize_permission_keys(input[], platform_scope=false):

  keys = map each input:
    if contains ':' → use as-is
    else → legacy_to_manifest_key(input)

  if platform_scope:
    reject any key where resource does not start with 'platform_'

  if tenant_scope:
    reject any key where resource starts with 'platform_'

  return keys
```

---

## Permission Matrix UI Spec

Behavioral requirements (any component library):

### Picker mode (role / user editor)

- Table rows = catalog permissions filtered by search + scope tab
- Checkbox column only for **assignable** rows (matches save scope)
- Toolbar: search, Select all, Selected only
- Scope tabs: All | Tenant | Platform
- Footer: `{selected} of {assignable_in_view} grants selected`
- Selected keys panel (optional chips/list)

### Catalog mode (readonly audit)

- No checkboxes
- Extra column: HTTP routes protected by each key
- Tabs: Protected | Public/Portal (exempt routes)

### Assignable filter rules

| Filter | Checkboxes on |
|--------|---------------|
| `tenant` | Non-`platform_*` in active tab |
| `platform` | `platform_*` only |
| `all` | All in active tab (validate on save) |

---

## Legacy Key Migration

If migrating from dot notation (`users.view`):

```
legacy_to_manifest_key(key):
  map = { 'users.view': 'user:read', 'users.manage': 'user:update', ... }
  return map.get(key, key)
```

Accept legacy at API input; store and transmit colon keys internally.

---

## When to Deviate

| Scenario | Change |
|----------|--------|
| Single tenant | Drop platform manifest, tenant_id, admin UI |
| No custom roles | Seed defaults; skip role CRUD |
| Row-level access | IAM for modules + ownership checks in services |
| External IdP | Sync groups → role bindings |
| Too many actions | Collapse to `manage` + wildcards |
| API-only backend | Skip matrix UI; expose catalog JSON |
| GraphQL | Field directives instead of route middleware |

---

## Audit Checklist

1. Manifest row for every protected route
2. Route enforcement metadata matches manifest resource/action
3. Sync job created all permissions in DB
4. Catalog API ⊆ manifest (plus routes array)
5. Default roles reference valid keys
6. Client `matches_grant` identical to server
7. Platform/tenant validation on role writes
