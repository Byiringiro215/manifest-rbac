# Manifest RBAC — Backend

Stack-agnostic backend guide. NestJS names shown as reference; map via [adapters.md](adapters.md).

---

## 1. Data model

Minimum tables (names flexible):

| Entity | Purpose |
|--------|---------|
| `IamPermission` | `resource`, `action`, `effect` (ALLOW/DENY) |
| `IamRole` | `companyId` (null = platform), `name`, `isSystem`, `legacyId` |
| `IamRolePermission` | role ↔ permission |
| `IamRoleBinding` | role ↔ user or group (`subjectType`, `subjectId`) |
| `IamGroup` | optional site-scoped group + `siteIds[]` |
| `IamGroupMember` | user ↔ group |
| `IamDirectPermissionBinding` | user ↔ permission (overrides) |

Full ER and API shapes: [reference.md](reference.md).

---

## 2. Route manifest

Single registry of protected routes:

```typescript
export interface RoutePermissionMeta {
  method: string;
  path: string;      // normalized, no API prefix
  resource: string;
  action: string;
  description?: string;
}

export const ROUTE_MANIFEST: RoutePermissionMeta[] = [
  { method: 'GET', path: '/tasks', resource: 'task', action: 'read' },
  { method: 'POST', path: '/tasks', resource: 'task', action: 'create' },
  // ...
];

export const EXEMPT_ROUTE_MANIFEST = [
  { method: 'POST', path: '/auth/login', category: 'public' },
  // worker portal, webhooks — catalog only, not assignable
];
```

Separate `PLATFORM_ROUTE_MANIFEST` for `platform_*` resources.

**Rule:** Adding a route = add manifest row + `@RequirePermission` on handler + run sync.

---

## 3. Sync permissions on startup

Idempotent job (`onModuleInit`):

1. Upsert `IamResource` / `IamPermission` from manifest (ALLOW only for assignable).
2. `ensurePlatformDefaultRoles()` — templates with `companyId = null`.
3. `ensureCompanyDefaultRoles(companyId)` per tenant — bind permission keys to roles.
4. `ensureOwnerBindingsForCompany` / `ensureSupervisorBindingsForCompany` — bind legacy `User.role` enum users to system IAM roles (migration aid only; **not** used at check time).

Default tenant role `company-owner`: `permissions: '*'`.  
Default `site-supervisor`: explicit list including `attendance:read`, `notification:read`, etc.

---

## 4. RequirePermission + global guard

```typescript
export const RequirePermission = (resource: string, action: string) =>
  SetMetadata(PERMISSIONS_KEY, { resource, action });

@Injectable()
export class PermissionsGuard implements CanActivate {
  async canActivate(ctx: ExecutionContext) {
    const required = reflector.get(PERMISSIONS_KEY, handler);
    if (!required) return true;
    const user = req.user;
    if (!user) throw UnauthorizedException();
    for (const { resource, action } of list) {
      if (!(await iam.checkPermission(user, resource, action))) {
        throw ForbiddenException();
      }
    }
    return true;
  }
}
```

Register guard **globally** after JWT auth. Remove old `@Roles()` / `RolesGuard`.

Public routes: `@Public()` decorator bypasses permission metadata.

---

## 5. IamService.checkPermission

```typescript
async checkPermission(user, resource, action): Promise<boolean> {
  // Platform user or platform_* resource → evaluate platform grants first
  const platformResource = TENANT_TO_PLATFORM_RESOURCE[resource] ?? resource;
  if (user.companyId === null || resource.startsWith('platform_')) {
    if (evaluateGrants(platformGrants, platformResource, action)) return true;
    if (user.companyId === null) return false;
  }
  // Tenant grants
  return evaluateGrants(tenantGrants, resource, action);
}
```

**TENANT_TO_PLATFORM_RESOURCE** (extend as needed):

```typescript
{
  dashboard: 'platform_overview',
  user: 'platform_user',
  site: 'platform_site',
  role: 'platform_role',
  attendance: 'platform_overview',  // cross-tenant activity feeds
  workforce: 'platform_site',
  // ...
}
```

**matchesGrant** (shared with frontend):

- `*` resource or action matches anything
- `manage` action on same resource matches all actions on that resource
- DENY grant beats ALLOW

**No bypass** for owner/admin enum — only grants count.

---

## 6. Effective permissions API

```http
GET /access/me
Authorization: Bearer …
→ 200 ["task:read", "user:invite", …]
```

Implementation: union ALLOW grants from direct bindings + role bindings (user + groups).  
Platform users: collect platform scope only. Tenant users: tenant scope only.

Do **not** require `role:manage` for `/access/me`.

---

## 7. Access catalog APIs

| Endpoint | Permission | Notes |
|----------|------------|-------|
| `GET /access/permissions` | `role:manage` | Tenant catalog |
| `GET /access/roles` | `role:manage` | Requires `companyId` |
| `GET /access/groups` | `role:manage` | |
| `GET /access/platform/roles` | `platform_role:manage` | |
| `GET /access/platform/permissions` | `platform_role:manage` | |
| `PUT /access/users/:id/permissions` | `role:manage` | Direct grants |

List roles calls `requireCompanyId(user)` — platform-only admins cannot hit tenant role list (guard may pass via mapping but service rejects null company).

---

## 8. Site scope (AccessService)

Separate from permission keys:

```typescript
async assertSiteAccess(user, siteId) {
  if (user.companyId === null) return site; // platform
  if (user.companyId !== site.companyId) throw FORBIDDEN;
  const scoped = await explicitSiteScope(user.id, user.companyId);
  if (scoped.length === 0 || scoped.includes(siteId)) return site;
  throw FORBIDDEN;
}
```

**explicitSiteScope** merges:

- `UserSiteScope` rows
- `SiteSupervisor` assignments
- IAM group `siteIds` (memberships + legacy `roleGroupId`)

Empty scope = all sites in company.

Call `assertSiteAccess` in services after permission guard passes.

---

## 9. User invite

Replace role enum with IAM role:

```typescript
// DTO
roleId: string;  // IAM role id or legacyId e.g. role-supervisor
roleGroupId?: string;
siteIds?: string[];

// Flow
const iamRole = await findRoleByLegacyOrId(companyId, dto.roleId);
await createUser({ ... });
await iam.bindUserToRole(user.id, companyId, iamRole.id);
```

Map response with `iamRoleId`, `iamRoleName` for UI labels — not `User.role` for display authority.

---

## 10. Service-layer rules

- Use `assertPermission(user, resource, action)` or rely on guard for HTTP.
- Never `if (user.role === UserRole.owner)` for authorization.
- Payouts, listings, deletes: use grants + site scope only.
- Worker JWT (`kind: 'worker'`) → 403 on staff IAM routes.

---

## 11. Verification

```bash
# Every controller handler has RequirePermission or Public
# No RolesGuard / @Roles in codebase
node scripts/audit-iam-coverage.mjs   # optional: manifest vs decorators
```

See [audit-checklist.md](audit-checklist.md).
