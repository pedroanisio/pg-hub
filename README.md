---
disclaimer:
  notice: >-
    No information within this document should be taken for granted.
    Any statement or premise not backed by a real logical definition
    or verifiable reference may be invalid, erroneous, or a hallucination.
  generated_by: "Claude Opus 4.7 via Claude Code"
  date: "2026-05-01"
---

# infra

Personal dev-machine infrastructure — small, self-contained tools that make
running multiple local codebases on one box less painful.

## Contents

| Path                       | What it is                                                |
|----------------------------|-----------------------------------------------------------|
| [pg-hub/](pg-hub/)         | Central Postgres manager: one container per project, distinct host ports, single `projects.yaml` as source of truth. |

## Conventions

- **Source-of-truth files** are committed; **generated files** (e.g.
  `pg-hub/docker-compose.yml`) are not.
- Each tool ships a CLI under `bin/` symlinked into `~/.local/bin/`.
- Configuration is plain YAML — edit via the CLI, not by hand.
- Defaults target a single dev box: no TLS, no secrets manager, no HA.

## Adding a new tool

Follow the pg-hub layout:

```
~/infra/<tool>/
├── <config>.yaml        # source of truth
├── bin/<tool>           # CLI on PATH
└── README.md            # what it does, install, commands, design, non-goals
```
