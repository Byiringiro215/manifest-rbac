# Manifest RBAC — Stack adapters

Map Fortress concepts to your framework. Same behavior everywhere.

| Concept | NestJS | Express | FastAPI | Django | Rails | Go (chi) |
|---------|--------|---------|---------|--------|-------|----------|
| Manifest | `route-manifest.ts` | `routes.manifest.json` | Python list / YAML | `permissions_manifest.py` | YAML initializer | Go slice |
| Require perm | `@RequirePermission` + Reflector | `requirePerm('task','read')` middleware factory | `@requires("task:read")` dep | `@permission_required` | `before_action` + policy | middleware |
| Global guard | `APP_GUARD` | `app.use(authz)` | dependency on router | middleware stack | `ApplicationController` | wrapper |
| IAM service | `@Injectable()` | module export | class + Depends | service class | concern | package |
| Sync on start | `onModuleInit` | `app.listen` callback | `@app.on_event("startup")` | `AppConfig.ready` | `after_initialize` | `main()` |
| /access/me | Controller | router.get | APIRouter | View | controller#me | handler |

## Frontend adapters

| Concept | SvelteKit | React | Vue 3 |
|---------|-----------|-------|-------|
| Grant store | writable + derived | Context + useReducer | Pinia store |
| Route gate | `+layout.svelte` + `$page.url` | `<Outlet />` wrapper + useLocation | `router.beforeEach` |
| can() | `get(userPermissions)` | `usePermissions()` hook | `store.can()` |
| Matrix | Svelte component | DataGrid + checkboxes | Vuetify table |

## ORM

Prisma, Drizzle, TypeORM, SQLAlchemy, Eloquent, GORM — same tables from [reference.md](reference.md). Use upsert for manifest sync.

## JWT payload

Include `sub`, `companyId` (null = platform). **Do not** put permission arrays in JWT — always resolve server-side for `/access/me`.
