# p4n4 — Multi-Repository Architecture

> Reference document for scaffolding the p4n4 GitHub organisation and its constituent repositories.
> Version 0.1 | 2026

---

## Table of Contents

1. [Overview](#1-overview)
2. [Repository Inventory](#2-repository-inventory)
3. [Directory Structures](#3-directory-structures)
4. [Cross-Repo Dependencies](#4-cross-repo-dependencies)
5. [Network & Secrets Strategy](#5-network--secrets-strategy)
6. [Versioning & Release Strategy](#6-versioning--release-strategy)
7. [CI/CD per Repository](#7-cicd-per-repository)
8. [Bootstrap Order](#8-bootstrap-order)
9. [GitHub Organisation Setup](#9-github-organisation-setup)
10. [Future Extractions (Phase 3+)](#10-future-extractions-phase-3)

---

## 1. Overview

p4n4 is organised across **12 repositories** under the `raisga` GitHub organisation. Each repository
has a single, well-defined responsibility and can be developed, versioned, and released
independently. The `p4n4` umbrella repo is a **monorepo** that aggregates all sub-repos as Git
submodules, grouped by concern.

```
raisga/
├── .github                 ← org profile, shared issue templates, org-wide Actions workflows
└── p4n4                    ← umbrella (monorepo with submodules)
    ├── stacks/
    │   ├── iot             ← p4n4-iot: Mosquitto · Node-RED · InfluxDB · Grafana
    │   ├── ai              ← p4n4-ai: Ollama · Letta · n8n
    │   └── edge            ← p4n4-edge: Edge Impulse inference stack
    ├── core/
    │   ├── lib             ← p4n4-lib: shared library (stacks ↔ clients)
    │   └── hw              ← p4n4-hw: hardware designs + RPi5 GPIO scripts
    ├── shared/
    │   └── templates       ← p4n4-templates: community template registry & index
    ├── client/
    │   ├── cli             ← p4n4-cli: Python CLI (published to PyPI as `p4n4`)
    │   ├── api             ← p4n4-api: REST API gateway
    │   └── dashboard       ← p4n4-dashboard: web dashboard client
    ├── demo/
    │   └── emu             ← p4n4-emu: workstation hardware emulator
    └── docs                ← p4n4-docs: full technical documentation site (this repo)
```

> **Naming note:** The `p4n4` umbrella repo and the `p4n4` PyPI package (from `p4n4-cli`) share
> the same short name intentionally — the CLI *is* the primary user-facing product. The umbrella
> repo is the GitHub landing page and monorepo root, not the package source.

---

## 2. Repository Inventory

| Repo | Monorepo path | Type | Standalone? | PyPI / Registry | Description |
|------|---------------|------|-------------|-----------------|-------------|
| `.github` | — | org meta | — | — | Org profile README, shared Actions, issue templates |
| `p4n4` | — | meta / monorepo | — | — | Umbrella: architecture, ADRs, all sub-repos as submodules |
| `p4n4-iot` | `stacks/iot` | stack | ✓ | — | Docker Compose IoT stack; owns `p4n4-net` bridge |
| `p4n4-ai` | `stacks/ai` | stack | ✓ | — | Docker Compose GenAI stack; attaches to `p4n4-net` |
| `p4n4-edge` | `stacks/edge` | stack | ✓ | — | Docker Compose Edge Impulse stack; attaches to `p4n4-net` |
| `p4n4-lib` | `core/lib` | library | ✓ | — | Shared library mediating between stacks and clients |
| `p4n4-hw` | `core/hw` | hardware | — | — | KiCad PCB designs and RPi5 GPIO scripts |
| `p4n4-templates` | `shared/templates` | registry | — | Git-native | Community template index + example templates |
| `p4n4-cli` | `client/cli` | tool | ✓ | `p4n4` on PyPI | Python CLI for scaffolding and lifecycle management |
| `p4n4-api` | `client/api` | service | ✓ | — | REST API gateway (port 8000) |
| `p4n4-dashboard` | `client/dashboard` | frontend | ✓ | — | Web dashboard client |
| `p4n4-emu` | `demo/emu` | tool | ✓ | — | Workstation hardware emulator (Docker resource constraints + QEMU) |
| `p4n4-docs` | `docs` | docs | — | — | Full technical reference; deployable as a static site |

---

## 3. Directory Structures

### 3.1 `.github` (org profile)

```
.github/
├── profile/
│   └── README.md               ← shown on github.com/raisga
├── ISSUE_TEMPLATE/
│   ├── bug_report.yml
│   ├── feature_request.yml
│   └── template_submission.yml
├── PULL_REQUEST_TEMPLATE.md
└── workflows/
    └── stale.yml               ← org-wide stale issue/PR bot
```

---

### 3.2 `p4n4` (umbrella)

```
p4n4/
├── README.md                   ← project overview, quick links to all sub-repos
├── ARCHITECTURE.md             ← this file
├── CONTRIBUTING.md
├── CODE_OF_CONDUCT.md
├── SECURITY.md
├── LICENSE
└── docs/
    ├── adr/                    ← Architecture Decision Records
    │   └── ADR-001.md
    ├── diagrams/               ← network topology, data flow (SVG / Mermaid source)
    │   └── .gitkeep
    ├── index.md
    ├── getting-started.md
    ├── iot-stack.md
    ├── ai-stack.md
    ├── edge-stack.md
    ├── cli-reference.md
    ├── template-registry.md
    └── security.md
```

---

### 3.3 `p4n4-iot` (IoT stack)

**Owns** the `p4n4-net` Docker bridge network. All other stacks attach to it as an external network.

> **Note:** The pre-existing `p4n4-iot` repo uses the network name `p4n4-net`. The new `p4n4-ai` and `p4n4-edge` stacks reference `p4n4-net` as an external network. When running all stacks together, ensure network names are aligned (the CLI handles this automatically via `p4n4 up`).

```
p4n4-iot/
├── docker-compose.yml          ← primary compose file (services + network definition)
├── docker-compose.override.yml ← local dev overrides (bind mounts, debug ports)
├── .env.example                ← standalone .env template
├── .gitignore
├── .dockerignore
├── Makefile                    ← convenience targets: up, down, status, test-mqtt
├── LICENSE
├── README.md
│
├── config/                     ← all service config files under a single top-level dir
│   ├── mosquitto/
│   │   ├── mosquitto.conf
│   │   ├── passwd              ← generated by p4n4-cli; placeholder in repo
│   │   └── acl
│   │
│   ├── node-red/
│   │   ├── flows.json          ← version-controlled base flows (IoT only)
│   │   └── settings.js
│   │
│   └── grafana/
│       └── provisioning/
│           ├── datasources/
│           │   └── datasources.yml
│           └── dashboards/
│               ├── dashboards.yml
│               └── json/
│                   └── iot-overview.json
│
├── scripts/
│   ├── init-sandbox.sh
│   └── selector.sh
│
└── .github/
    └── workflows/
        └── ci.yml
```

**Network block in `docker-compose.yml`:**
```yaml
networks:
  p4n4-net:
    name: p4n4-net
    driver: bridge
```

---

### 3.4 `p4n4-ai` (GenAI stack)

Attaches to `p4n4-net` as an external network (must be created by `p4n4-iot` first, or via `p4n4 up --all`).

```
p4n4-ai/
├── docker-compose.yml
├── docker-compose.override.yml
├── .env.example
├── .gitignore
├── LICENSE
├── README.md
│
├── ollama/
│   └── pull-models.sh          ← helper script: docker exec ollama ollama pull <model>
│
├── letta/
│   └── config/
│       └── letta.conf
│
└── n8n/
    └── workflows/
        ├── alert-enrichment.json
        ├── scheduled-digest.json
        ├── device-onboarding.json
        └── incident-escalation.json
```

**Network block in `docker-compose.yml`:**
```yaml
networks:
  p4n4-net:
    external: true
    name: p4n4-net
```

---

### 3.5 `p4n4-edge` (Edge Impulse stack)

Fully independent. Attaches to `p4n4-net` as external when used with the IoT stack, but can also run standalone on a dedicated network.

```
p4n4-edge/
├── docker-compose.yml
├── docker-compose.override.yml
├── .env.example
├── .gitignore
├── README.md
│
└── edge-impulse/
    └── models/
        └── .gitkeep            ← .eim binaries are NEVER committed; provided at deploy time
```

**Network block in `docker-compose.yml`:**
```yaml
networks:
  p4n4-net:
    external: true
    name: p4n4-net
```

---

### 3.6 `p4n4-cli` (Python CLI / PyPI package)

The `pip install p4n4` package. Bundles all Jinja2 templates internally so scaffold works fully offline.

```
p4n4-cli/
├── pyproject.toml              ← build system, deps, version, PyPI metadata
├── README.md
├── CHANGELOG.md
├── PUBLISHING.md               ← release checklist and PyPI publishing guide
├── .github/
│   └── workflows/
│       ├── ci.yml              ← lint + test on every PR
│       └── publish.yml         ← publish to PyPI on version tag
│
├── p4n4/
│   ├── __init__.py             ← version = "0.1.0"
│   ├── cli.py                  ← Typer app entrypoint; command registration
│   │
│   ├── commands/
│   │   ├── __init__.py
│   │   ├── init.py             ← `p4n4 init` interactive wizard
│   │   ├── add.py              ← `p4n4 add`
│   │   ├── remove.py           ← `p4n4 remove`
│   │   ├── lifecycle.py        ← `p4n4 up / down / status / logs`
│   │   ├── ei.py               ← `p4n4 ei` subcommands
│   │   ├── secret.py           ← `p4n4 secret`
│   │   ├── validate.py         ← `p4n4 validate`
│   │   ├── upgrade.py          ← `p4n4 upgrade`
│   │   └── template.py         ← `p4n4 template` subcommands
│   │
│   ├── scaffold/
│   │   ├── __init__.py
│   │   ├── manifest.py         ← .p4n4.json read / write / validate
│   │   └── renderer.py         ← Jinja2 context builder + file renderer
│   │
│   ├── templates/              ← bundled Jinja2 templates (source of truth for scaffold)
│   │   ├── iot/
│   │   │   ├── docker-compose.yml.j2
│   │   │   ├── env.example.j2
│   │   │   ├── mosquitto/config/mosquitto.conf.j2
│   │   │   ├── mosquitto/config/acl.j2
│   │   │   ├── node-red/flows.json.j2
│   │   │   ├── node-red/settings.js.j2
│   │   │   ├── influxdb/config/influxdb.conf.j2
│   │   │   ├── grafana/provisioning/datasources/influxdb.yaml.j2
│   │   │   └── grafana/provisioning/dashboards/p4n4-base.json
│   │   ├── ai/
│   │   │   ├── docker-compose.yml.j2
│   │   │   ├── env.example.j2
│   │   │   ├── ollama/pull-models.sh.j2
│   │   │   ├── letta/config/letta.conf.j2
│   │   │   └── n8n/workflows/
│   │   │       └── alert-enrichment.json   ← starter workflow (static JSON copy)
│   │   ├── edge/
│   │   │   ├── docker-compose.yml.j2
│   │   │   └── env.example.j2
│   │   └── shared/
│   │       ├── .gitignore.j2
│   │       └── .p4n4.json.j2   ← project manifest template
│   │
│   └── utils/
│       ├── __init__.py
│       ├── docker.py           ← subprocess wrappers for docker/compose calls
│       ├── secrets.py          ← cryptographically secure secret generation
│       └── network.py          ← p4n4-net existence checks and creation
│
└── tests/
    ├── conftest.py
    ├── test_init.py
    ├── test_scaffold.py
    ├── test_manifest.py
    ├── test_secret.py
    └── test_validate.py
```

**`pyproject.toml` key fields:**
```toml
[project]
name = "p4n4"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
  "typer[all]>=0.12",
  "rich>=13",
  "questionary>=2",
  "jinja2>=3",
  "tomli>=2; python_version < '3.11'",
  "pyyaml>=6",
]

[project.scripts]
p4n4 = "p4n4.cli:app"

[project.urls]
Homepage = "https://github.com/raisga/p4n4-cli"
Documentation = "https://github.com/raisga/p4n4-docs"
```

---

### 3.7 `p4n4-templates` (community template registry)

Acts as the `https://github.com/raisga/templates` index resolved by `p4n4 template search`.

```
p4n4-templates/
├── index.json                  ← community template index (short-name → repo URL map)
├── README.md                   ← how to use and contribute templates
├── TEMPLATE_GUIDE.md           ← full authoring guide (p4n4-template.toml spec)
├── .github/
│   └── workflows/
│       └── validate-index.yml  ← CI: validate index.json schema on every PR
│
├── scripts/
│   ├── validate_index.py       ← validates index.json schema
│   └── validate_templates.py   ← validates all p4n4-template.toml files
│
└── examples/
    ├── factory-baseline/       ← full IoT + GenAI + EI starter (manufacturing)
    │   ├── p4n4-template.toml
    │   ├── docker-compose.iot.yml.j2
    │   ├── docker-compose.ai.yml.j2
    │   ├── docker-compose.edge.yml.j2
    │   ├── mosquitto/config/mosquitto.conf.j2
    │   ├── mosquitto/config/acl.j2
    │   ├── node-red/flows.json
    │   ├── grafana/provisioning/dashboards/factory-base.json
    │   └── n8n/workflows/alert-enrichment.json
    │
    └── iot-minimal/            ← IoT-only starter (no AI, no Edge)
        ├── p4n4-template.toml
        ├── docker-compose.iot.yml.j2
        ├── mosquitto/config/mosquitto.conf.j2
        ├── node-red/flows.json
        └── grafana/provisioning/dashboards/minimal.json
```

**`index.json` schema:**
```json
{
  "org": "community",
  "index_version": "1",
  "templates": {
    "factory-baseline": {
      "repo": "https://github.com/raisga/p4n4-templates.git",
      "subdir": "examples/factory-baseline",
      "description": "Full IoT + GenAI + EI stack for discrete manufacturing",
      "tags": ["manufacturing", "vibration", "edge-impulse", "genai"],
      "latest": "0.1.0"
    },
    "iot-minimal": {
      "repo": "https://github.com/raisga/p4n4-templates.git",
      "subdir": "examples/iot-minimal",
      "description": "Minimal IoT-only stack for simple sensor monitoring",
      "tags": ["minimal", "iot"],
      "latest": "0.1.0"
    }
  }
}
```

---

### 3.8 `p4n4-docs` (documentation site)

```
p4n4-docs/
├── README.md                   ← full technical reference
├── ARCHITECTURE.md             ← this file
├── mkdocs.yml                  ← site config (deploy pending)
├── docs/                       ← MkDocs source pages
│   ├── adr/
│   │   └── ADR-001.md
│   ├── diagrams/
│   │   └── .gitkeep
│   ├── index.md
│   ├── getting-started.md
│   ├── iot-stack.md
│   ├── ai-stack.md
│   ├── edge-stack.md
│   ├── hardware.md
│   ├── emulator.md
│   ├── cli-reference.md
│   ├── template-registry.md
│   ├── security.md
│   ├── design.md
│   └── specs.md
└── .github/
    └── workflows/
        └── deploy-docs.yml     ← publish to GitHub Pages on push to main
```

### 3.9 `p4n4-hw` (hardware designs and scripts)

KiCad PCB projects and Raspberry Pi 5 GPIO scripts for the p4n4 platform reference hardware.

```
p4n4-hw/
├── README.md
├── LICENSE
├── dev-board/                  ← KiCad project: main dev board (schematic + PCB)
├── prototypes/
│   └── leds-indicator/         ← KiCad project: LED indicator prototype board
└── scripts/
    └── rpi5/
        ├── p4n4_common.py          ← shared GPIO helpers, service catalogue, TCP probe
        ├── p4n4_boot_sim.py        ← boot sequence simulator (LED blink patterns)
        ├── p4n4_shutdown_sim.py    ← graceful shutdown simulator
        ├── p4n4_health_monitor.py  ← service health monitor (TCP probe + GPIO LED)
        ├── p4n4_button_handler.py  ← physical button (single / double / long press)
        └── p4n4_mqtt_indicator.py  ← MQTT traffic activity indicator
```

All RPi5 scripts share a common GPIO pin assignment (GPIO 17 = status LED, GPIO 27 = push button)
and depend on `RPi.GPIO` plus optionally `paho-mqtt`.

---

### 3.10 `p4n4-emu` (hardware emulator)

Workstation emulator that applies Docker resource constraints per hardware profile so developers
can test against realistic edge hardware limits without a physical board.

```
p4n4-emu/
├── README.md
├── guide.md                    ← step-by-step setup and LED toggle guide
├── pyproject.toml
├── uv.lock
├── LICENSE
├── p4n4_emu/
│   ├── cli.py                  ← Typer entrypoint (p4n4-emu command)
│   ├── profiles/               ← hardware profile definitions (rpi4, rpi5, nuc, mcu-class)
│   ├── overlay/                ← Compose overlay generator (deploy.resources.limits)
│   ├── sim/
│   │   └── sensor_sim.py       ← synthetic MQTT sensor publisher
│   └── hw/
│       └── gpio_stub.py        ← drop-in RPi.GPIO replacement for workstations
└── tests/
```

**Hardware profiles:**

| Profile | CPU | Memory | Disk R/W | Arch |
|---------|-----|--------|----------|------|
| `rpi4` | 4 cores | 3.5 GB | 50 MB/s | arm64 |
| `rpi5` | 4 cores | 7 GB | 100 MB/s | arm64 |
| `mcu-class` | 1 core | 256 MB | 10 MB/s | x86_64 |
| `nuc` | 4 cores | 14 GB | 200 MB/s | x86_64 |

---

## 4. Cross-Repo Dependencies

```
                      ┌─────────────┐
                      │    p4n4     │  (umbrella — no code deps)
                      └──────┬──────┘
                             │ links to
          ┌──────────────────┼──────────────────┐
          │                  │                  │
   ┌──────▼──────┐   ┌───────▼──────┐   ┌──────▼──────┐
   │  p4n4-iot   │   │   p4n4-ai    │   │  p4n4-edge  │
   │ (owns net)  │   │ (ext: net)   │   │ (ext: net)  │
   └──────┬──────┘   └──────────────┘   └─────────────┘
          │ p4n4-net (bridge, created by p4n4-iot)
          │ ← p4n4-ai/edge attach via p4n4-net (see §5.1)
          │
   ┌──────▼──────────────────────────────────────────┐
   │                  p4n4-cli                        │
   │  bundles Jinja2 templates for all three stacks   │
   │  reads p4n4-templates index for `template` cmds  │
   └──────────────────────────────────────────────────┘
          │
   ┌──────▼──────┐
   │p4n4-templates│  (Git-native; no code import)
   └─────────────┘
```

### Runtime dependency order (when deploying all stacks together)

```
1. p4n4-iot    → creates p4n4-net + starts Mosquitto & InfluxDB first
2. p4n4-ai     → attaches to p4n4-net; needs Mosquitto (optional) + InfluxDB healthy
3. p4n4-edge   → attaches to p4n4-net; fully independent of ai stack
```

The `p4n4 up --all` command in the CLI enforces this order via Docker healthcheck polling.

---

## 5. Network & Secrets Strategy

### 5.1 Docker Network

- The pre-existing `p4n4-iot` creates a network named **`p4n4-net`**.
- `p4n4-ai` and `p4n4-edge` declare **`p4n4-net`** as `external: true`.
- When running all stacks together, the CLI (`p4n4 up`) creates and reconciles the shared network automatically. Manual deployments must ensure the network name is consistent across stacks.
- When stacks are used standalone (without the IoT stack), they create their own default network.

### 5.2 Secrets & `.env` Files

**Managed by CLI (recommended):**
`p4n4 init` generates a **single project-level `.env`** that is shared by all stacks via the
`--env-file` flag in every `docker compose` call. The CLI owns the full secret lifecycle.

```
my-project/
├── .env                        ← one file, all secrets, never committed
├── .env.example                ← committed; union of all stack .env.example files
├── docker-compose.iot.yml      ← generated
├── docker-compose.ai.yml       ← generated
└── docker-compose.edge.yml     ← generated
```

**Standalone (without CLI):**
Each stack repo ships its own `.env.example`. Copy to `.env` and fill in values manually.
Cross-stack secrets (e.g. `INFLUXDB_ADMIN_TOKEN` used by both `p4n4-iot` and `p4n4-ai`) must be
kept consistent manually — document this prominently in each stack's README.

### 5.3 Shared Secret Reference

| Variable | Owner stack | Consumed by |
|---|---|---|
| `MQTT_USER` / `MQTT_PASSWORD` | `p4n4-iot` | Node-RED |
| `INFLUXDB_ADMIN_TOKEN` | `p4n4-iot` | Node-RED, Grafana, n8n |
| `INFLUXDB_ORG` / `INFLUXDB_BUCKET` | `p4n4-iot` | Node-RED, Grafana, n8n |
| `GF_SECURITY_ADMIN_PASSWORD` | `p4n4-iot` | Grafana |
| `N8N_BASIC_AUTH_USER` / `_PASSWORD` | `p4n4-ai` | n8n |
| `N8N_ENCRYPTION_KEY` | `p4n4-ai` | n8n |
| `LETTA_SERVER_PASSWORD` | `p4n4-ai` | Letta, Node-RED |
| `EI_API_KEY` | `p4n4-edge` | Edge Impulse containers (optional) |

---

## 6. Versioning & Release Strategy

### 6.1 Per-Repo Semver

Each repository is versioned independently with **`MAJOR.MINOR.PATCH`** Git tags.

| Repo | Version bumps when… |
|---|---|
| `p4n4-iot` | compose/config changes, new service added, breaking config change |
| `p4n4-ai` | same as above for AI services |
| `p4n4-edge` | same for Edge Impulse services |
| `p4n4-cli` | new CLI command, flag changes, template schema change, bug fixes |
| `p4n4-templates` | new example template, index entry added/changed, guide updated |
| `p4n4-docs` | no formal releases; `main` is always current |

### 6.2 CLI ↔ Stack Template Compatibility

The CLI bundles its own Jinja2 templates for scaffolding. To track which bundled template
version is compatible with which stack config version, a compatibility table is maintained
in `p4n4-cli/p4n4/compat.py`:

```python
STACK_COMPAT = {
    "iot":  ">=0.1.0",
    "ai":   ">=0.1.0",
    "edge": ">=0.1.0",
}
```

When a breaking change is made to a stack repo, the CLI's bundled templates are updated in the
same PR and the `STACK_COMPAT` table is bumped. The CLI warns if the running stack version is
outside the compatible range (detected via a `VERSION` file in each stack repo).

### 6.3 Release Flow (`p4n4-cli`)

```
1. Bump version in p4n4/cli.py __version__ and pyproject.toml
2. Update CHANGELOG.md
3. Open PR → CI passes → merge to main
4. Tag: git tag v0.2.0 && git push origin v0.2.0
5. publish.yml workflow triggers → builds wheel → uploads to PyPI
```

---

## 7. CI/CD per Repository

### `p4n4-iot`, `p4n4-ai`, `p4n4-edge`

```yaml
# .github/workflows/ci.yml
on: [push, pull_request]
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Validate docker-compose
        run: docker compose -f docker-compose.yml config --quiet
      - name: Lint YAML/JSON configs
        run: |
          pip install yamllint
          yamllint .
      - name: Check .env.example completeness
        run: python scripts/check_env_example.py
```

### `p4n4-cli`

```yaml
# ci.yml
on: [push, pull_request]
jobs:
  test:
    strategy:
      matrix:
        python-version: ["3.11", "3.12", "3.13"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "${{ matrix.python-version }}" }
      - run: pip install -e ".[dev]"
      - run: ruff check .
      - run: pytest tests/ -v --cov=p4n4

# publish.yml
on:
  push:
    tags: ["v*"]
jobs:
  publish:
    steps:
      - uses: actions/checkout@v4
      - run: pip install build
      - run: python -m build
      - uses: pypa/gh-action-pypi-publish@release/v1
        with:
          password: ${{ secrets.PYPI_API_TOKEN }}
```

### `p4n4-templates`

```yaml
# validate-index.yml
on: [push, pull_request]
jobs:
  validate:
    steps:
      - uses: actions/checkout@v4
      - name: Validate index.json schema
        run: python scripts/validate_index.py index.json
      - name: Validate all example p4n4-template.toml files
        run: python scripts/validate_templates.py examples/
```

### `p4n4-docs`

```yaml
# deploy-docs.yml
on:
  push:
    branches: [main]
jobs:
  deploy:
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
      - run: pip install mkdocs-material
      - run: mkdocs gh-deploy --force
```

---

## 8. Bootstrap Order

Follow this order when scaffolding the repositories from scratch:

```
Step 1 — Create GitHub org
  → Create raisga on GitHub
  → Set org profile, billing, team permissions

Step 2 — Scaffold .github (org profile)
  → Create raisga/.github repo
  → Add profile/README.md, shared issue templates

Step 3 — Scaffold p4n4 (umbrella)
  → README, CONTRIBUTING, SECURITY, CODE_OF_CONDUCT
  → docs/adr/ directory with ADR-001 (this architecture decision)

Step 4 — Scaffold p4n4-docs (this repo)
  → Move/copy README.md (full technical reference)
  → Add ARCHITECTURE.md (this file)
  → Wire up mkdocs.yml + deploy workflow (stub, activate in Phase 2)

Step 5 — Scaffold p4n4-iot
  → docker-compose.yml with p4n4-net definition
  → All service configs with sane defaults
  → .env.example
  → CI workflow

Step 6 — Scaffold p4n4-ai
  → docker-compose.yml with external p4n4-net
  → n8n workflows (starter JSON exports)
  → .env.example
  → CI workflow

Step 7 — Scaffold p4n4-edge
  → docker-compose.yml with external p4n4-net
  → edge-impulse/models/.gitkeep
  → .env.example
  → CI workflow

Step 8 — Scaffold p4n4-cli
  → pyproject.toml + package skeleton
  → Copy Jinja2 templates from reference configs in steps 5–7
  → CI + publish workflows
  → Stub tests

Step 9 — Scaffold p4n4-templates
  → index.json with two starter entries
  → examples/factory-baseline/ and examples/iot-minimal/
  → TEMPLATE_GUIDE.md
  → index validation CI
```

---

## 9. GitHub Organisation Setup

### Labels (applied across all repos)

| Label | Colour | Use |
|---|---|---|
| `stack: iot` | `#e07b39` | Relates to p4n4-iot |
| `stack: ai` | `#7b39e0` | Relates to p4n4-ai |
| `stack: edge` | `#39b0e0` | Relates to p4n4-edge |
| `stack: cli` | `#3de05e` | Relates to p4n4-cli |
| `type: bug` | `#d73a4a` | Something is broken |
| `type: feature` | `#0075ca` | New feature request |
| `type: docs` | `#cfd3d7` | Documentation change |
| `type: security` | `#b60205` | Security issue |
| `good first issue` | `#7057ff` | Good for newcomers |

### Branch Protection (all repos)

- Require PRs for `main`
- Require at least 1 review
- Require status checks to pass (CI)
- No force-push to `main`

### Repo Topics (for discoverability)

Apply to all stack repos: `iot`, `docker-compose`, `docker`, `self-hosted`
- `p4n4-iot` also: `mosquitto`, `node-red`, `influxdb`, `grafana`
- `p4n4-ai` also: `ollama`, `letta`, `n8n`, `llm`, `genai`
- `p4n4-edge` also: `edge-impulse`, `tinyml`, `edge-ai`
- `p4n4-cli` also: `cli`, `python`, `typer`, `scaffolding`

---

## 10. Future Extractions (Phase 3+)

These are **not** part of the initial scaffold. They are noted here to avoid architectural
decisions that would make them harder to do later.

### `p4n4-flows` (Phase 3)

Extract Node-RED flows out of `p4n4-iot` into a standalone community flows library.

```
p4n4-flows/
├── index.json                  ← flow registry (analogous to template index)
├── flows/
│   ├── telemetry-ingest/
│   ├── anomaly-detection/
│   ├── ai-digest/
│   └── device-command/
```

**Compatibility note:** `flows.json` in `p4n4-iot` stays as the default base; `p4n4-flows` is an
opt-in extension library. The CLI would gain a `p4n4 flows pull <name>` subcommand.

### Org Index Repos (Phase 3)

Teams using the template system with private templates will create their own index repos
(e.g. `acme/p4n4-index`). The `raisga/templates` repo is the community default.
No changes to the core repos are needed — the CLI already supports `p4n4 template org add`.

### Multi-Site Federation (Phase 3)

Requires no new repos. Implemented as a new set of n8n workflows and Node-RED flows deployed
via the existing `p4n4-ai` + `p4n4-iot` repos, plus a `multi-site` example in `p4n4-templates`.

---

*Document maintained in `raisga/p4n4-docs`. Open issues or PRs there for changes.*
