---
name: manifest-rbac
description: >-
  Implements Manifest RBAC: route manifest sync, resource:action permission keys,
  role and direct user bindings, tenant vs platform scopes, and permission matrix
  UI. Stack-agnostic — any backend, database, and frontend. Use when adding IAM,
  RBAC, permissions, roles, access control, authorization middleware, route guards,
  or manifest-driven RBAC to any project.
---

# Manifest RBAC

**Stack-agnostic** RBAC pattern. The concepts are fixed; the wiring adapts to whatever framework, ORM, and UI library the project uses.

Deep reference: [reference.md](reference.md) · Stack adapters: [adapters.md](adapters.md) · Examples: [examples.md](examples.md)

---

## First Step: Detect the Stack

Before implementing, read the project's existing auth and structure. Then map Manifest RBAC concepts to native primitives:

| Concept | Your project equivalent |
|---------|-------------------------|
| Route manifest | JSON/YAML module or code registry — language-agnostic |
| Require permission on route | Decorator, middleware, filter, policy, or route attribute |
| Permission guard | Global middleware / interceptor / before_action |
| IAM service | Service class, module, or package — any language |
| Persistence | Any DB/ORM (SQL, NoSQL, Prisma, Drizzle, SQLAlchemy, Eloquent…) |
| Permission matrix UI | Any component framework (React, Vue, Svelte, Blade, HTMX…) |

**Never copy a reference stack wholesale.** Implement the same behavior using the project's conventions.

---

## Core Rules (non-negotiable)

1. **Manifest is source of truth** — every protected route has a manifest row; route enforcement must match.
2. **Keys are `resource:action`** — e.g. `task:read`, `platform_user:update`. Normalize legacy formats at API boundaries only.
3. **Platform scope = `platform_*` prefix** + null tenant id on IAM rows.
4. **Effective = direct ∪ role (user + groups)** — owner/super-admin may bypass; DENY beats ALLOW.
5. **Exempt routes** (public login, webhooks, secondary portals) are cataloged but never assignable.
6. **Client matches server** — same grant-matching logic; load effective keys from an API, not hardcoded role names.

---

## Architecture (stack-neutral)

```
┌─────────────────────────────────────────────────────────┐
│  ROUTE MANIFEST (code/JSON)                             │
│  method, path, resource, action                         │
└────────────────────┬────────────────────────────────────┘
                     │ sync on startup
                     ▼
┌─────────────────────────────────────────────────────────┐
│  PERMISSION STORE (DB)                                  │
│  resources, permissions, roles, bindings, direct grants │
└────────────────────┬────────────────────────────────────┘
                     │
     ┌───────────────┼───────────────┐
     ▼               ▼               ▼
 Route guard    Access APIs     Client catalog
 (middleware)   (CRUD roles)    (matrix UI)
```

---

## Implementation Checklist

```
Manifest RBAC Progress:
- [ ] 1. Audit existing auth (sessions, JWT, roles, multi-tenant?)
- [ ] 2. Add persistence model (see reference.md — ORM-agnostic)
- [ ] 3. Create route manifest + sync job on app start
- [ ] 4. Add route-level permission requirement + global enforcement hook
- [ ] 5. Annotate all protected routes; list exempt routes in manifest
- [ ] 6. Seed default role templates
- [ ] 7. Expose catalog + role CRUD + direct user-grant APIs
- [ ] 8. Client: grant utils, load /access/me, permission matrix, UI gates
- [ ] 9. Audit: manifest ↔ route metadata ↔ DB ↔ catalog API
```

### Step 1 — Project discovery

| Question | Impact |
|----------|--------|
| Multi-tenant? | Add `tenant_id` / `company_id` on roles and bindings |
| Super-admin layer? | Add `platform_*` manifest + admin UI |
| Custom roles UI? | Full access CRUD; else seed-only |
| Row-level scoping? | IAM for module gates + separate resource ownership checks |
| Monolith vs API-only? | Matrix UI optional for API-only backends |

### Step 2 — Persistence (conceptual)

Entities (map to your ORM/migrations):

- `iam_resource` — resource name
- `iam_permission` — resource, action, effect (ALLOW|DENY)
- `iam_role` — optional tenant_id (null = platform role), name, is_system
- `iam_role_permission` — role ↔ permission
- `iam_role_binding` — role ↔ user or group
- `iam_direct_permission_binding` — permission ↔ user (override)
- `iam_group` + `iam_group_member` — optional, for scoped role assignment

See [reference.md](reference.md) for fields and relationships.

### Step 3 — Route manifest

Store as JSON, YAML, or code — any language:

```json
[
  { "method": "GET", "path": "/tasks", "resource": "task", "action": "read" },
  { "method": "POST", "path": "/tasks", "resource": "task", "action": "create" }
]
```

