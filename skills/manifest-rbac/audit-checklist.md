# Manifest RBAC — Audit checklist

Run when migrating to grant-only IAM or before release.

---

## Backend grep (should be **zero** hits in auth paths)

```bash
# Legacy role guards
rg "@Roles\(|RolesGuard|UserRole\.(owner|admin|supervisor)" backend/src --glob "*.ts"

# Role bypass in services
rg "user\.role\s*===|role\s*===\s*UserRole" backend/src --glob "*.service.ts"

# Owner/admin skip in checkPermission
rg "isOwner|isAdmin|bypass" backend/src/iam
```

**Allowed:** `User.role` in seed, JWT payload, invite legacy mapping, `ensureOwnerBindingsForCompany` migration.

Every controller handler must have `@RequirePermission` or `@Public()`.

---

## Frontend grep (should be **zero** for access control)

```bash
rg "RoleGate|role === 'owner'|role === 'admin'|role === 'supervisor'" frontend/src
rg "defaultPermissionsForRole|homePathForRole" frontend/src
rg "RolesGuard|allow=\{.*owner" frontend/src
```

**Replace with:** `can()`, `permissionGranted()`, `userIamRoleLabel()`, `homePathForPermissions()`.

---

## API call audit

Every `api.*` in layout/header/onMount must either:

- Be gated with `can('resource:action')`, or
- Hit an endpoint the user always has (e.g. `/access/me`, `/users/me`)

Common leaks:

- `/access/roles` without `role:manage`
- `/notifications` without `notification:read`
- Tenant catalog for platform-only admin

---

## Matrix UI audit

| Context | assignableFilter | scope | showScopeFilter |
|---------|------------------|-------|-----------------|
| Tenant role modal | tenant | tenant | false |
| Platform role modal | platform | platform | false |
| API catalog | — | all/platform | true, readonly |

---

## Manual test matrix

| Account | Expect |
|---------|--------|
| Platform admin | Admin nav; no 403 on `/admin/activity` scans; no tenant `/access/roles` from layout |
| Tenant owner | Full tenant nav; `/access/me` has `role:manage` |
| Supervisor | Site ops nav; `notification:read`; no `/access/roles` |
| Custom role | Only granted nav items; 403 on ungranted API |

---

## Definition of done

- [ ] Manifest row for every protected route
- [ ] Global permission guard active
- [ ] No `@Roles()` in codebase
- [ ] `GET /access/me` drives client
- [ ] Fail-closed client on `/access/me` failure
- [ ] Nav + route gates use grant manifest
- [ ] Invite uses `roleId`
- [ ] IAM role name in UI labels
- [ ] Logout clears permission cache
- [ ] Site scope enforced in services
