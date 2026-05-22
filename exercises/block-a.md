# Block A — Ignition 8.3 file structure decoded

**Duration:** ~90 minutes
* 20 min demo
* 20 min we-do
* 30 min you-do
* 15 min debrief
* ~5 min buffer

## Goal

You should leave this block able to:

- Navigate an Ignition 8.3 gateway's `data/` directory and name what each top-level item is for
- Explain the difference between **project-level**, **gateway-level**, and **operational** state
- Predict which files appear on disk when you make a given change in the gateway UI
- Decide which of those files belong in git and which don't

## Pre-flight

```bash
git fetch --tags
git checkout block-a-start
cp -n .env.example .env
docker compose up -d
# Wait ~60s for the gateway to come up, then:
curl -fsS http://localhost:8088/StatusPing && echo " ← gateway up"
```

Open <http://localhost:8088> (the `local` gateway — the one you'll work with for Block A) in your browser. Login: `admin` / `lab04password` (or whatever you set as `GATEWAY_ADMIN_PASSWORD_LOCAL` in `.env`). Dev and prod gateways (`:8089` / `:8090`) come up too but stay empty until Block B.

If you'd like to read ahead: [`docs/ignition-file-structure.md`](../docs/ignition-file-structure.md).

## I do (20 min)

The instructor walks the gateway's `data/` directory live, both from the inside (via `docker exec`) and from the outside (the host bind mounts).

```bash
docker exec -it lab04-ignition-local bash -lc "ls /usr/local/bin/ignition/data"
```

Tour, in order:

1. **`projects/<name>/`** — project-level state. One directory per project. Inside: `project.json` + `views/`, `scripts/`, `tags/`, `udts/`, `resources/`. *This is the thing you'll deploy via file-based CI/CD.*
2. **`config/resources/`** — gateway-level config: database connections, identity providers, tag history connections. Shared across all projects.
3. **`config/`** root — gateway-level non-resource config (small).
4. **`modules.json`** — which modules the gateway has enabled. Gateway-level.
5. **`modules/`** — the actual installed module binaries (`.modl` files). Almost never committed to git; usually managed separately.
6. **`db/`, `users.idb`** — internal H2 / SQLite stores. **Operational state.** Never commit.
7. **`logs/`, `temp/`, `.metadata/`** — runtime stuff. Never commit.

The instructor sketches the three-bucket model on the board:

```
PROJECT-LEVEL     GATEWAY-LEVEL              OPERATIONAL
projects/<x>/     config/resources/          db/, users.idb, logs/, temp/
                  modules.json               .metadata/
                  modules/                   (anything that changes at runtime
                                              without you touching the gateway)
```

The deploy story is different for each bucket:

- **Project-level:** copy → trigger project scan. No restart needed.
- **Gateway-level:** copy → trigger config scan. *Sometimes* a restart is needed (depends on what changed).
- **Operational:** never deployed. Gateway owns this state; backups are the recovery mechanism.

## We do (20 min)

Following along on your own gateway:

1. `docker exec -it lab04-ignition-local bash -lc "ls -la /usr/local/bin/ignition/data"` — list every top-level entry. Note what each is.
2. In the gateway UI, go to **Config → Networking → Web Server**. Change the *Idle Timeout* to something different (default is 300). Save.
3. Find the file that changed on disk:
   ```bash
   docker exec lab04-ignition-local find /usr/local/bin/ignition/data/config -newer /tmp/marker -type f 2>/dev/null
   ```
   (Set a marker first: `docker exec lab04-ignition-local touch /tmp/marker` before step 2.)
4. Open that file with `docker exec lab04-ignition-local cat <path>`. Notice what's in there.
5. In the gateway UI, **Config → Databases → Connections → New**. Add a Postgres datasource pointing at host `timescaledb`, port `5432`, db `ignition_loc`, user/password from your `.env` (`POSTGRES_USER` / `POSTGRES_PASSWORD`, default `ignition` / `ignition`). Save. The hostname `timescaledb` is the compose service name — the local gateway resolves it on the lab's docker network.
6. Repeat step 3 — find the new file. Where does the datasource config live on disk? Gateway-level or project-level?

## You do (30 min)

Solo. Make a series of changes in the gateway UI and answer: **which bucket is each in, and where does it live on disk?**

Pick three changes from this list (or invent your own). For each, document in `NOTES.local.md`:

- **What you changed** (one sentence)
- **Where it lives on disk** (path relative to `data/`)
- **Bucket**: project-level / gateway-level / operational
- **One sentence: would you commit this to git?**

Suggested changes:

1. Create a new **Perspective view** in a project (you may need to create a project first via the Designer; if you don't have the Designer installed, **simulate** by creating a directory `projects/sample/views/Hello/view.json` with minimal content from [`docs/ignition-file-structure.md`](../docs/ignition-file-structure.md))
2. Change the **gateway timezone** (Config → System → Time)
3. Add a **new user** to the gateway (Config → Security → Users)
4. Add a **new tag provider** (Config → Tags → Realtime Tag Providers)
5. Enable a **new module** (toggle a module in Config → Modules)

End state should match `block-a-end` (your `NOTES.local.md` is gitignored, so the tag itself doesn't change much — but conceptually you've earned it by completing the classification).

## Stretch challenge `[OPTIONAL]`

Draft a starter `.gitignore` for an Ignition project repo. Include patterns for the **operational** bucket (logs, db, idb, temp). Compare with the shipped [`.gitignore`](../.gitignore). What did you miss? What did the shipped one miss?

This is the precursor to `.dockerignore` (lab-04-image-based) and `.deployignore` (Block B of *this* lab — already shipped in the repo for you to study).

## Debrief (15 min)

- What surprised you about the on-disk layout?
- Which bucket has the trickiest deploy story? (Hint: it's the one that *sometimes* needs a restart.)
- For your current customer's CI/CD: where does each bucket live, and which are versioned?
- What's the smallest atomic change you could meaningfully deploy? (This frames Block B.)
