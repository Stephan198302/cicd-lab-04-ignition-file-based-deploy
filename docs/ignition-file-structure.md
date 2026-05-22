# Ignition 8.3 file structure — cheat sheet

Reference reading for Block A. Everything Ignition stores on disk, organized by *who owns it* and *whether it belongs in git*.

## The three buckets

Everything inside an Ignition gateway's `data/` directory falls into one of three buckets:

| Bucket | What | Owner | In git? |
|---|---|---|---|
| **Project-level** | Per-project resources: views, scripts, tags, UDTs | The application (versioned, peer-reviewed) | **Yes** |
| **Gateway-level** | Cross-project config: DB connections, identity providers, tag history connections, enabled modules | The application (versioned, but smaller scope) | **Yes** |
| **Operational** | Runtime state: internal DBs, logs, temp, metadata, runtime users | The gateway (gateway owns this, full stop) | **No** |

The whole point of Block A is to internalize this split. If you can answer "which bucket?" for any file in `data/`, you'll know how to version, deploy, and roll back.

## The `data/` directory, top to bottom

What you'll see when you `ls /usr/local/bin/ignition/data` inside the container:

### `projects/`

Project-level. One subdirectory per Ignition project.

```
projects/
└── <project-name>/
    ├── project.json              ← project descriptor (title, parent, enabled)
    ├── views/
    │   └── <ViewName>/
    │       └── view.json
    ├── scripts/
    │   └── <library>.py
    ├── tags/
    │   └── <provider>.json
    ├── udts/
    │   └── <type>.json
    ├── resources/                ← images, attachments
    └── transforms/               ← Perspective view transforms
```

This is the bread and butter of CI/CD for Ignition. The whole `projects/<name>/` directory is what the `deploy.yml` workflow ships onto the **dev** gateway (on push to `main`) and what `release.yml` ships onto **prod** (on tag push) in Block B. On the **local** gateway it just sits there via bind mount — edit-and-scan, no copy step.

### `config/`

Gateway-level config. Shared across all projects.

```
config/
└── resources/
    ├── datasources/              ← database connections
    │   └── <name>.json
    ├── identity/                 ← identity providers (LDAP, OIDC, etc.)
    ├── tag-history/              ← tag history connection definitions
    └── …                         ← many other resource types
```

Resources here are *referenced* by projects but defined gateway-wide. A view in `projects/<x>/views/Main/view.json` might query a database — but the database connection lives in `config/resources/datasources/`. Move a project to a new gateway and you'd port the project; you'd also port the matching resources.

### `modules.json`

Gateway-level. A simple list of which modules to enable.

```json
{"modules": ["com.inductiveautomation.perspective", "..."]}
```

Editing this file *plus* a config scan changes which modules the gateway loads. Versioning this is good practice — it documents the gateway's dependency surface.

### `modules/`

Gateway-level binaries. `.modl` files for each installed module.

- **In git?** Generally **no**. Modules are large binary artifacts. Pin module *versions* in a manifest (e.g., a separate `module-versions.txt`); install modules separately via your runner setup or a custom Docker image.
- For lab 04, the host bind mount on `modules.json` enables modules at startup; the gateway downloads/installs the matching `.modl` files automatically.

### `db/`, `users.idb`

**Operational.** Internal H2 / SQLite stores. The gateway is constantly reading and writing these.

- **In git?** Absolutely **no**.
- **Backup story?** Gateway-level backup (`gwbk` file), not git.

### `logs/`, `temp/`, `.metadata/`

**Operational.** Runtime breadcrumbs.

- **In git?** No.
- **Backup?** Often you don't even back these up — they're regenerable.

### `local/`

Per-instance overrides; rarely touched. Mostly empty. Ignore unless you're doing something unusual.

## The two questions to ask

For any file you see on a running gateway, ask:

1. **Would this file be different on a teammate's identical clone?** If yes → operational. If no → versionable.
2. **Would I want this file in git history?** If yes → versioned. If no → ignored / backed up some other way.

These two questions correctly classify ~99% of `data/` contents.

## Minimal example for the lab

Block A's "you do" suggests creating a project on disk manually if you don't have the Designer installed. Here's a minimal viable structure:

```bash
mkdir -p projects/sample/views/Hello
cat > projects/sample/project.json <<'EOF'
{"title":"Sample","description":"Demo project for Block A","parent":"","enabled":true,"inheritable":false}
EOF
cat > projects/sample/views/Hello/view.json <<'EOF'
{
  "custom": {},
  "params": {},
  "props": { "defaultSize": { "height": 600, "width": 800 } },
  "root": { "type": "ia.container.coord", "version": 0 }
}
EOF
```

After triggering a project scan against the local gateway (`scripts/trigger-scan.sh projects --gateway local`), this project should appear in the local gateway UI. It won't look like much — that's the point. You'll add a real one before the cohort runs the lab.

## What changes when

Some changes require **only** a scan; others require a **restart**.

| Change | Scan only? |
|---|---|
| Add/modify a Perspective view | ✓ |
| Add/modify a project script | ✓ |
| Add/modify a tag UDT | ✓ |
| Add a database connection | ✓ (config scan) |
| Add/remove a module from `modules.json` | ✗ — needs restart |
| Change gateway memory (`-m` arg) | ✗ — needs restart |
| Change Java args | ✗ — needs restart |

The shipped `scripts/trigger-scan.sh` only handles the scan-able cases. For the restart cases, the deploy needs an extra step (`docker compose restart ignition-local` / `-dev` / `-prod` in the lab; `Restart-Service` or `systemctl restart` on a real host).

## Further reading

- [Inductive Automation Docker image docs](https://docs.inductiveautomation.com/docs/8.3/platform/docker-image/) — what the official image expects under `data/`
- [Ignition 8.3 Configuration files](https://docs.inductiveautomation.com/docs/8.3/configuration/) — official descriptions of resource files
- [Project resources](https://docs.inductiveautomation.com/docs/8.3/platform/projects/) — what's a project, what's a resource
