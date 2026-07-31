# Manifest RBAC — Stack Adapters

Map Manifest RBAC concepts to common frameworks. Pick the row that matches the project; adapt naming to local conventions.

---

## Route Enforcement

### Node — Express / Fastify

```javascript
function requirePermission(resource, action) {
  return async (req, res, next) => {
    const ok = await iam.checkPermission(req.user, resource, action);
    if (!ok) return res.status(403).json({ code: 'FORBIDDEN' });
    next();
  };
}

router.get('/tasks', requirePermission('task', 'read'), listTasks);
router.post('/tasks', requirePermission('task', 'create'), createTask);
```

Global: register manifest sync in `app.listen` callback or startup hook.

### Python — FastAPI

```python
def require_permission(resource: str, action: str):
    async def dep(user: User = Depends(get_current_user)):
        if not await iam.check_permission(user, resource, action):
            raise HTTPException(403, "FORBIDDEN")
        return user
    return dep

@router.get("/tasks")
async def list_tasks(user: User = Depends(require_permission("task", "read"))):
    ...
```

Startup: `@app.on_event("startup")` → `await iam.sync_from_manifest()`.

### Python — Django REST Framework

```python
class TaskPermission(BasePermission):
    resource, action = "task", "read"
    def has_permission(self, request, view):
        return iam.check_permission(request.user, self.resource, self.action)

class TaskViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated, TaskPermission]
```

### Ruby — Rails

```ruby
module RequiresPermission
  def require_permission(resource, action)
    before_action do
      head :forbidden unless IamService.check_permission(current_user, resource, action)
    end
  end
end

class TasksController < ApplicationController
  require_permission "task", "read"
  def index; end
end
```

Manifest: `config/iam_manifest.yml` loaded in initializer.

### Go — chi / gin / echo

```go
func RequirePermission(iam *IAM, resource, action string) func(http.Handler) http.Handler {
  return func(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
      user := UserFromContext(r.Context())
      if !iam.CheckPermission(user, resource, action) {
        http.Error(w, "FORBIDDEN", 403)
        return
      }
      next.ServeHTTP(w, r)
    })
  }
}
```

### PHP — Laravel

```php
Route::get('/tasks', [TaskController::class, 'index'])
    ->middleware(['auth', 'permission:task,read']);
```

### Java — Spring Boot

```java
@PreAuthorize("@iam.checkPermission(authentication, 'task', 'read')")
@GetMapping("/tasks")
public List<Task> list() { ... }
```

### .NET — ASP.NET Core

```csharp
[RequirePermission("task", "read")]
[HttpGet]
public IActionResult List() => ...
```

---

## Persistence Adapters

| ORM / tool | Notes |
|------------|-------|
| Prisma / Drizzle / TypeORM | TS/JS migrations; see reference ER diagram |
| SQLAlchemy / Django ORM | Models mirror same tables; sync via management command |
| ActiveRecord (Rails) | Migrations + `rails iam:sync` task |
| GORM / sqlx (Go) | Migrations with goose/flyway; sync in `main()` |
| Eloquent (Laravel) | Models + `php artisan iam:sync` |
| Entity Framework | DbContext + startup hosted service for sync |
| Raw SQL | Flyway/Liquibase migrations |

Core tables: `iam_resources`, `iam_permissions`, `iam_roles`, `iam_role_permissions`, `iam_role_bindings`, `iam_direct_permission_bindings`, `iam_groups`, `iam_group_members`.

---

## Manifest Storage

| Format | When to use |
|--------|-------------|
| JSON/YAML file | Polyglot, easy audit, CI validation |
| Code registry | Type-safe, co-located with routes |
| OpenAPI extension | `x-permission: task:read` per operation |
| Route annotations | Scan decorators at build time |

---

## Client UI Adapters

**Permission Matrix** is a behavioral spec — implement in any UI:

| Framework | Gate pattern |
|-----------|--------------|
| React | `<PermissionGate permission="task:read">` |
| Vue 3 | `v-if="can('task:read')"` composable |
| Svelte | `{#if $can('task:read')}` |
| Angular | `*appPermission="'task:read'"` directive |
| Blade / HTMX | Server-rendered effective keys |

Matrix columns: Select, Scope, Resource, Action, Key, Routes (catalog mode).

---

## Auth Integration

| Auth style | Notes |
|------------|-------|
| JWT | Reload grants from DB per request or cache with TTL |
| Session | Load user + bindings in middleware |
| OAuth/OIDC | Sync IdP groups → role bindings |
| Secondary portal | Separate JWT; exempt routes in manifest |

Jobs, GraphQL, WebSockets — call `check_permission()` directly.

---

## Minimal File Layout (adapt paths)

```
iam/manifest.json
iam/service.*
access/routes.*
middleware/
migrations/
client/permissions.*
client/PermissionMatrix.*
```

Monorepo: share `grant_utils` package between API and web app.

---

## Reference Implementation (optional)

SiteWise (NestJS + Prisma + Svelte) — behavioral reference only. Port **logic**, not **files**.
