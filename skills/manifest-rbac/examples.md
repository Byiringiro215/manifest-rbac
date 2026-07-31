# Manifest RBAC — Examples (multi-stack)

Same feature, different stacks. All follow the same manifest and grant logic.

---

## Example 1: Add `stock` module (any backend)

**manifest.json**

```json
{ "method": "GET", "path": "/stock/items", "resource": "stock", "action": "read" },
{ "method": "POST", "path": "/stock/items", "resource": "stock", "action": "manage" }
```

**Express:** `router.get('/stock/items', requirePermission('stock', 'read'), listItems);`

**FastAPI:** `Depends(require_permission("stock", "read"))`

**Django:** `HasManifestPermission("stock", "read")`

**Rails:** `require_permission "stock", "read"`

**Go:** `p.Require("stock", "read", listItems)`

Add `stock:read` to default staff role. Gate client nav with `can('stock:read')`.

---

## Example 2: React permission gate

```tsx
export const useCan = (key: string) =>
  permissionGranted(useContext(PermissionsContext), key);

function ApproveButton() {
  if (!useCan('wage:update')) return null;
  return <button>Approve</button>;
}
```

Load keys after login: `fetch('/access/me')`.

---

## Example 3: Permission matrix (React — behavioral)

Table with scope tabs, search, checkboxes on assignable rows, footer `{selected} of {assignable} grants selected`. See [reference.md](reference.md) for full spec.

---

## Example 4: Direct user grant API

```http
PUT /access/users/usr_123/permissions
{ "permissions": ["wage:update", "report:read"] }
```

Response includes `direct`, `inherited`, `effective`, `read_only`.

---

## Example 5: Django sync command

```python
# python manage.py iam_sync
for perm in unique_permissions(manifest):
    IamPermission.objects.update_or_create(
        resource=perm["resource"], action=perm["action"],
        defaults={"effect": "ALLOW"},
    )
```

---

## Example 6: MVP (minimal, any stack)

```
iam/manifest.json
iam/service.*
auth/middleware.*
access/routes.*
client/permissions.*
```

Skip platform scope, groups, matrix UI. Seed one admin role with `*`.

---

## Example 7: Agent prompt

```
@manifest-rbac

This is an Express + PostgreSQL + React project. Implement Manifest RBAC:
1. Detect existing JWT auth and adapt the guard chain
2. Add iam_* migrations
3. Create config/iam_manifest.json from existing routes
4. Add requirePermission middleware
5. GET /access/me and GET /access/permissions
6. React PermissionMatrix + useCan hook
7. Use this stack's patterns only — do not introduce other frameworks
```

Follow SKILL.md and [adapters.md](adapters.md).
