---
disclaimer:
  notice: >-
    No information within this document should be taken for granted.
    Any statement or premise not backed by a real logical definition
    or verifiable reference may be invalid, erroneous, or a hallucination.
  generated_by: "Claude Opus 4.7 via Claude Code"
  date: "2026-04-19"
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

## Usage

```sh
pg-hub claim registry                     # allocate next free port
pg-hub claim my-api --port 5440 --db api  # pin a port + custom db name
pg-hub up registry                        # start one container
pg-hub up                                 # start all claimed containers
pg-hub list                               # show allocations + running state
pg-hub env registry                       # print env snippet for a project .env
pg-hub psql registry                      # interactive psql
pg-hub down registry                      # stop (volume retained)
pg-hub release registry --purge           # remove + delete volume
pg-hub regen                              # rebuild docker-compose.yml after hand edits
```

## Wiring a project to the hub

```sh
pg-hub claim myproj
pg-hub env myproj >> /path/to/myproj/.env   # or copy the snippet into existing .env
pg-hub up myproj
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

## Non-goals

- Not a production / multi-host orchestrator — dev-box convenience only.
- Not a secrets manager — passwords are plaintext dev defaults.
- No TLS, no backups, no replication.
