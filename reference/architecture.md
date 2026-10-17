# p4n4 — Multi-Repository Architecture

> Reference document for scaffolding the p4n4 GitHub organisation and its constituent repositories.
> Version 0.4 (p4n4 CLI 0.4.0) | 2026

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

p4n4 is organised across **15 repositories** under the `raisga` GitHub organisation (plus one under
`jraleman`). Each repository has a single, well-defined responsibility and can be developed,
versioned, and released independently. The `p4n4` umbrella repo is a **monorepo** that aggregates
the platform's sub-repos as Git submodules, grouped by concern. The websites (`p4n4-blog` and
`p4n4.com`) are submodules under `web/`, and the private `p4n4-sandbox` is one under
`projects/sandbox`. All three are set to `update = none`, so a recursive clone skips them.

```
raisga/
├── .github                 ← org profile, shared issue templates, org-wide Actions workflows
└── p4n4                    ← umbrella (monorepo with submodules)
    ├── stacks/
    │   ├── iot             ← p4n4-iot: Mosquitto · Node-RED · InfluxDB · Grafana
    │   ├── ai              ← p4n4-ai: Ollama · Letta · n8n · System 1 · inference runner (Edge Impulse · ONNX)
    │   └── extras          ← p4n4-extras: optional services, none by default (formerly p4n4-edge)
    ├── lib                 ← p4n4-lib: shared library (stacks ↔ clients)
    ├── cli                 ← p4n4-cli: Python CLI (published to PyPI as `p4n4`)
    ├── api                 ← p4n4-api: REST API gateway
    ├── dashboard           ← p4n4-dashboard: Flutter dashboard app
    ├── tools/
    │   ├── hw              ← p4n4-hw: hardware designs + RPi5 GPIO scripts
    │   ├── templates       ← p4n4-templates: community template registry & index
    │   └── emu             ← p4n4-emu: workstation hardware emulator
    ├── docs                ← p4n4-docs: full technical documentation site (this repo)
    ├── projects/
    │   └── sandbox         ← p4n4-sandbox: sandbox p4n4 projects for local testing (private)
    └── web/
        ├── blog            ← raisga/p4n4-blog: project blog
        └── p4n4.com        ← jraleman/p4n4.com: public-facing website (private)
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
| `p4n4-ai` | `stacks/ai` | stack | ✓ | — | Docker Compose AI stack: GenAI and the edge inference runner (Edge Impulse / ONNX, profile `ei-runner`); attaches to `p4n4-net` |
| `p4n4-extras` | `stacks/extras` | stack | ✓ | — | Optional services outside the core, one directory per extra, none enabled by default; attaches to `p4n4-net`. Formerly `p4n4-edge` ([ADR-005](../decisions/adr/ADR-005.md)) |
| `p4n4-lib` | `lib` | library | ✓ | — | Shared library mediating between stacks and clients |
| `p4n4-hw` | `tools/hw` | hardware | — | — | KiCad PCB designs and RPi5 GPIO scripts |
| `p4n4-templates` | `tools/templates` | registry | — | Git-native | Community template index + example templates |
| `p4n4-cli` | `cli` | tool | ✓ | `p4n4` on PyPI | Python CLI for scaffolding and lifecycle management |
| `p4n4-api` | `api` | service | ✓ | — | REST API gateway (port 8000); FastAPI on `p4n4-lib`, see [REST API](api.md) |
| `p4n4-dashboard` | `dashboard` | frontend | ✓ | — | Flutter web service (`dashboard` layer, port 8088) and desktop/mobile apps, white-label; see [Dashboard](dashboard.md), [ADR-003](../decisions/adr/ADR-003.md) |
| `p4n4-emu` | `tools/emu` | tool | ✓ | — | Workstation hardware emulator (Docker resource constraints + QEMU) |
| `p4n4-docs` | `docs` | docs | — | — | Full technical reference; deployable as a static site |
| `p4n4.com` | `web/p4n4.com` | website | — | — | Public-facing website (jraleman/p4n4.com, private); submodule with `update = none` |
| `p4n4-blog` | `web/blog` | website | — | — | Project blog; submodule with `update = none` |
| `p4n4-sandbox` | `projects/sandbox` | projects | — | — | Sandbox p4n4 projects for local testing (private); submodule with `update = none` |

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
├── CONTRIBUTING.md
├── CODE_OF_CONDUCT.md
├── LICENSE
├── .github/                    ← CODEOWNERS, issue and PR templates, stale bot
├── scripts/                    ← run p4n4 from source (see the Development guide)
│   ├── dev                     ← editable venv, `p4n4`, API and hot-reload dashboard
│   ├── dev_api.py              ← p4n4-api with auto-reload, used by `scripts/dev`
│   ├── doctor                  ← read-only checks: tools, submodules, venv, ports
│   └── repos                   ← every submodule's branch, changes and pointer
├── stacks/
│   ├── iot/                    ← submodule: p4n4-iot
│   ├── ai/                     ← submodule: p4n4-ai
│   └── extras/                 ← submodule: p4n4-extras (formerly p4n4-edge)
├── lib/                        ← submodule: p4n4-lib
├── cli/                        ← submodule: p4n4-cli
├── api/                        ← submodule: p4n4-api
├── dashboard/                  ← submodule: p4n4-dashboard
├── tools/
│   ├── hw/                     ← submodule: p4n4-hw
│   ├── templates/              ← submodule: p4n4-templates
│   └── emu/                    ← submodule: p4n4-emu
├── docs/                       ← submodule: p4n4-docs (ARCHITECTURE.md lives here)
├── projects/
│   └── sandbox/                ← submodule: p4n4-sandbox (private, update = none)
└── web/
    ├── blog/                   ← submodule: p4n4-blog (update = none)
    └── p4n4.com/               ← submodule: p4n4.com (private, update = none)
```

