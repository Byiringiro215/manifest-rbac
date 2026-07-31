# Manifest RBAC — Frontend

Grant-driven UI: one shell, permissions decide visibility. No `if (user.role === 'owner')` for nav, routes, or actions.

---

## 1. File layout (reference: SvelteKit)

```
lib/access/
  permissions.ts          # Permission union, PERMISSION_META, matchesGrant, permissionGranted
  platform-permissions.ts # platform_* keys + admin nav map
  catalog.ts              # normalizeIamCatalog API response
  iam.ts                  # inviteableRoles, userIamRoleLabel
  scope.ts                # hasPlatformScope(user) → companyId === null
lib/stores/
  user.ts                 # currentUser (break circular imports)
  access.ts               # loadUserPermissions, can(), userPermissions store
  auth.ts                 # login/logout; re-exports user
lib/nav.ts                # NAV catalogs, ROUTE_PERMISSIONS, permissionSpecForPath
components/access/
  FortressPermissionMatrix.svelte
  UserPermissionEditor.svelte
components/layout/
  RoutePermissionGate.svelte
  PermissionGate.svelte
```

Adapt paths to React/Vue/Angular; keep **behaviors**.

---

## 2. Grant utilities (must mirror backend)

```typescript
export function matchesGrant(gr, ga, resource, action): boolean {
  if (gr === '*' || gr === resource) {
    if (ga === '*' || ga === action) return true;
  }
  if (gr === resource && ga === 'manage') return true;
  return false;
}

export function permissionGranted(perms, key): boolean {
  const { resource, action } = parsePermissionKey(normalizePermissionKey(key));
  for (const k of perms) {
    const { resource: gr, action: ga } = parsePermissionKey(k);
    if (matchesGrant(gr, ga, resource, action)) return true;
  }
  return false;
}
```

Legacy map: `'tasks.view' → 'task:read'`, `'roles.manage' → 'role:manage'`.

---

## 3. Access store

```typescript
// Load once per user id after login
async function loadUserPermissions(force = false) {
  const user = get(currentUser);
  if (!user) { resetAccessCatalog(); return; }
  if (!force && permissionsUserId === user.id && permissionsLoaded) return;
  try {
    const perms = await api.getMyPermissions();
    resolvedPermissions.set(perms.map(normalizePermissionKey));
  } catch {
    resolvedPermissions.set([]); // fail-closed
  }
  permissionsUserId = user.id;
}

export function can(key) {
  return permissionGranted(get(userPermissions), key);
}

export function resetAccessCatalog() {
  resolvedPermissions.set(null);
  permissionsUserId = null;
  // clear role/group catalog cache
}
```

**Never** use `defaultPermissionsForRole(user.role)` for effective permissions.

Call `resetAccessCatalog()` from `clearSession()` / logout.

---

## 4. App layout bootstrap

```svelte
$effect(() => {
  const userId = user?.id;
  if (userId) void loadUserPermissions().then(() => loadAccessCatalog());
});
```

`loadAccessCatalog` — **only** when tenant role manager:

```typescript
if (!user?.companyId || !can('role:manage')) return;
await api.listAccessRoles();
await api.listRoleGroups();
```

Do not load tenant catalog for platform-only admins (`platform_role:manage` alone).

---

## 5. Nav manifest

Single catalog per scope; filter with `permissionGranted`:

```typescript
const TENANT_NAV = [
  { href: '/site/tasks', permission: 'task:read' },
  { href: '/users', permission: 'user:read' },
  { href: '/access', permission: 'role:manage' },
];

export function navForPermissions(perms) {
  return catalog
    .map(section => ({
      ...section,
      items: section.items.filter(i => permissionGranted(perms, i.permission)),
    }))
    .filter(s => s.items.length);
}
```

Choose admin vs tenant catalog by grants (`platform_overview:read` without `site:read` → admin nav).

**Home route:** first matching grant from `HOME_GRANTS` (`dashboard:read`, `site:read`, `platform_overview:read`) → typically `/dashboard`.

---

## 6. Route permission gate

Central path manifest (longest prefix match):

```typescript
export function permissionSpecForPath(pathname): { permissions, mode } | undefined {
  if (pathname === '/dashboard') return { permissions: HOME_GRANTS, mode: 'any' };
  if (pathname.startsWith('/site/')) {
    return { permissions: ['site:read', routePerm], mode: 'all' };
  }
  // ...
}
```

Layout wrapper:

```svelte
<!-- RoutePermissionGate.svelte -->
const spec = $derived(permissionSpecForPath($page.url.pathname));
{#if spec}
  <PermissionGate permissions={spec.permissions} mode={spec.mode}>
    {@render children()}
  </PermissionGate>
{:else}
  {@render children()}
{/if}
```

`PermissionGate`: if user logged in and lacks grants → redirect `homePath(user)` (permission-aware, not role enum).

---

## 7. Permission-gated API calls

Before fetch in components:

```typescript
if (!can('notification:read')) return;
if (!can('attendance:read')) return;
```

Hide UI affordances too:

```svelte
{#if canViewNotifications}
  <NotificationBell />
{/if}
```

Prevents 403 noise; backend still enforces.

---

## 8. FortressPermissionMatrix

Props:

| Prop | Role editor | API catalog (readonly) |
|------|-------------|------------------------|
| `readonly` | false | true |
| `assignableFilter` | `tenant` or `platform` | — |
| `scope` | `tenant` or `platform` | `all` |
| `catalogApi` | `tenant` / `platform` | same |
| `showScopeFilter` | false when assignableFilter set | true |
| `showRoutes` | optional | true |

**Platform role modal:**

```svelte
<FortressPermissionMatrix
  bind:selected={rolePerms}
  scope="platform"
  catalogApi="platform"
  assignableFilter="platform"
/>
```

No All/Tenant tabs — platform roles only accept `platform_*` keys.

**Tenant role modal:**

```svelte
<FortressPermissionMatrix
  bind:selected={rolePerms}
  scope="tenant"
  catalogApi="tenant"
  assignableFilter="tenant"
/>
```

API catalog tab: `readonly`, `showRoutes`, `showExempt` — no checkboxes.

---

## 9. User invite UI

- Dropdown: IAM roles from `listAccessRoles()` (when `can('role:manage')`).
- Submit `roleId`, not `role: 'supervisor'`.
- Display: `userIamRoleLabel(user)` → `iamRoleName` fallback to legacy label.
- Filter users by `iamRoleId`, not enum role.

---

## 10. Labels and i18n

Show **IAM role name** in Sidebar, UserMenu, tables — not hardcoded owner/supervisor/admin.

Remove `RoleGate.svelte` and all `{#if user.role === …}` for access control.

Keep `User.role` on JWT only for migration/seed if backend still stores it.

---

## 11. Unified dashboard pattern

Prefer **one `/dashboard`** with panels gated by grants:

```svelte
{#if permissionGranted($userPermissions, 'platform_overview:read')}
  <PlatformOverviewPanel />
{:else if permissionGranted($userPermissions, 'site:read')}
  <SiteHomePanel />
{/if}
```

Not separate `/admin` vs `/site` home by enum.

---

## 12. Frontend audit

Grep and eliminate:

```
RoleGate|role === 'owner'|role === 'admin'|defaultPermissionsForRole
homePathForRole(.*role
```

Replace with `can()`, `permissionGranted`, `homePathForPermissions`.

See [audit-checklist.md](audit-checklist.md).
