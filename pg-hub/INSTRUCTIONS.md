---
disclaimer:
  notice: >-
    No information within this document should be taken for granted.
    Any statement or premise not backed by a real logical definition
    or verifiable reference may be invalid, erroneous, or a hallucination.
  generated_by: "Claude Opus 4.7 via Claude Code"
  date: "2026-04-19"
---

# Postgres via pg-hub — Setup Instructions

> **Copy this file into each codebase that needs a local Postgres.**
> Fill in the one field below; the rest is deterministic.

This project uses **pg-hub** — a single place on the host that runs one
Postgres container per project on a distinct port. You do **not** run a
project-local `docker-compose` on `:5432`; that path leads to conflicts when
several codebases are active at once.

Hub location: `~/infra/pg-hub/`  
Port range: `5433–5500` (host port `5432` is intentionally left free)

---

## 0. Fill this in

| Field | Value |
|---|---|
| `PROJECT_NAME` | _e.g._ `registry`, `my-api` — lowercase, hyphen-separated |
| `DB_NAME` *(optional)* | defaults to `PROJECT_NAME` with `-` → `_` |
| `DB_USER` *(optional)* | defaults to `PROJECT_NAME` with `-` → `_` |
| `NEEDS_TEST_DB` | `yes` / `no` — claim a separate `<name>-test` DB? |

Throughout this doc, `<name>` means your `PROJECT_NAME`.

---

## 1. One-time host setup (skip if pg-hub is already installed)

```sh
# Put pg-hub on PATH
ln -s ~/infra/pg-hub/bin/pg-hub ~/.local/bin/pg-hub

# Runtime dep
uv tool install pyyaml     # or: pipx install pyyaml

# Sanity check
pg-hub --help
```

If `~/infra/pg-hub/` does not exist, ask whoever owns this host to bootstrap
it (or clone the hub repo into that path).

---

## 2. Claim a database for this project

The simplest path — one idempotent command, safe to put in a `make setup`:

```sh
pg-hub ensure <name> > .env.pg-hub        # claims if missing, starts, waits healthy
# If you also need a test DB:
pg-hub ensure <name>-test > .env.pg-hub.test
```

`ensure` writes the env snippet to stdout; status messages go to stderr.

Lower-level alternative (manual claim + start):

```sh
pg-hub claim <name>                       # allocate (errors if already claimed)
pg-hub claim <name> --if-missing          # idempotent variant
pg-hub claim <name>-test                  # separate test DB if needed
```

Custom overrides (rarely needed):

```sh
pg-hub claim <name> --port 5440 --db custom_db --user myuser --password secret
```

---

## 3. Wire `.env`

Generate the env snippet and append to your project's `.env`:

```sh
pg-hub env <name> >> .env
# with a test DB:
pg-hub env <name>-test | sed 's/^POSTGRES_/POSTGRES_TEST_/' >> .env
```

Other formats:

```sh
pg-hub env <name> --format json              # for tooling
eval "$(pg-hub env <name> --format export)"  # one-shot shell export
pg-hub env <name> --format compose-fragment  # if your app runs in docker too
```

When your app runs inside its own container and needs to reach the hub, use
`host.docker.internal` (see the `compose-fragment` output for the exact
snippet to paste into your project's `docker-compose.yml`).

Verify the resulting `.env` contains:

```
POSTGRES_HOST=localhost
POSTGRES_PORT=<allocated port>
POSTGRES_DB=<db>
POSTGRES_USER=<user>
POSTGRES_PASSWORD=<password>
DATABASE_URL=postgresql://<user>:<password>@localhost:<port>/<db>
```

> ⚠ Never commit `.env`. Add it to `.gitignore` if it is not already.

---

## 4. Start the container(s) and apply migrations

```sh
pg-hub up <name>           # start just this project's container
pg-hub up                  # or: start all claimed containers

# project-specific migration step (replace with your own):
make migrate               # or: alembic upgrade head, etc.
```

Confirm:

```sh
pg-hub list
# NAME          PORT   DATABASE   USER       STATUS
# <name>        5433   <db>       <user>     running
```

---

## 5. Common tasks

| Task | Command |
|---|---|
| Open psql | `pg-hub psql <name>` |
| Show ports/status | `pg-hub list` |
| Stop container (data kept) | `pg-hub down <name>` |
| Restart after reboot | `pg-hub up <name>` |
| Reset DB (drop volume) | `pg-hub release <name> --purge && pg-hub claim <name>` |
| Print env snippet again | `pg-hub env <name>` |
| Connect from host tool | use `DATABASE_URL` from `.env` |

---

## 6. Troubleshooting

**`port already in use`** — another process is on the allocated port. Check
with `ss -ltnp sport = :<port>`. Release + re-claim without `--port` to let
the hub pick the next free one:

```sh
pg-hub release <name>
pg-hub claim <name>
```

**`docker-compose.yml missing`** — run `pg-hub regen`.

**`connection refused`** — container not started yet. `pg-hub up <name>`
and wait 1–2 s for the healthcheck. Confirm with
`docker logs pg-hub-<name>`.

**Forgot the password** — `pg-hub env <name>` prints the current values.
They live in `~/infra/pg-hub/projects.yaml` (plaintext, dev-only).

**Tests need isolation** — claim a dedicated `<name>-test` DB (step 2). Do
not run tests against the main DB.

**Something feels off** — run `pg-hub doctor`. It compares
`projects.yaml`, `docker-compose.yml`, and the actual container/volume/port
state, and reports drift with a non-zero exit. Use `--json` in CI.

---

## 7. What NOT to do

- ❌ Do not run a project-local `docker-compose up` that binds `:5432`.
- ❌ Do not hand-edit `~/infra/pg-hub/docker-compose.yml` — it is
  regenerated from `projects.yaml` by the `pg-hub` CLI.
- ❌ Do not hard-code `POSTGRES_PORT=5432` anywhere; always read from `.env`.
- ❌ Do not reuse pg-hub passwords in any non-dev environment.

---

## 8. Removing this project's DB

```sh
pg-hub down <name>                    # stop only
pg-hub release <name>                 # remove allocation (volume kept)
pg-hub release <name> --purge         # remove allocation AND delete data
```
