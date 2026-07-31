# Manifest RBAC — Reference

## Permission key

```
<resource>:<action>
```

Examples: `task:read`, `user:invite`, `role:manage`, `platform_user:read`, `platform_role:manage`.

Actions: `read`, `create`, `update`, `delete`, `manage`, `invite`, `upload`, + domain verbs.

---

## Wildcard matching (implement once, share server + client)

```
matchesGrant(gr, ga, resource, action):
  if (gr == '*' or gr == resource) and (ga == '*' or ga == action): return true
  if gr == resource and ga == 'manage': return true
  return false
```

Evaluation: any DENY on match → false; else true if any ALLOW matches.

---

## API contract

| Method | Path | Auth | Response |
|--------|------|------|----------|
| GET | `/access/me` | staff JWT | `string[]` permission keys |
| GET | `/access/permissions?scope=tenant` | `role:manage` | catalog + exempt routes |
| GET | `/access/roles` | `role:manage` | roles with permission keys |
| POST | `/access/roles` | `role:manage` | create role |
| GET | `/access/platform/roles` | `platform_role:manage` | platform roles |
| GET | `/access/platform/permissions` | `platform_role:manage` | platform catalog |
| PUT | `/access/users/:id/permissions` | `role:manage` | direct grant keys |

Catalog entry shape:

```json
{
  "resource": "task",
  "action": "read",
  "key": "task:read",
  "description": "List and view tasks",
  "routes": [{ "method": "GET", "path": "/tasks" }]
}
```

---

## Default role templates

**Tenant** (`companyId` set):

| Template | Permissions |
|----------|-------------|
| company-owner | `*` |
| site-supervisor | explicit ops list |
| payroll-manager | finance subset |
| custom | admin-defined |

**Platform** (`companyId = null`):

| Template | Permissions |
|----------|-------------|
| platform-super-admin | `*` (platform manifest) |
| platform-support | complaints + demo read |

System roles (`isSystem: true`) — protect from permission edits or deletion.

---

## Invite payload

```json
{
  "name": "…",
  "email": "…",
  "roleId": "role-supervisor",
  "roleGroupId": "optional",
  "siteIds": ["…"]
}
```

User response includes `iamRoleId`, `iamRoleName`.

---

## Site scope algorithm

```
explicitSiteScope(userId, companyId):
  ids = userSiteScope ∪ siteSupervisor ∪ group.siteIds ∪ roleGroup.siteIds
  if ids empty → company-wide site access
  else → only listed site ids
```

---

## Reference codebase (SiteWise)

Behavioral reference only — do not copy paths blindly:

| Area | Path |
|------|------|
| Manifest | `backend/src/iam/route-manifest.ts` |
| IAM service | `backend/src/iam/iam.service.ts` |
| Guard | `backend/src/common/guards/permissions.guard.ts` |
| Site scope | `backend/src/access/access.service.ts` |
| Frontend store | `frontend/src/lib/stores/access.ts` |
| Nav manifest | `frontend/src/lib/nav.ts` |
| Matrix | `frontend/src/lib/components/access/FortressPermissionMatrix.svelte` |

Repo: https://github.com/Byiringiro215/sitewise
