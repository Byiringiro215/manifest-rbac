---
name: manifest-rbac
description: >-
  Implements Fortress-style manifest RBAC end-to-end: backend route manifest,
  resource:action grants, IAM roles/bindings, global permission guard, GET
  /access/me, and frontend grant store with nav/route gates and permission matrix
  UI. Stack-agnostic with NestJS+Svelte reference paths. Use when adding IAM,
  RBAC, Fortress authorization, @RequirePermission, permission matrix, tenant vs
  platform scopes, or migrating off role-enum checks.
---

# Manifest RBAC (Fortress-style)

Grant-based authorization: **manifest → DB → guard → `/access/me` → UI gates**.  
No login-role labels for access decisions. Custom IAM role names drive behavior.

**Read next by task:**
- Backend: [backend.md](backend.md)
- Frontend: [frontend.md](frontend.md)
- Data model & APIs: [reference.md](reference.md)
- Stack wiring: [adapters.md](adapters.md)
- Scenarios: [examples.md](examples.md)
- Audit / migration: [audit-checklist.md](audit-checklist.md)

---

## Non-negotiable rules

1. **Manifest is source of truth** — every protected route has `{ method, path, resource, action }`.
2. **Keys are `resource:action`** — normalize legacy dot keys at boundaries only (`tasks.view` → `task:read`).
3. **Grant-only enforcement** — no `if (user.role === 'admin')` bypass in guards or services for authorization.
4. **Effective permissions = API** — client loads `GET /access/me`; fail-closed on error (empty grants).
5. **Platform scope** — `platform_*` resources + `companyId = null` on IAM rows; map tenant checks via `TENANT_TO_PLATFORM_RESOURCE`.
6. **Site scope is separate** — data access filtered by site assignments; do not infer from role name.
7. **Exempt routes** — login, webhooks, worker portal cataloged but not assignable in matrix.
8. **Client matches server** — same `matchesGrant` algorithm in backend IAM service and frontend utils.

---

## Detect stack first

| Concept | Map to project |
|---------|----------------|
| Route manifest | TS/JSON registry, OpenAPI extension, or codegen |
| `@RequirePermission` | Decorator, middleware, policy, attribute |
| Global guard | APP_GUARD, router.use, before_action |
| IAM service | Domain service module |
| `/access/me` | Auth/access controller |
| Grant store | Zustand, Pinia, Svelte store, React context |
| Route gate | Layout wrapper reading path manifest |
| Matrix UI | Table with checkboxes + readonly catalog tab |

**Port behavior, not files.** Reference stack: NestJS + Prisma + SvelteKit (SiteWise).

---

## Implementation order

```
Progress:
- [ ] 1. Persistence (IamPermission, IamRole, bindings, groups) — reference.md
- [ ] 2. Route manifest + syncPermissionsFromManifest on startup
- [ ] 3. Global permission guard + RequirePermission on every protected handler
- [ ] 4. IamService: collect grants, checkPermission, getEffectivePermissionKeys
- [ ] 5. Default role templates per tenant + platform; bind owners/supervisors on init
- [ ] 6. Access APIs: catalog, roles, groups, /access/me, user direct grants
- [ ] 7. Site scope service (assertSiteAccess, listAccessibleSiteIds)
- [ ] 8. Frontend: permissions.ts, access store, loadUserPermissions, can()
- [ ] 9. Frontend: nav manifest + RoutePermissionGate + permission-gated API calls
- [ ] 10. Frontend: FortressPermissionMatrix + role editor + invite by roleId
- [ ] 11. Audit: run audit-checklist.md grep patterns; remove role-enum UI gates
```

Details for steps 1–7: [backend.md](backend.md). Steps 8–10: [frontend.md](frontend.md).

---

## Quick backend snippet (pattern)

```typescript
// Route
@Get('tasks')
@RequirePermission('task', 'read')
list(@Req() req: { user: RequestUser }) { ... }

// Guard (global): read @RequirePermission metadata → iam.checkPermission(user, resource, action)
// Deny → 403. No role enum checks.
```

---

## Quick frontend snippet (pattern)

```typescript
// After login
await loadUserPermissions(); // GET /access/me → store

// Nav item
{ permission: 'task:read', href: '/tasks' }

// Before API call
if (!can('task:read')) return;

// Page gate (layout)
permissionSpecForPath(pathname) → PermissionGate
```

---

## Tenant vs platform

| | Tenant | Platform |
|---|--------|----------|
| IAM `companyId` | tenant uuid | `null` |
| Permission prefix | `task`, `user`, `site`… | `platform_user`, `platform_role`… |
| Admin UI | `/access` | `/admin/access` |
| Role editor matrix | `assignableFilter: tenant` | `assignableFilter: platform` only |
| Cross-tenant read | — | Map e.g. `attendance` → `platform_overview` for activity feeds |

---

## Common mistakes (avoid)

| Mistake | Fix |
|---------|-----|
| `can('platform_role:manage')` loads tenant `/access/roles` | Only load tenant catalog when `can('role:manage') && user.companyId` |
| Role enum on invite | Use `roleId` (IAM role id/legacyId) |
| `defaultPermissionsForRole(owner)` on client | Always `/access/me`; empty on failure |
| Notification bell fetches for all users | Gate with `can('notification:read')` |
| Platform admin sees tenant matrix tabs | `scope="platform"`, hide All/Tenant tabs when `assignableFilter !== 'all'` |
| Stale permissions after logout | `resetAccessCatalog()` in clearSession |
| `@Roles()` still on controllers | Remove; use `@RequirePermission` only |

---

## Invoke

```
@manifest-rbac Implement full Fortress RBAC on this project (backend + frontend)
```

```
@manifest-rbac Audit this repo for role-enum leaks and fix to grant-only IAM
```

When implementing, read **backend.md** and **frontend.md** in full before editing code.