Required helpers (implement in project language):

- `permission_key(resource, action)` → `"task:read"`
- `parse_permission_key(key)` → `{ resource, action }`
- `unique_manifest_permissions()` → deduped list for sync

Optional sections: `platform_manifest`, `default_tenant_roles`, `default_platform_roles`, `exempt_routes`.

**Sync on startup:** idempotent upsert of resources and permissions. Do not silently delete stale permissions.

### Step 4 — Route enforcement

Every protected handler must declare its required grant. Pattern name varies by stack — see [adapters.md](adapters.md).

**Guard chain (conceptual):**

1. Authenticate → attach principal to request context
2. Optional: coarse role check (`admin`, `owner`)
3. Optional: block secondary identities (worker portal JWT) from staff routes
4. **Permission check** → load grants → `check_permission(principal, resource, action)`
5. Exempt/public routes skip steps 3–4

**check_permission logic:**

1. Owner / super-admin bypass (if applicable)
2. Resolve scope (`tenant` vs `platform`)
3. Collect direct + role + group grants
4. Map admin tenant resource → platform resource when paths overlap
5. Evaluate grants (wildcards, `manage`, DENY-first)

Implement `matches_grant()` once; reuse on server and client.

### Step 5 — Access APIs

REST shape (adapt paths to project conventions):

| Endpoint | Purpose |
|----------|---------|
| `GET /access/permissions` | Catalog: permissions + exempt routes |
| CRUD `/access/roles` | Tenant role templates |
| CRUD `/access/groups` | Optional scoped groups |
| `GET/PUT /access/users/:id/permissions` | Direct user grants |
| `GET /access/me` | Current user's effective permission keys |

Platform mirror under `/access/platform/*` when multi-scope. Validate on write:

- Tenant roles → reject `platform_*` keys
- Platform roles → require `platform_*` keys only

### Step 6 — Client (any UI framework)

Required behaviors (not specific components):

1. **Grant utilities** — `parsePermissionKey`, `matchesGrant`, `permissionGranted(perms, key)`
2. **Load effective keys** — fetch `GET /access/me` after login; cache in store/context
3. **Permission matrix** — table with columns: scope, resource, action, key; checkboxes in picker mode; search + scope tabs (All/Tenant/Platform)
4. **UI gates** — hide nav/buttons/routes when `!permissionGranted(effective, 'task:read')`
5. **Role editor** — bind selected keys; validate scope on save
6. **User grant editor** — show direct vs inherited; block edit for protected users

Matrix modes:

| Mode | Behavior |
|------|----------|
| Picker | Checkboxes, select all, selected count, assignable filter |
| Catalog (readonly) | No checkboxes; show mapped routes; public/portal exempt tab |

---

## Adding a New Protected Feature

1. Add manifest row(s)
2. Attach permission requirement to route handler
3. Add to default roles if needed by default
4. Gate client nav/page if applicable
5. Re-run sync / audit

**Never** enforce a permission that isn't in the manifest — role bindings skip unknown keys silently.

---

## Default Role Templates

```json
{
  "company-owner": { "permissions": "*", "is_system": true },
  "staff": { "permissions": ["task:read", "task:create", "auth:read"] }
}
```

- `*` = all keys in that scope's manifest
- Sync idempotently per tenant / platform on init
- System roles: lock permission edits

---

## Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Manifest drift | Audit: manifest ↔ route metadata ↔ DB |
| Platform keys on tenant roles | Validate at write boundary |
| Admin on shared paths | Resource alias map (tenant → platform) |
| Client/server mismatch | Share or duplicate identical `matches_grant` logic |
| Secondary auth on staff routes | Separate guard; exempt manifest for portal |
| Matrix "0 of 0 selected" | Assignable filter must match save scope |
| Framework-specific copy-paste | Use [adapters.md](adapters.md); adapt to project |

---

## Simplifications (MVP)

Single-tenant / no admin layer:

- One manifest, one role set, no `platform_*`
- Seed 2–3 roles; optional role UI
- Minimum: manifest + middleware + `GET /access/me`

---

## Verification

1. Every protected route ↔ manifest entry
2. Catalog API = full manifest + exempt routes
3. Role save/load round-trips keys
4. Direct + role grants merge correctly
5. Matrix checkboxes match assignable scope
6. Protected super-users cannot be stripped

---

## Agent Instructions

When applying this skill:

1. **Inspect the project first** — identify auth, routing, ORM, and UI patterns
2. **Implement concepts, not file names** — use project conventions for paths and naming
3. **One canonical manifest** — JSON/YAML/code in any language the project uses
4. **Reuse grant matching** — same algorithm server-side and client-side
5. **Consult [adapters.md](adapters.md)** for stack-specific wiring examples
6. **Reference implementation** (optional): SiteWise — patterns only, not a template to copy
