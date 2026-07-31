# Manifest RBAC — Examples

## A. Greenfield Express + React

1. Add Prisma models from [reference.md](reference.md).
2. `route-manifest.json` + sync script on `app.listen`.
3. Middleware: `requirePermission('task','read')` reading JWT user.
4. `GET /access/me` → `iam.getEffectiveKeys(user)`.
5. React: `PermissionProvider` fetches `/access/me` on login.
6. `<Can permission="task:read"><TasksLink /></Can>`.
7. Matrix page: checkbox table bound to `POST /access/roles`.

---

## B. Migrate NestJS app off `@Roles()`

1. Generate manifest from existing routes (audit script).
2. Add `PermissionsGuard` globally; swap `@Roles()` → `@RequirePermission`.
3. Seed IAM roles matching old enums; bind existing users on startup.
4. Deploy backend first (guards enforce grants).
5. Frontend: add access store + replace role checks (see [frontend.md](frontend.md)).
6. Run [audit-checklist.md](audit-checklist.md).

---

## C. Add new module "invoices"

**Backend**

```typescript
// manifest
{ method: 'GET', path: '/invoices', resource: 'invoice', action: 'read' },
{ method: 'POST', path: '/invoices', resource: 'invoice', action: 'create' },

// controller
@RequirePermission('invoice', 'read')
@Get()
list() { ... }
```

Add `invoice:read`, `invoice:create` to supervisor template or custom roles.

**Frontend**

```typescript
// nav.ts
'/invoices': 'invoice:read',

// page
permissionSpecForPath auto-gates via ROUTE_PERMISSIONS
```

---

## D. Platform admin activity feed (cross-tenant)

Backend: map tenant permission to platform grant:

```typescript
TENANT_TO_PLATFORM_RESOURCE.attendance = 'platform_overview';
```

Platform admin passes guard for `GET /fingerprint/scans` via `platform_overview:read`.

Frontend: `/admin/activity` gated by `platform_overview:read` only.

---

## E. Tenant invite with IAM role

```typescript
await api.inviteUser({
  name, email,
  roleId: selectedRole.id,  // not role: 'supervisor'
  siteIds: [...],
});
```

UI loads roles only when `can('role:manage')`.

---

## F. Readonly API catalog page

```svelte
<FortressPermissionMatrix
  readonly
  scope="all"
  catalogApi="platform"
  showScopeFilter
  showRoutes
  showExempt
/>
```

No checkboxes — reference for admins only.

---

## G. Mock / offline frontend

Mock adapter implements `getMyPermissions(userId)` from role bindings table — same keys as production.  
Never hardcode `if (user.role === 'owner') return ALL_PERMISSIONS`.