Release notes live in each repository's `release-notes/` folder (see
[Releasing](../project/releasing.md)), and security reports go through a
[private security advisory](https://github.com/raisga/p4n4/security/advisories/new) (see the
[Security guide](../guides/security.md#reporting-vulnerabilities)).

---

### 3.3 `p4n4-iot` (IoT stack)

**Owns** the `p4n4-net` Docker bridge network. All other stacks attach to it as an external network.

> **Note:** The pre-existing `p4n4-iot` repo uses the network name `p4n4-net`. The `p4n4-ai` and `p4n4-extras` stacks reference `p4n4-net` as an external network. When running all stacks together, ensure network names are aligned (the CLI handles this automatically via `p4n4 up`).

```
p4n4-iot/
├── docker-compose.yml          ← primary compose file (services + network definition)
├── docker-compose.override.yml.example ← local overrides to copy (bind mounts, debug ports)
├── .env.example                ← standalone .env template
├── .gitignore
├── Makefile                    ← convenience targets: up, down, status, test-mqtt
├── LICENSE
├── README.md
│
├── config/                     ← all service config files under a single top-level dir
│   ├── mosquitto/
│   │   ├── mosquitto.conf      ← persistence, logging; includes the generated conf.d
│   │   ├── entrypoint.sh       ← listeners, device logins, TLS and bridge from .env
│   │   ├── passwd.example, acl.example
│   │   └── certs/
│   │
│   ├── node-red/
│   │   ├── flows/
│   │   │   └── flows.json      ← version-controlled base flows (IoT only)
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
│   ├── init-buckets.sh
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

### 3.4 `p4n4-ai` (AI stack)

Attaches to `p4n4-net` as an external network (must be created by `p4n4-iot` first, or via `p4n4 up`, which starts `p4n4-iot` first).

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
├── n8n/
│   └── workflows/
│       ├── alert-enrichment.json
│       ├── scheduled-digest.json
│       ├── device-onboarding.json
│       └── incident-escalation.json
│
├── runner/                     ← edge inference runner (service and profile `ei-runner`)
│   ├── Dockerfile              ← python:3.11-slim
│   ├── runner.py               ← MQTT → model (.eim / .onnx / mock) → MQTT + InfluxDB; HTTP API
│   └── requirements.txt
├── edge-impulse/models/        ← .eim binaries are NEVER committed; provided at deploy time
├── onnx/models/                ← .onnx files are NEVER committed
└── tests/                      ← the runner's tests
```

The runner was the `p4n4-edge` stack until it merged into this one
([ADR-005](../decisions/adr/ADR-005.md)); see [Edge Inference](../stacks/edge-inference.md).

**Network block in `docker-compose.yml`:**
```yaml
networks:
  p4n4-net:
    external: true
    name: p4n4-net
```

---

### 3.5 `p4n4-extras` (optional services)

Formerly `p4n4-edge`. Optional services outside the core: none enabled by default, none
depended on by `p4n4-iot` or `p4n4-ai`. Each extra is a directory of its own, which the
layer's `docker-compose.yml` pulls in with `include:`, so all extras run as one Compose
project with one `.env`, each in a profile of its own name. See [Extras](../stacks/extras.md).

```
p4n4-extras/
├── docker-compose.yml          ← `include:` one line per extra (none yet); external p4n4-net
├── .env.example                ← COMPOSE_PROFILES= (empty), every extra's variables
├── Makefile                    ← make up (skips when nothing is enabled), make config
├── README.md                   ← core vs extras, adding an extra
├── _template/                  ← the skeleton extra to copy to <name>/
│   ├── docker-compose.yml
│   ├── .env.example
│   ├── config/
│   └── README.md
└── scripts/
    └── check_env_example.py    ← every ${VAR} in every included file is in .env.example
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

The `pip install p4n4` package. A thin Typer/Rich presentation layer: all shared,
framework-free logic lives in `p4n4-lib` (see the module table below). Scaffolding
fetches stack files from the canonical stack repos at init time (`git clone --depth 1`),
or from a local checkout via `--source-iot` / `--source-ai` / `--source-extras` for offline use.

```
p4n4-cli/
├── pyproject.toml              ← build system, deps, version, PyPI metadata
├── README.md
├── CHANGELOG.md
├── .github/
│   └── workflows/
│       ├── ci.yml              ← lint + test on every PR
│       └── publish.yml         ← publish to PyPI on release
│
├── p4n4/
│   ├── __init__.py             ← package version
│   ├── cli.py                  ← Typer app entrypoint; command registration
│   ├── project.py              ← locate the current project's manifest / compose dirs
│   │
│   └── commands/
│       ├── __init__.py
│       ├── init.py             ← `p4n4 init` interactive wizard
│       ├── add.py              ← `p4n4 add`
│       ├── remove.py           ← `p4n4 remove`
│       ├── lifecycle.py        ← `p4n4 up / down / status / logs`
│       ├── ei.py               ← `p4n4 ei` subcommands
│       ├── secret.py           ← `p4n4 secret`
│       ├── validate.py         ← `p4n4 validate`
│       ├── upgrade.py          ← `p4n4 upgrade`
│       └── template.py         ← `p4n4 template` subcommands
│
└── tests/
    ├── conftest.py
    └── test_cli.py
```

**Shared library (`p4n4-lib`, imported as `p4n4_lib`):**

| Module | Purpose |
|--------|---------|
| `p4n4_lib.manifest` | `.p4n4.json` read / write / find (walk-up) |
| `p4n4_lib.env` | Dotenv read/write, template-preserving writes |
| `p4n4_lib.layers` | Layer registry: repo URLs, copy paths, required files/env keys |
| `p4n4_lib.layout` | Project layout: flat (single-layer) vs per-layer subdirectories |
| `p4n4_lib.scaffold` | Fetch stack sources and copy them into a project |
| `p4n4_lib.validate` | Pure project validation returning (passed, errors) |
| `p4n4_lib.secrets` | Token generation and rotatable-key policy |
| `p4n4_lib.compose` | Docker Compose subprocess wrappers |

**Generated project layout:** single-layer projects keep stack files at the project
root. Multi-layer projects give each layer its own subdirectory (`<project>/iot/`,
`<project>/ai/`) with its own `docker-compose.yml`, `config/`, `scripts/`, and `.env`,
so the stacks run as separate Compose projects sharing the `p4n4-net` network
(see [§5.2](#52-secrets--env-files)).

**`pyproject.toml` key fields:**
```toml
[project]
name = "p4n4"
requires-python = ">=3.11"
dependencies = [
  "p4n4-lib>=0.4.0",
  "typer[all]>=0.12,<0.28",
  "rich>=13",
  "questionary>=2",
  "jinja2>=3",
]

[project.scripts]
p4n4 = "p4n4.cli:app"

[project.urls]
Homepage = "https://github.com/raisga/p4n4-cli"
Documentation = "https://github.com/raisga/p4n4-docs"
```

---

### 3.7 `p4n4-templates` (community template registry)

Git-native, with no server. Each template is a runnable project of its own; `p4n4 template`
commands are planned, so today a template is copied as the project root.

```
p4n4-templates/
├── README.md                   ← catalog, using templates, contributing
├── docs/
│   ├── authoring.md            ← how to write a template
│   ├── roadmap.md              ← the demo template series
│   └── use-cases/              ← end-to-end walkthroughs
├── schema/
│   ├── template.schema.json    ← template.yaml schema
│   └── theme.schema.json       ← dashboard theme (brand.json) schema
├── scripts/
│   └── validate.py             ← CI: schema, layers vs .p4n4.json, services, .env.example, dashboard block
├── .github/workflows/ci.yml    ← validate, then each template's smoke test
└── projects/
    ├── mqtt-influx-grafana/    ← single-layer template: files at its root
    │   ├── template.yaml       ← metadata (layers, services, ports, data contract)
    │   ├── .p4n4.json
    │   ├── docker-compose.yml · .env.example · config/
    │   ├── theme/              ← optional p4n4-dashboard theme
    │   └── tests/smoke.sh
    └── mqtt-influx-grafana-ollama-go2rtc/   ← multi-layer template: iot/, ai/, extras/
```

---

### 3.8 `p4n4-docs` (documentation site)

Markdown, read on GitHub; a MkDocs site is planned (no site config or CI yet).

```
p4n4-docs/
├── README.md                   ← overview and index
├── getting-started.md
├── stacks/                     ← iot-stack, ai-stack, edge-inference, extras, dashboard
├── reference/                  ← architecture (this file), cli-reference, api, dashboard,
│                                  configuration, mqtt-topics, template-registry, hardware, emulator
├── guides/                     ← deployment, development, security, troubleshooting,
│                                  migrating-from-edge, raspberry-pi, jetson, aws, gcp, azure, digitalocean
├── decisions/                  ← design, specs, design drafts, adr/ (ADR-001 … ADR-005)
└── project/                    ← releasing
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
├── TODO.md                     ← remaining work, by priority
├── pyproject.toml
├── uv.lock
├── LICENSE
├── scripts/install-binfmt.sh   ← QEMU binfmt registration
├── p4n4_emu/
│   ├── cli.py                  ← Typer entrypoint (p4n4-emu command)
│   ├── commands/               ← setup, up, down, status, logs, profile, sim
│   ├── profiles/               ← hardware profile definitions (rpi4, rpi5, nuc, mcu-class)
│   ├── overlays/
│   │   ├── generator.py        ← per-service limits (deploy.resources.limits, memswap, blkio)
│   │   ├── paths.py            ← one overlay folder per stack directory; the profile it records
│   │   └── templates/          ← stack.emu.yml.j2
│   ├── utils/
│   │   ├── compose.py          ← docker compose wrapper (project files + overlay)
│   │   ├── project.py          ← .p4n4.json discovery, flat and multi-layer layouts
│   │   ├── usage.py            ← live usage (docker stats) vs. applied limits (docker inspect)
│   │   ├── preflight.py        ← Docker, Compose, cgroup v2 and QEMU checks
│   │   └── docker_info.py      ← block device behind Docker's data root
│   ├── sim/
│   │   └── sensor_sim.py       ← synthetic MQTT sensor publisher (Docker image)
│   └── hw/                     ← `p4n4-emu run`: the emulated board a script sees
│       ├── pins.py, buses.py   ← the GPIO lines and the I2C / SPI / UART buses all stubs share
│       ├── gpio_stub.py, lgpio_stub.py, gpiod_stub/   ← RPi.GPIO, lgpio (gpiozero), gpiod
│       ├── smbus_stub.py, spidev_stub.py, serial_stub.py
│       ├── devices/            ← BME280, MPU-6050, ADS1115, MCP3008, DS18B20 register models
│       ├── vfs.py              ← /proc/device-tree, 1-Wire sysfs and /dev nodes
│       ├── mqtt_bridge.py      ← emu/gpio/<pin>/set and /state
│       └── shims.py            ← installs it all (from site/sitecustomize.py under `run`)
└── tests/
```

`up` writes each stack's overlay to `~/.p4n4-emu/overlays/<stack-dir>-<hash>/<stack>.emu.yml`
and `down` deletes it. The overlay records its profile in an `x-p4n4-emu` block, so `down`,
`status` and `logs` need no `--profile`. See the [Emulator reference](emulator.md).

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
   │  p4n4-iot   │   │   p4n4-ai    │   │ p4n4-extras │
   │ (owns net)  │   │ (ext: net)   │   │ (ext: net)  │
   └──────┬──────┘   └──────────────┘   └─────────────┘
          │ p4n4-net (bridge, created by p4n4-iot)
          │ ← p4n4-ai/extras attach via p4n4-net (see §5.1)
          │
   ┌──────▼──────────────────────────────────────────┐
   │           p4n4-cli (through p4n4-lib)            │
   │  clones each stack repo at `p4n4 init`           │
   │  (`template` commands planned)                   │
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
3. p4n4-extras → attaches to p4n4-net; optional, skipped while nothing is enabled
4. dashboard   → attaches to p4n4-net; proxies p4n4-api (and optionally Grafana), so it starts last
```

Running `p4n4 up` with no stack argument enforces this order: it starts each stack's
Compose project in sequence (`iot → ai → extras → dashboard`); `p4n4 down` stops them in reverse.
A project from before ADR-005 may still have an `edge` layer: it runs right after `ai` until
`p4n4 migrate edge` moves it.

---

## 5. Network & Secrets Strategy

### 5.1 Docker Network

- The pre-existing `p4n4-iot` creates a network named **`p4n4-net`**.
- `p4n4-ai` and `p4n4-extras` declare **`p4n4-net`** as `external: true`.
- When running all stacks together, the CLI (`p4n4 up`) starts the stacks in dependency
  order (`iot` first), so the IoT compose project creates `p4n4-net` before the other
  stacks attach to it. Manual deployments must ensure the network name is consistent
  across stacks.
- Because the stacks define `p4n4-net` differently (owner vs `external: true`), their
  compose files must **not** be merged into one Compose project — this is why multi-layer
  projects scaffold each stack into its own directory (see 5.2).
- When stacks are used standalone (without the IoT stack), they create their own default network.

### 5.2 Secrets & `.env` Files

**Managed by CLI (recommended):**
`p4n4 init` scaffolds each enabled stack with its own `.env`, generated with
cryptographically secure secrets. Single-layer projects keep everything at the project
root; multi-layer projects give each stack its own subdirectory and `.env`, and every
`docker compose` call runs inside that stack's directory.

```
my-project/                     ← multi-layer layout (`--layer iot,ai`)
├── .p4n4.json                  ← manifest: project name, enabled layers
├── iot/
│   ├── docker-compose.yml      ← copied from p4n4-iot
│   ├── .env                    ← iot secrets, never committed
│   ├── config/ …
│   └── scripts/ …
└── ai/
    ├── docker-compose.yml      ← copied from p4n4-ai
    ├── .env                    ← ai secrets + shared InfluxDB values
    ├── config/ …
    └── scripts/ …
```

Cross-stack values (`INFLUXDB_TOKEN`, `INFLUXDB_ORG`, `INFLUXDB_BUCKET`) are written
identically to every stack's `.env` at init time. `p4n4 secret rotate` writes one new value
per key to every `.env` containing it, but only for the secrets services read at every start
(`NODE_RED_PASSWORD`, `LETTA_SERVER_PASSWORD`); the shared InfluxDB token is setup-only and
changed in InfluxDB itself (see the [Security guide](../guides/security.md#secret-rotation)).

**Standalone (without CLI):**
Each stack repo ships its own `.env.example`. Copy to `.env` and fill in values manually.
Cross-stack secrets (e.g. `INFLUXDB_TOKEN` used by `p4n4-iot` and `p4n4-ai`) must be
kept consistent manually — document this prominently in each stack's README.

### 5.3 Shared Secret Reference

| Variable | Owner stack | Consumed by |
|---|---|---|
| `MQTT_USER` / `MQTT_PASSWORD` | `p4n4-iot` | Node-RED, edge runner |
| `INFLUXDB_TOKEN` | `p4n4-iot` | Node-RED, Grafana, n8n, edge runner |
| `INFLUXDB_ORG` / `INFLUXDB_BUCKET` | `p4n4-iot` | Node-RED, Grafana, n8n |
| `GRAFANA_USER` / `GRAFANA_PASSWORD` | `p4n4-iot` | Grafana |
| `NODE_RED_USER` / `NODE_RED_PASSWORD` | `p4n4-iot` | Node-RED editor login |
| `N8N_BASIC_AUTH_USER` / `_PASSWORD` | `p4n4-ai` | n8n |
| `N8N_ENCRYPTION_KEY` | `p4n4-ai` | n8n |
| `LETTA_SERVER_PASSWORD` | `p4n4-ai` | Letta, Node-RED |
| `EI_API_KEY` | `p4n4-ai` | edge runner (optional) |

---

## 6. Versioning & Release Strategy

### 6.1 Per-Repo Semver

Each repository is versioned independently with **`MAJOR.MINOR.PATCH`** Git tags.

| Repo | Version bumps when… |
|---|---|
| `p4n4-iot` | compose/config changes, new service added, breaking config change |
| `p4n4-ai` | same as above for AI services, the edge inference runner included |
| `p4n4-extras` | an extra added, changed or removed |
| `p4n4-cli` | new CLI command, flag changes, template schema change, bug fixes |
| `p4n4-templates` | new example template, index entry added/changed, guide updated |
| `p4n4-docs` | no formal releases; `main` is always current |

### 6.2 CLI ↔ Stack Compatibility

There is no runtime version check, `compat.py` or stack `VERSION` file. `p4n4 init` copies
each layer from its stack repo's `main` (or a local checkout with `--source-<layer>`), and
p4n4-lib's layer registry (`layers.py`, `sources.yaml`) names the repos, the files a project
copies and the files and `.env` keys it must have. So a stack change that moves or requires
files ships with a p4n4-lib release that knows about it: stacks pushed first, then p4n4-lib,
then the CLI, p4n4-api and p4n4-emu that require it (`p4n4-lib>=0.4.0`); see
[releasing](../project/releasing.md). A layer the registry drops keeps running as a legacy
layer until `p4n4 migrate` moves it (`edge`, [ADR-005](../decisions/adr/ADR-005.md)).

### 6.3 Release Flow (`p4n4-cli`)

```
1. Bump the version in pyproject.toml and p4n4/__init__.py
2. Update CHANGELOG.md and write release-notes/v<version>.md
3. Open PR → CI passes → merge to main
4. gh release create v<version> --notes-file release-notes/v<version>.md
5. publish.yml (on a published release) → builds wheel → uploads to PyPI
```

---

## 7. CI/CD per Repository

### `p4n4-iot`, `p4n4-ai`, `p4n4-extras`

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
# ci.yml
on: [push, pull_request]
jobs:
  validate:
    steps:
      - uses: actions/checkout@v4
      - run: uv run scripts/validate.py          # every template
      - run: echo "templates=$(uv run scripts/validate.py --list)" >> "$GITHUB_OUTPUT"
  smoke:                                         # one job per template
    needs: validate
    steps:
      - run: ./tests/smoke.sh                    # in an isolated Compose project
```

### `p4n4-docs`

No CI yet. Once the MkDocs site exists, a push to `main` will run `mkdocs gh-deploy`.

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
  → docs/decisions/adr/ directory with ADR-001 (this architecture decision)

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

Step 7 — Scaffold p4n4-extras (formerly p4n4-edge; its runner is now in p4n4-ai)
  → docker-compose.yml with external p4n4-net and an empty include:
  → _template/ (the skeleton extra)
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
| `stack: extras` | `#39b0e0` | Relates to p4n4-extras (was `stack: edge`) |
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
- `p4n4-ai` also: `ollama`, `letta`, `n8n`, `llm`, `genai`, `edge-impulse`, `tinyml`, `edge-ai`
- `p4n4-extras` also: `extras`, `addons`
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

*Document maintained in `raisga/p4n4-docs` (monorepo path: `docs`). Open issues or PRs there for changes.*
