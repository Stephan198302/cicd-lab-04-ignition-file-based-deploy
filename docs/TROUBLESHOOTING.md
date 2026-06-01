# Troubleshooting

Quick fixes for the things that bite people during Labs A and B. Work top to bottom; most issues
are one of the first three.

## The stack won't start / `scripts/setup.sh` fails

- **Docker isn't running or isn't Compose v2.** `docker compose version` must work (note the space —
  not `docker-compose`). Start Docker Desktop and re-run `scripts/setup.sh` (it's idempotent).
- **Not enough RAM.** Three gateways at 1 GB each + TimescaleDB + the runner need **≥ 8 GB free**.
  If gateways crash or never reach RUNNING, raise Docker's memory or bump the per-gateway limit in
  [`docker-compose.yaml`](../docker-compose.yaml).
- **Port already in use.** 8088/8089/8090 (gateways) or 5432 (TimescaleDB) taken by something else.
  Stop the other process or change the host port mappings in `docker-compose.yaml`.

## A gateway never reaches RUNNING

- Give it time — a cold JVM start can take 60–120 s per gateway. `setup.sh` polls up to ~240 s each.
- Check logs (note: use `docker logs`, the container name — *not* `docker compose logs`, which wants
  the service name):
  ```bash
  docker logs --tail 200 lab04-ignition-local     # or -dev / -prod
  ```
- **Trial expired.** Each gateway runs in 2-hour trial mode. After it lapses the gateway stops
  serving. Reset via *Gateway → Config → Licensing → Reset Trial* (unlimited, legal for dev).

## `git status` shows dozens of `config.json` / `resource.json` changes

Ignition rewrites these on every interaction. The repo ships a git hook that marks them
`skip-worktree` so they stay out of `git status` — but the hook only runs after a
checkout/merge/rewrite, and `scripts/setup.sh` installs it. If you see the churn:

```bash
scripts/setup.sh                                  # (re)installs the hooks, then
bash scripts/git-hooks/skip-worktree-ignition-resources
```

To intentionally change a seed config and commit it:
```bash
git update-index --no-skip-worktree <path>
# edit, commit, push — the next pull re-applies skip-worktree
```

## The self-hosted runner is offline / jobs queue forever

- Container up? `docker compose ps github-runner` and `docker logs github-runner` (look for
  *"Listening for Jobs"*).
- `RUNNER_REPO_URL` in `.env` must point at **your fork**, not the upstream.
- `RUNNER_GITHUB_PAT` must be a real PAT with `repo` scope — the `.env.example` placeholder won't
  register. After editing `.env`: `docker compose restart github-runner`.
- In your fork: *Settings → Actions → Runners* should list it online with the `self-hosted, lab04`
  labels.

## The deploy 403s on the scan step

The `IGNITION_API_KEY` for that environment is missing, wrong, or under-scoped. Generate the key
**on the target gateway's own UI** (*Config → Security → API Keys*), scope it to **Project Scan +
Config Scan**, and set it as the `IGNITION_API_KEY` secret on the matching GitHub environment
(`lab-gateway-dev` / `lab-gateway-prod`). Keys are **per-gateway** — a dev key won't authenticate
against prod.

## The deploy ran but my change isn't visible

- Are you looking at the right gateway? `local` = :8088, `dev` = :8089, `prod` = :8090.
- Did the scan return HTTP 200? `scripts/trigger-scan.sh` pretty-prints the response with a
  `lastScanTimestamp`. Files on disk without a successful scan = gateway hasn't reloaded.
- Module enable/disable (`services/modules.json`) needs a **restart**, not a scan:
  `docker compose restart ignition-dev`.

## Validate before you push

```bash
scripts/validate.sh    # JSON parse + .deployignore syntax + actionlint — mirrors CI
```

Still stuck? The instructor answer keys ([block-a-key.md](../instructor-notes/block-a-key.md),
[block-b-key.md](../instructor-notes/block-b-key.md)) have a deeper failure-mode walkthrough.
