# Manifest RBAC — Fortress IAM for Cursor

Complete **backend + frontend** agent skill for manifest-driven, grant-only RBAC (Fortress-style).

- Route manifest → `resource:action` permissions  
- IAM roles, groups, direct user grants  
- Tenant vs platform scopes  
- Global permission guard (no role-enum bypass)  
- Client: `/access/me`, nav/route gates, permission matrix UI  

Works with **any stack** — NestJS + SvelteKit reference in [reference.md](skills/manifest-rbac/reference.md).

---

## Install

### Personal skill (all projects)

```powershell
# Windows
Copy-Item -Recurse -Force "skills\manifest-rbac" "$env:USERPROFILE\.cursor\skills\manifest-rbac"
```

```bash
# macOS / Linux
cp -r skills/manifest-rbac ~/.cursor/skills/manifest-rbac
```

### Project skill (team)

```
your-repo/.cursor/skills/manifest-rbac/
```

Copy the `skills/manifest-rbac/` folder from this repo.

### Cursor plugin (local test)

```powershell
Copy-Item -Recurse -Force "." "$env:USERPROFILE\.cursor\plugins\local\manifest-rbac"
```

Reload Cursor → **Developer: Reload Window**.

---

## Usage

```
@manifest-rbac Implement Fortress RBAC on this project (backend and frontend)
```

```
@manifest-rbac Audit and remove all role-enum authorization leaks
```

```
@manifest-rbac Add invoice module with manifest permissions and UI gates
```

---

## What's inside

```
skills/manifest-rbac/
├── SKILL.md           # Start here — rules, checklist, invoke
├── backend.md         # Manifest, guard, IAM, site scope, invites
├── frontend.md        # Store, nav, gates, matrix, API gating
├── reference.md       # Keys, APIs, data model, SiteWise paths
├── audit-checklist.md # Grep patterns + definition of done
├── examples.md        # Migration & greenfield scenarios
└── adapters.md        # Express, FastAPI, Django, React, Vue…
```

---

## Core flow

```
ROUTE_MANIFEST → sync DB → PermissionsGuard → GET /access/me → can() → UI
```

---

## Publish

Submit at [cursor.com/marketplace/publish](https://cursor.com/marketplace/publish) with this repo URL.

---

## Reference app

[SiteWise](https://github.com/Byiringiro215/sitewise) — production Fortress RBAC implementation.

---

## License

MIT
