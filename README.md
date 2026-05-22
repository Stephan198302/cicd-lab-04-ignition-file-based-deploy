# Lab 04 — Ignition file-based deploy

Day 3, Blocks A and B of the [CI/CD for Ignition Masterclass](https://github.com/mustry-academy/cicd-masterclass).

> Decode the Ignition 8.3 file structure, then build a file-based deploy pipeline that promotes project changes from a local working gateway, through a dev environment on push to main, to a prod environment on tag release — all with a hot scan, no gateway restarts.

This is the **first lab where Ignition shows up**. Labs 01–03 deliberately stayed on a generic Python stack so we could teach Git, branching, PRs, linters, and GitHub Actions without the Ignition file format adding noise. The plumbing is solid; now we point it at three real gateways that simulate a local → dev → prod promotion flow.

Block C (image-based deploys) lives in [`cicd-lab-05-ignition-image-based-deploy`](https://github.com/mustry-academy/cicd-lab-05-ignition-image-based-deploy). Block D (multi-gateway) lives in [`cicd-lab-06-multi-gateway-deploy`](https://github.com/mustry-academy/cicd-lab-06-multi-gateway-deploy).

## Prerequisites

- A fork of this repo (the self-hosted runner registers against your fork, not the upstream)
- A GitHub Personal Access Token with `repo` scope — the runner uses it to auto-register itself; never leaves your `.env`
- **≥ 8 GB free RAM for Docker** — three Ignition gateways each cap at 1 GB, plus TimescaleDB, the runner, and the usual Docker Desktop overhead
- _Optional but recommended:_ pass [`cicd-preflight`](https://github.com/mustry-academy/cicd-preflight) so unrelated env issues don't bite you mid-lab
- _Background reading:_ [Lab 03](https://github.com/mustry-academy/cicd-lab-03-github-actions) covers the GitHub Actions fundamentals this lab builds on, but this lab stands alone — the self-hosted runner ships in `docker-compose.yaml`, no manual `docker run` of `myoung34/github-runner` required

## Quick start

```bash
gh repo clone mustry-academy/cicd-lab-04-ignition-file-based-deploy
cd cicd-lab-04-ignition-file-based-deploy
cp .env.example .env
scripts/setup.sh    # brings up the stack, waits for all three gateways, prints credentials
```

Once setup finishes you have three Ignition gateways:

| Gateway | URL | Source of project files |
|---|---|---|
| `local` | http://localhost:8088 | Bind-mounted from `./projects/` and `./services/config/` — edits show up immediately |
| `dev` | http://localhost:8089 | Empty until `deploy.yml` runs on push to `main` |
| `prod` | http://localhost:8090 | Empty until `release.yml` runs on tag push `v*` |

Login to any of them with the credentials from `.env` (`GATEWAY_ADMIN_USERNAME_LOCAL/_DEV/_PROD`, default `admin / lab04password`).

> **Trial mode:** each gateway runs in 2-hour trial mode. Reset via *Gateway → Config → Licensing → Reset Trial* — unlimited and entirely legal for development. You'll do this **three times** if you keep all three gateways up long enough.

## Lab structure

| Block | Topic | Exercise |
|---|---|---|
| A | Ignition 8.3 file structure decoded | [`exercises/block-a.md`](./exercises/block-a.md) |
| B | File-based deploy mechanic | [`exercises/block-b.md`](./exercises/block-b.md) |

> Blocks C and D of Day 3 are in separate labs ([image-based](https://github.com/mustry-academy/cicd-lab-05-ignition-image-based-deploy), [multi-gateway](https://github.com/mustry-academy/cicd-lab-06-multi-gateway-deploy)).

## Checkpoints

```bash
git fetch --tags
git checkout block-a-start
git checkout block-a-end
git checkout block-b-start
git checkout block-b-end
```

## Repo layout

```
cicd-lab-04-ignition-file-based-deploy/
├── README.md
├── docker-compose.yaml                 ← three Ignition gateways + TimescaleDB + bundled self-hosted runner
├── .env.example                        ← copy to .env before running
├── .deployignore                       ← what NOT to copy onto dev/prod gateways
├── .gitattributes                      ← JSON line-ending normalization + binary markers
├── .github/
│   ├── workflows/
│   │   ├── ci.yml                      ← PR validation (ubuntu-latest, free)
│   │   ├── deploy.yml                  ← push to main → dev gateway (self-hosted)
│   │   ├── release.yml                 ← tag v* → prod gateway (self-hosted)
│   │   └── azure.yml                   ← reference Azure DevOps pipeline (ignored by GH)
│   └── pull_request_template.md
├── exercises/
│   ├── block-a.md
│   └── block-b.md
├── docs/                               ← reference reading
│   ├── ignition-file-structure.md
│   └── file-based-deploy-pattern.md
├── instructor-notes/                   ← answer keys (read after solo work)
│   ├── block-a-key.md
│   └── block-b-key.md
├── scripts/
│   ├── setup.sh                        ← bootstraps the whole stack
│   ├── teardown.sh                     ← stop the stack (with --volumes to wipe)
│   ├── trigger-scan.sh                 ← curl the scan API (any gateway via --gateway)
│   ├── lib.sh                          ← shared helpers
│   └── git-hooks/                      ← skip-worktree hooks for Ignition state files
├── projects/                           ← project content (bind-mounted into `local` only)
│   └── README.md
└── services/
    ├── config/                         ← gateway-level config (bind-mounted into `local`)
    ├── modules.json                    ← module enablement (shared by all three gateways)
    └── timescaledb-init/               ← creates ignition_dev + ignition_prd databases
```

## The Compose stack

Three Ignition 8.3 gateways + one TimescaleDB. The three gateways simulate the classic local → dev → prod promotion:

- **`ignition-local`** bind-mounts `./projects/` and `./services/config/` from the host. Anything you write into those paths on your laptop is *immediately on disk* inside the local gateway. Hit `scripts/trigger-scan.sh both` to make the gateway notice. That's the tight inner feedback loop.
- **`ignition-dev`** uses a named volume (not a bind mount). It starts empty; the deploy workflow (`deploy.yml`) `docker cp`s the working tree into the container and triggers a scan. Mirrors how a real shared dev environment gets fed by CI.
- **`ignition-prod`** is the same shape as dev, populated by the release workflow (`release.yml`) when you push a tag.

The single TimescaleDB hosts three logical databases (`ignition_loc`, `ignition_dev`, `ignition_prd`) so each gateway can have its own historian data without crosstalk.

Memory is set to 1 GB per gateway via Compose limits. Tight but workable; bump it in `docker-compose.yaml` if you see GC pauses.

## A note on the CI/CD workflows

Three workflows under [`.github/workflows/`](./.github/workflows/):

| File | Trigger | Runner | Purpose |
|---|---|---|---|
| [`ci.yml`](./.github/workflows/ci.yml) | PR to `main` | `ubuntu-latest` (free) | Validate JSON, `.deployignore` syntax, and the workflow files themselves. |
| [`deploy.yml`](./.github/workflows/deploy.yml) | Push to `main`, manual | `[self-hosted, lab04]` | File-based deploy to the **dev** gateway via `docker cp`. |
| [`release.yml`](./.github/workflows/release.yml) | Tag `v*`, manual | `[self-hosted, lab04]` | File-based deploy to the **prod** gateway. Same mechanics, different environment. |

Both deploy workflows need:

- The bundled self-hosted runner (`github-runner` service in `docker-compose.yaml`) registered against your fork with the `lab04` label. It auto-registers using `RUNNER_GITHUB_PAT` from `.env` and shares the host's Docker daemon (mounted `/var/run/docker.sock`) so the workflows can `docker cp` files into the dev/prod gateway containers. If you'd rather use your own runner instead, set `runner.labels` to include `lab04` and skip the bundled service.
- A GitHub **environment** per workflow with the right secret + variables:

| Scope | Name | Type | Purpose |
|---|---|---|---|
| Environment `lab-gateway-dev` (deploy.yml) | `IGNITION_API_KEY` | Secret | Token from the dev gateway with Project Scan + Config Scan permission |
| Environment `lab-gateway-dev` | `IGNITION_URL` | Variable (optional) | Defaults to `http://ignition-dev:8088` (bundled-runner case). Override to `http://localhost:8089` if your runner is on the host. |
| Environment `lab-gateway-dev` | `IGNITION_CONTAINER` | Variable (optional) | Defaults to `lab04-ignition-dev` |
| Environment `lab-gateway-prod` (release.yml) | (same three) | | Defaults: URL `http://ignition-prod:8088`, container `lab04-ignition-prod` |

Add **required reviewers** on the `lab-gateway-prod` environment if you want a manual approval gate on tag releases — common pattern, no workflow change required.

Block B walks through the end-to-end setup.

## Licence

Apache 2.0 — see [`LICENSE`](./LICENSE).
