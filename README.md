# Manifest RBAC — Cursor Plugin

**Manifest RBAC** is a stack-agnostic pattern for fine-grained authorization:

- **Route manifest** as source of truth (method, path → `resource:action`)
- **Permission keys** like `task:read`, `platform_user:update`
- **Roles + direct user grants** with optional tenant vs platform scopes
- **Permission matrix UI** spec for role editors and API catalogs

Works with **any** backend (Express, FastAPI, Django, Rails, Go, NestJS, Laravel, Spring, .NET), **any** database/ORM, and **any** frontend.

---

## Install

### Option A — Cursor Plugin (recommended)

**Test locally:**

```bash
# Windows (PowerShell)
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.cursor\plugins\local\manifest-rbac"
Copy-Item -Recurse -Force "path\to\cursor-manifest-rbac\*" "$env:USERPROFILE\.cursor\plugins\local\manifest-rbac\"

# macOS / Linux
mkdir -p ~/.cursor/plugins/local/manifest-rbac
cp -r /path/to/cursor-manifest-rbac/* ~/.cursor/plugins/local/manifest-rbac/
```

Reload Cursor (**Developer: Reload Window**). Check **Customize** → skills.

**Publish:** submit this repo at [cursor.com/marketplace/publish](https://cursor.com/marketplace/publish) (open source, manual review). Optionally list on [cursor.directory](https://cursor.directory).

### Option B — Project skill (team via git)

Copy into your repo:

```
your-project/.cursor/skills/manifest-rbac/
```

Commit — everyone who clones gets the skill.

### Option C — Personal global skill

Copy `skills/manifest-rbac/` to:

| OS | Path |
|----|------|
| Windows | `%USERPROFILE%\.cursor\skills\manifest-rbac\` |
| macOS/Linux | `~/.cursor/skills/manifest-rbac/` |

---

## Usage

In Cursor Agent chat:

```
@manifest-rbac Add RBAC to this Express + PostgreSQL + React app
```

```
/manifest-rbac Implement route manifest, roles, and GET /access/me
```

The agent reads `SKILL.md`, detects your stack, and follows the implementation checklist.

---

## What's inside

```
cursor-manifest-rbac/
├── .cursor-plugin/plugin.json
├── skills/manifest-rbac/
│   ├── SKILL.md        # Main workflow + checklist
│   ├── reference.md    # Data model, APIs, algorithms
│   ├── adapters.md     # Express, FastAPI, Django, Rails, Go, etc.
│   └── examples.md     # Multi-stack examples
├── README.md
└── LICENSE
```

---

## Core idea

```
Route manifest  →  sync to DB  →  middleware/guard enforces resource:action
                      ↓
              roles + direct grants  →  GET /access/me  →  UI gates + matrix
```

---

## Team marketplace (Cursor Teams / Enterprise)

1. Push this repo to GitHub
2. **Dashboard → Plugins → Add Marketplace → Import from Repo**
3. Assign to org groups; set Default On or Required

---

## Reference implementation

[SiteWise](https://github.com/your-org/sitewise) (NestJS + Prisma + Svelte) — behavioral reference only. The skill tells the agent to port **logic**, not copy **files**.

---

## Contributing

PRs welcome. Keep `SKILL.md` under 500 lines; put deep detail in `reference.md` / `adapters.md`.

## License

MIT
