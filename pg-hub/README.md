---
disclaimer:
  notice: >-
    No information within this document should be taken for granted.
    Any statement or premise not backed by a real logical definition
    or verifiable reference may be invalid, erroneous, or a hallucination.
  generated_by: "Claude Opus 4.7 via Claude Code"
  date: "2026-05-01"
---

# pg-hub

Central dev-machine Postgres manager: **one container per project, each on a
distinct host port**, with a single source of truth and a small CLI.

Solves the "port 5432 is already taken" problem when you run several
codebases locally and each one ships its own `docker-compose.yml` binding the
default port.

## Layout

```
~/infra/pg-hub/
├── projects.yaml        # source of truth — edit via CLI
├── docker-compose.yml   # GENERATED from projects.yaml (not checked in)
├── bin/pg-hub           # CLI
└── README.md
```

## Install

```sh
# one-time: put the CLI on PATH
ln -s ~/infra/pg-hub/bin/pg-hub ~/.local/bin/pg-hub

# PyYAML is the only runtime dependency
uv tool install pyyaml   # or: pipx install pyyaml
```

## Commands

```sh
pg-hub claim <name> [--port N] [--db ...] [--user ...] [--password ...] [--if-missing]
pg-hub release <name> [--purge]
pg-hub list | status
pg-hub up [<name> ...]                      # all if none given
pg-hub down [<name> ...]
pg-hub psql <name>
pg-hub env <name> [--format dotenv|export|json|compose-fragment]
pg-hub ensure <name> [--port ...] [--timeout 30] [--format ...]
pg-hub doctor [--json]
pg-hub regen
```

### Common flows

```sh
pg-hub claim registry                       # allocate next free port
pg-hub claim my-api --port 5440 --db api    # pin a port + custom db name
pg-hub up registry                          # start one container
pg-hub list                                 # show allocations + running state
pg-hub env registry >> /path/to/proj/.env   # dotenv (default)
pg-hub psql registry                        # interactive psql
pg-hub release registry --purge             # remove + delete volume
```

### `ensure` — one-shot, idempotent

Claims the project (if not already claimed), starts the container, waits for
the healthcheck to go green, and prints the env snippet. Safe to run from CI
or project bootstrap scripts:

```sh
pg-hub ensure my-api --format json
```

### `env --format`

| format             | use                                                                               |
|--------------------|-----------------------------------------------------------------------------------|
| `dotenv` (default) | append to a project `.env`                                                        |
| `export`           | `eval "$(pg-hub env my-api --format export)"`                                     |
| `json`             | machine-readable; pipe into `jq`                                                  |
| `compose-fragment` | commented snippet for an app's `docker-compose.yml` (via `host.docker.internal`)  |

### `claim --if-missing`

Idempotent claim — if the project is already claimed, prints its env snippet
instead of erroring. Useful in setup scripts.

### `doctor`

Self-assessment of hub state: validates `projects.yaml`, detects drift
between `projects.yaml` and `docker-compose.yml`, and checks container/port/
volume health. Use `--json` for scripting.

## Wiring a project to the hub

```sh
pg-hub claim myproj
pg-hub env myproj >> /path/to/myproj/.env
pg-hub up myproj
```

For an app running inside its own Docker network, use the compose fragment so
the app reaches the hub via `host.docker.internal`:

```sh
pg-hub env myproj --format compose-fragment
```

## Design

- **Port range:** `5433–5500`. `5432` is intentionally free so any
  system/ambient Postgres can coexist.
- **Postgres version:** pinned at `postgres:17-alpine` by default, overridable
  via the top-level `postgres_image:` field in `projects.yaml`.
- **Volumes:** named Docker volumes (`pg-hub-<project>`); survive container
  recreation. Only `release --purge` deletes data.
- **Source of truth:** `projects.yaml`. `docker-compose.yml` is always
  regenerated; never hand-edit it.
- **Healthchecks:** every service has a `pg_isready` healthcheck; `ensure`
  blocks until it goes green.

## Non-goals

- Not a production / multi-host orchestrator — dev-box convenience only.
- Not a secrets manager — passwords are plaintext dev defaults.
- No TLS, no backups, no replication.
