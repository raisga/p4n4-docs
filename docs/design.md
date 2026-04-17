# p4n4 — Design Document

**Version:** 0.1  
**Date:** 2026-04-16  
**Status:** Living document

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Goals and Non-Goals](#2-goals-and-non-goals)
3. [System Architecture](#3-system-architecture)
4. [Component Design](#4-component-design)
   - 4.1 [IoT Stack (p4n4-iot)](#41-iot-stack-p4n4-iot)
   - 4.2 [GenAI Stack (p4n4-ai)](#42-genai-stack-p4n4-ai)
   - 4.3 [Edge AI Stack (p4n4-edge)](#43-edge-ai-stack-p4n4-edge)
   - 4.4 [CLI (p4n4-cli)](#44-cli-p4n4-cli)
   - 4.5 [Template Registry (p4n4-templates)](#45-template-registry-p4n4-templates)
   - 4.6 [Documentation Site (p4n4-docs)](#46-documentation-site-p4n4-docs)
5. [Data Flow](#5-data-flow)
6. [Network Design](#6-network-design)
7. [Secret Management](#7-secret-management)
8. [Security Model](#8-security-model)
9. [Repository Strategy](#9-repository-strategy)
10. [Versioning and Compatibility](#10-versioning-and-compatibility)
11. [CI/CD Strategy](#11-cicd-strategy)
12. [Deployment Model](#12-deployment-model)
13. [Extension Points](#13-extension-points)
14. [Design Decisions and Trade-offs](#14-design-decisions-and-trade-offs)
15. [Future Roadmap](#15-future-roadmap)

---

## 1. Project Overview

**p4n4** is a self-hosted, open-source platform for building end-to-end IoT pipelines with local AI inference. It assembles best-of-breed open-source services into three Docker Compose stacks that can be deployed independently or together:

| Stack | Purpose | Core Services |
|-------|---------|---------------|
| **MING** (IoT) | Ingest, store, and visualise sensor data | Mosquitto · Node-RED · InfluxDB · Grafana |
| **GenAI** | Local LLM inference, agent memory, workflow automation | Ollama · Letta · n8n |
| **Edge AI** | On-device model inference | Edge Impulse Linux Runner |

The primary user-facing product is the `p4n4` Python CLI (published to PyPI). It scaffolds projects, manages secrets, and orchestrates all stacks with a single command.

### Target users

- **Makers and hobbyists** — rapid IoT prototyping with full AI capabilities, no cloud dependency.
- **Industrial operators** — predictive maintenance and anomaly detection on private infrastructure.
- **Researchers** — local, reproducible AI + sensor data pipelines.
- **Self-hosters** — privacy-preserving alternatives to cloud IoT + AI services.

---

## 2. Goals and Non-Goals

### Goals

- **Self-hosted first** — no required cloud accounts, no data leaving the local network.
- **Composable** — each stack is independently deployable; users pick only what they need.
- **CLI-first UX** — one command (`p4n4 up`) to go from zero to a running platform.
- **Extensible via templates** — community templates provide domain-specific starting points.
- **Secure by design** — cryptographically generated secrets, no default plaintext passwords committed.
- **Low barrier** — runs on a Raspberry Pi 4 (IoT stack); full stack needs 8 GB RAM.

### Non-Goals

- **Kubernetes** — Phase 1 explicitly targets Docker Compose on single-host deployments.
- **Managed hosting** — p4n4 is self-hosted only; no SaaS offering is planned.
- **Real-time streaming at scale** — the IoT stack is optimised for moderate sensor volumes, not high-throughput industrial data lakes.
- **Multi-tenancy** — projects are single-tenant; each deployment is isolated at the host level.

---

## 3. System Architecture

```
  Sensors / Devices
        │ MQTT (port 1883 / WS 9001)
        ▼
  ┌─────────────────────────────────────────────────────────┐
  │  p4n4-iot  (MING stack)                                 │
  │                                                         │
  │  Mosquitto ──► Node-RED ──► InfluxDB                    │
  │                                  │                      │
  │                              Grafana                    │
  └─────────────────────────────────────────────────────────┘
           │ p4n4-net (Docker bridge 172.20.0.0/16)
           ├────────────────────────────────────┐
           ▼                                    ▼
  ┌────────────────────────┐        ┌──────────────────────┐
  │  p4n4-ai (GenAI stack) │        │ p4n4-edge (Edge AI)  │
  │  Ollama · Letta · n8n  │        │ Edge Impulse Runner  │
  └────────────────────────┘        └──────────────────────┘

  ┌───────────────────────────────────────────────────────┐
  │  Client layer                                         │
  │  p4n4-cli (Python, PyPI)  ·  p4n4-api (REST :8000)   │
  │  p4n4-dashboard (Web UI)                              │
  └───────────────────────────────────────────────────────┘
```

All three Docker stacks share a single Docker bridge network (`p4n4-net`). The IoT stack owns and creates this network; the AI and Edge stacks attach to it as an external network.

The client layer (CLI, REST API, dashboard) communicates with stack services over `localhost` ports and with each other via `p4n4-lib`, the shared Python library.

---

## 4. Component Design

### 4.1 IoT Stack (p4n4-iot)

The foundation of every p4n4 deployment. It **owns** `p4n4-net` — no other stack can start without it (when running together).

#### Services

| Service | Image | Port | Role |
|---------|-------|------|------|
| Mosquitto | `eclipse-mosquitto:2` | 1883 / 9001 | MQTT broker |
| InfluxDB | `influxdb:2` | 8086 | Time-series storage |
| Node-RED | `nodered/node-red:latest` | 1880 | Data routing and transformation |
| Grafana | `grafana/grafana:latest` | 3000 | Dashboarding and alerting |

#### Data path

```
Device → [MQTT publish] → Mosquitto → Node-RED → [write] → InfluxDB
                                                              │
                                                          Grafana (query)
```

Node-RED is the central routing hub. It subscribes to MQTT topics, applies transformations (unit conversion, tagging, deduplication), and writes structured records to InfluxDB. Grafana queries InfluxDB for visualisation.

#### Key design choices

- **Node-RED over custom code** — a visual, flow-based environment lowers the barrier for non-programmers to wire together IoT pipelines.
- **InfluxDB v2** — native time-series semantics (measurements, tags, fields), Flux query language, and a built-in UI for ad-hoc exploration.
- **Mosquitto** — lightweight, well-supported MQTT broker; supports MQTT 3.1.1 and 5.0, WebSocket transport, TLS, ACL files, and password authentication.

#### Configuration files

```
config/
├── mosquitto/
│   ├── mosquitto.conf    ← broker config (auth, TLS, persistence)
│   ├── passwd            ← generated by CLI; never committed
│   └── acl               ← topic-level access rules
├── node-red/
│   ├── flows.json        ← version-controlled base flows
│   └── settings.js
└── grafana/
    └── provisioning/
        ├── datasources/  ← InfluxDB datasource auto-provision
        └── dashboards/   ← pre-built IoT overview dashboard
```

---

### 4.2 GenAI Stack (p4n4-ai)

Provides local LLM inference, stateful AI agents, and event-driven workflow automation. Attaches to `p4n4-net` as an external network.

#### Services

| Service | Image | Port | Role |
|---------|-------|------|------|
| Ollama | `ollama/ollama:latest` | 11434 | Local LLM runtime (llama3, mistral, etc.) |
| Letta | `letta/letta:latest` | 8283 | AI agent framework with persistent memory |
| n8n | `n8nio/n8n:latest` | 5678 | Workflow automation and event orchestration |

#### Interaction model

```
InfluxDB ──► n8n (scheduled trigger) ──► Ollama (LLM summarise) ──► output channel
Mosquitto ──► n8n (MQTT trigger) ──► Letta (agent reasoning) ──► action
```

n8n is the orchestration layer. It triggers on schedules or events (MQTT messages, InfluxDB thresholds, webhooks), passes context to Ollama for natural-language generation, and routes results to notification channels or back into the system.

Letta provides stateful memory for AI agents — useful for long-running scenarios like "learn normal sensor patterns over time" or "track ongoing incidents".

#### Starter workflows (n8n)

| File | Trigger | Purpose |
|------|---------|---------|
| `alert-enrichment.json` | MQTT alert | Enrich with LLM-generated root-cause analysis |
| `scheduled-digest.json` | Cron | Daily telemetry summary in plain English |
| `device-onboarding.json` | MQTT new-device topic | Auto-register and tag new devices |
| `incident-escalation.json` | Threshold breach | Classify severity and route escalation |

#### GPU support

The `docker-compose.override.yml` contains a commented-out `deploy.resources` block for NVIDIA GPU passthrough to the Ollama container. Enabling it requires the NVIDIA Container Toolkit on the host.

---

### 4.3 Edge AI Stack (p4n4-edge)

Runs Edge Impulse model inference inside Docker. Fully independent — can operate with or without the IoT and AI stacks.

#### Services

| Service | Image | Port | Role |
|---------|-------|------|------|
| edge-impulse-runner | `edgeimpulse/linux-runner:latest` | 8080 | Runs `.eim` model inference |

#### Model lifecycle

```
1. Train model in Edge Impulse Studio (cloud)
2. Export as .eim Linux binary
3. p4n4 ei deploy model.eim   ← copies binary; restarts runner
4. Runner exposes inference API on :8080
5. Node-RED or n8n queries :8080 for predictions
```

`.eim` model binaries are never committed to the repository (`.gitignore`). They are provided at deploy time, either via the CLI (`p4n4 ei deploy`) or by manually copying into `edge-impulse/models/`.

#### Sensor / camera access

USB cameras and serial devices are exposed to the container via the `devices` block in `docker-compose.override.yml`. This is commented out by default to avoid device permission errors on hosts without the hardware attached.

---

### 4.4 CLI (p4n4-cli)

The primary user interface. Published to PyPI as `p4n4`. Bundled Jinja2 templates make offline scaffolding possible with no network calls.

#### Tech stack

| Component | Library |
|-----------|---------|
| CLI framework | [Typer](https://typer.tiangolo.com/) + [Rich](https://rich.readthedocs.io/) |
| Interactive prompts | [Questionary](https://questionary.readthedocs.io/) |
| Templating | [Jinja2](https://jinja.palletsprojects.com/) |
| YAML/TOML parsing | PyYAML, tomllib (stdlib 3.11+) |
| Secret generation | `secrets` module (stdlib) |

#### Package structure

```
p4n4/
├── cli.py           ← Typer app; command registration
├── commands/        ← one module per top-level command
├── scaffold/        ← manifest (.p4n4.json) + Jinja2 renderer
├── templates/       ← bundled Jinja2 templates (offline scaffold)
│   ├── iot/
│   ├── ai/
│   ├── edge/
│   └── shared/
└── utils/
    ├── docker.py    ← subprocess wrappers for docker/compose
    ├── secrets.py   ← cryptographic secret generation
    └── network.py   ← p4n4-net existence checks and creation
```

#### Command surface

| Command | Description |
|---------|-------------|
| `p4n4 init [PATH]` | Interactive wizard — scaffold stacks, generate secrets, write `.p4n4.json` |
| `p4n4 add STACK` | Add a stack to an existing project |
| `p4n4 remove STACK` | Remove a stack from an existing project |
| `p4n4 up [STACK]` | Start stacks in dependency order |
| `p4n4 down [STACK]` | Stop stacks |
| `p4n4 status` | Show container status for all stacks |
| `p4n4 logs STACK` | Tail logs for a stack |
| `p4n4 validate` | Validate manifest, compose files, `.env` presence |
| `p4n4 upgrade [STACK]` | Pull latest Docker images |
| `p4n4 secret show\|rotate\|generate` | Manage project secrets |
| `p4n4 ei deploy\|run\|status` | Edge Impulse model lifecycle |
| `p4n4 template search\|install\|list` | Community template registry |

#### Project manifest (.p4n4.json)

The manifest is the source of truth for a scaffolded project. It records which stacks are enabled, the project name, and CLI version used for scaffolding.

```json
{
  "name": "my-project",
  "version": "0.1.0",
  "cli_version": "0.1.0",
  "stacks": ["iot", "ai"],
  "created_at": "2026-04-16T00:00:00Z"
}
```

`p4n4 validate` checks that all referenced compose files exist, the `.env` is present, and all required variables are set.

---

### 4.5 Template Registry (p4n4-templates)

A Git-native community registry. No server required — the CLI fetches `index.json` directly from the GitHub repository.

#### Registry format

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
    }
  }
}
```

#### Template metadata (p4n4-template.toml)

Each template ships a `p4n4-template.toml` descriptor:

```toml
[template]
name        = "factory-baseline"
version     = "0.1.0"
description = "Full IoT + GenAI + EI stack for discrete manufacturing"
author      = "raisga"
tags        = ["manufacturing", "vibration", "edge-impulse"]

[requires]
cli    = ">=0.1.0"
stacks = ["iot", "ai", "edge"]
```

Template files are Jinja2 (`.j2`) using the same context variables as the built-in scaffold. Static JSON files (Grafana dashboards, n8n workflows) are copied verbatim.

#### Private / org indexes

Teams can host their own `index.json` in a private repository. The CLI supports adding alternative indexes via `p4n4 template org add <url>`, allowing enterprise templates to be distributed without publishing to the community registry.

---

### 4.6 Documentation Site (p4n4-docs)

Static site built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/). Deployed to GitHub Pages on every push to `main`.

Content structure:
- `docs/index.md` — platform overview
- `docs/getting-started.md` — installation and first project
- `docs/iot-stack.md` — IoT stack reference
- `docs/ai-stack.md` — AI stack reference
- `docs/edge-stack.md` — Edge stack reference
- `docs/cli-reference.md` — full CLI command reference
- `docs/template-registry.md` — using and contributing templates
- `docs/security.md` — security hardening guide
- `docs/adr/` — Architecture Decision Records

---

## 5. Data Flow

### Telemetry ingest path

```
Sensor
  │  publish("sensors/device-1/temperature", {"v": 22.5, "ts": 1700000000})
  ▼
Mosquitto (MQTT broker)
  │  internal routing
  ▼
Node-RED (subscribe all "sensors/#")
  │  transform: add metadata tags, validate schema, unit conversion
  ▼
InfluxDB (write API)
  │  measurement=temperature, tags={device="device-1"}, field=value, time=ts
  ▼
Grafana (Flux query on schedule)
  └── renders time-series panels
```

### AI analysis path (GenAI stack)

```
InfluxDB
  │  n8n polls on cron schedule
  ▼
n8n workflow (scheduled-digest)
  │  Flux query → JSON rows
  ▼
Ollama API (/api/generate)
  │  prompt: "Summarise these sensor readings: {rows}"
  ▼
LLM response (plain English digest)
  │  n8n routes to: webhook / email / Slack / MQTT publish
  ▼
Output channel
```

### Edge AI inference path

```
Camera / sensor device
  │  raw frames / audio / IMU data
  ▼
Edge Impulse Runner (edge-impulse-runner container)
  │  .eim model inference
  ▼
Inference result (HTTP :8080 /api/infer or MQTT publish)
  │
  ├──► Node-RED (trigger action flows)
  └──► n8n (trigger workflow on anomaly)
```

---

## 6. Network Design

### Docker network

The `p4n4-iot` stack creates `p4n4-net`:

```yaml
networks:
  p4n4-net:
    name: p4n4-net
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

`p4n4-ai` and `p4n4-edge` declare it as external:

```yaml
networks:
  p4n4-net:
    external: true
    name: p4n4-net
```

All services on `p4n4-net` can resolve each other by container name (e.g., `http://influxdb:8086`). No additional DNS configuration is needed.

### Port exposure

| Service | Host port | Container port | Stack |
|---------|-----------|----------------|-------|
| Mosquitto MQTT | 1883 | 1883 | iot |
| Mosquitto WebSocket | 9001 | 9001 | iot |
| InfluxDB | 8086 | 8086 | iot |
| Node-RED | 1880 | 1880 | iot |
| Grafana | 3000 | 3000 | iot |
| Ollama | 11434 | 11434 | ai |
| Letta | 8283 | 8283 | ai |
| n8n | 5678 | 5678 | ai |
| Edge Impulse Runner | 8080 | 8080 | edge |
| p4n4 REST API | 8000 | 8000 | api |

### Production hardening

- Place all services behind a reverse proxy (Nginx, Caddy, Traefik) with TLS termination.
- Use firewall rules to restrict 1883 and 8086 to the local network.
- Use Mosquitto TLS (`listener 8883`) for MQTT in production.

---

## 7. Secret Management

### CLI-managed (recommended)

`p4n4 init` generates a **single project-level `.env`** with cryptographically secure secrets for all stacks. All `docker compose` invocations use `--env-file .env`.

```
my-project/
├── .env              ← all secrets; in .gitignore
├── .env.example      ← placeholder values; committed
└── ...
```

Secrets are generated with Python's `secrets.token_hex(32)` for tokens and `secrets.token_urlsafe(24)` for passwords.

### Cross-stack shared secrets

| Variable | Owned by | Used by |
|----------|----------|---------|
| `INFLUXDB_ADMIN_TOKEN` | p4n4-iot | Node-RED, Grafana, n8n |
| `INFLUXDB_ORG` | p4n4-iot | Node-RED, Grafana, n8n |
| `INFLUXDB_BUCKET` | p4n4-iot | Node-RED, Grafana, n8n |
| `MQTT_USER` / `MQTT_PASSWORD` | p4n4-iot | Node-RED |
| `GF_SECURITY_ADMIN_PASSWORD` | p4n4-iot | Grafana |
| `N8N_BASIC_AUTH_USER` / `_PASSWORD` | p4n4-ai | n8n |
| `N8N_ENCRYPTION_KEY` | p4n4-ai | n8n |
| `LETTA_SERVER_PASSWORD` | p4n4-ai | Letta, Node-RED |
| `EI_API_KEY` | p4n4-edge | Edge Impulse Runner |

When using stacks without the CLI, these must be kept consistent manually across per-stack `.env` files.

---

## 8. Security Model

### Threat model (self-hosted, LAN deployment)

| Threat | Mitigation |
|--------|-----------|
| Default credentials exploited | CLI generates unique secrets per project; no global defaults committed |
| Secrets in version control | `.env` is in `.gitignore`; only `.env.example` (placeholders) is committed |
| Unauthenticated MQTT | Default `allow_anonymous true` for dev; disable for production + ACL |
| Exposed services on public IP | Bind services to localhost or use firewall; reverse proxy with TLS for WAN |
| Model binary tampering | `.eim` files are externally sourced; integrity verification is out of scope for Phase 1 |
| Compromised container | Docker network isolation limits blast radius; no host bind-mounts for sensitive paths |

### Development vs production posture

| Setting | Development (default) | Production (hardened) |
|---------|-----------------------|-----------------------|
| MQTT anonymous | `allow_anonymous true` | `allow_anonymous false` + passwd + ACL |
| Service binding | `0.0.0.0` | `127.0.0.1` or reverse proxy |
| TLS | Off | Reverse proxy terminates TLS |
| Secrets | Placeholder or CLI-generated | CLI-generated; rotated periodically |
| InfluxDB exposed | Yes (`:8086`) | Firewall-restricted or internal-only |

### Vulnerability disclosure

Vulnerabilities are reported privately via the process in [SECURITY.md](https://github.com/raisga/p4n4/blob/main/SECURITY.md). Public issues must not contain exploit details.

---

## 9. Repository Strategy

p4n4 is split across **8 repositories** under the `raisga` GitHub organisation (ADR-001). Each has a single, independently releasable responsibility.

| Repo | Type | PyPI | Description |
|------|------|------|-------------|
| `.github` | org meta | — | Org profile, shared workflows, issue templates |
| `p4n4` | umbrella | — | Landing page, ADRs, cross-cutting docs |
| `p4n4-iot` | stack | — | IoT Docker Compose stack |
| `p4n4-ai` | stack | — | GenAI Docker Compose stack |
| `p4n4-edge` | stack | — | Edge AI Docker Compose stack |
| `p4n4-cli` | tool | `p4n4` | Python CLI and scaffold engine |
| `p4n4-templates` | registry | — | Community template index |
| `p4n4-docs` | docs | — | MkDocs documentation site |

### Why multi-repo

- Stack repos can be version-tagged, forked, and deployed independently.
- PyPI publishing for the CLI only requires access to `p4n4-cli` — no monorepo tooling needed.
- Teams can contribute templates without repository-wide write access.
- Reduces blast radius of a compromised CI token.

### Cross-repo coordination

The CLI is the integration layer. When a stack repo has a breaking config change, the corresponding Jinja2 template in `p4n4-cli` is updated in the same PR and `STACK_COMPAT` is bumped. No runtime cross-repo calls are made; the CLI bundles all templates offline.

---

## 10. Versioning and Compatibility

### Per-repo semver

Each repository uses independent `MAJOR.MINOR.PATCH` Git tags. The `p4n4-cli` package on PyPI follows the same scheme.

### CLI ↔ stack compatibility

The CLI tracks compatible stack versions in `p4n4/compat.py`:

```python
STACK_COMPAT = {
    "iot":  ">=0.1.0",
    "ai":   ">=0.1.0",
    "edge": ">=0.1.0",
}
```

Each stack repo ships a `VERSION` file. On `p4n4 up`, the CLI reads this file and warns if the running stack version is outside the compatible range.

### Release flow (p4n4-cli)

```
1. Bump __version__ in p4n4/__init__.py and pyproject.toml
2. Update CHANGELOG.md
3. Open PR → CI passes (lint + test on Python 3.11/3.12/3.13)
4. Merge to main
5. git tag v0.2.0 && git push origin v0.2.0
6. publish.yml workflow → python -m build → PyPI upload
```

---

## 11. CI/CD Strategy

### Stack repos (p4n4-iot, p4n4-ai, p4n4-edge)

On every push and PR:

1. `docker compose config --quiet` — validate compose YAML is syntactically correct.
2. `yamllint .` — lint all YAML and JSON configuration files.
3. `python scripts/check_env_example.py` — verify `.env.example` lists all required variables.

### p4n4-cli

Matrix test across Python 3.11, 3.12, 3.13 on every PR:

1. `ruff check .` — lint.
2. `pytest tests/ -v --cov=p4n4` — unit tests with coverage.

On version tag push:

1. `python -m build` — build wheel and sdist.
2. `pypa/gh-action-pypi-publish` — upload to PyPI using `PYPI_API_TOKEN` secret.

### p4n4-templates

On every PR:

1. `python scripts/validate_index.py index.json` — validate `index.json` schema.
2. `python scripts/validate_templates.py examples/` — validate all `p4n4-template.toml` files.

### p4n4-docs

On push to `main`:

1. `mkdocs gh-deploy --force` — build MkDocs site and push to `gh-pages` branch.

---

## 12. Deployment Model

### CLI-managed (recommended path)

```bash
pip install p4n4
p4n4 init my-project     # interactive wizard
cd my-project
p4n4 up                  # starts: iot → ai → edge (in order)
```

The CLI enforces startup order via Docker healthcheck polling:

1. `p4n4-iot` — creates `p4n4-net`, waits for Mosquitto and InfluxDB to be healthy.
2. `p4n4-ai` — attaches to `p4n4-net`; starts after IoT healthy.
3. `p4n4-edge` — attaches to `p4n4-net`; independent of AI stack.

### Manual path (without CLI)

```bash
# Step 1: IoT stack (creates p4n4-net)
docker network create --driver bridge --subnet 172.20.0.0/16 p4n4-net
cd docker/iot && cp .env.example .env
# edit .env
docker compose up -d

# Step 2: AI stack
cd docker/ai && cp .env.example .env
# INFLUXDB_ADMIN_TOKEN must match docker/iot/.env
docker compose up -d

# Step 3: Edge stack (optional)
cd docker/edge && cp .env.example .env
docker compose up -d
```

### Hardware requirements

| Use case | RAM | Disk | GPU |
|----------|-----|------|-----|
| IoT stack only | 2 GB | 5 GB | — |
| IoT + AI (Ollama) | 8 GB | 20 GB | Optional (NVIDIA) |
| Full stack | 8 GB | 25 GB | Optional |
| Minimum supported | 4 GB | 10 GB | — |

---

## 13. Extension Points

### Adding a custom service to a stack

Add a new service block to the stack's `docker-compose.override.yml`. The `p4n4-net` network is already available for inter-service communication.

### Custom Node-RED flows

Edit `config/node-red/flows.json` directly or use the Node-RED UI at `:1880`. Export flows and commit to version control.

### Custom Grafana dashboards

Place `.json` dashboard files in `config/grafana/provisioning/dashboards/json/`. They are auto-provisioned on container start.

### Community templates

1. Create a Git repo with template files and a `p4n4-template.toml`.
2. Add an entry to `raisga/p4n4-templates/index.json` via pull request.
3. Once merged, users can install with `p4n4 template install <name>`.

### Private org index

```bash
p4n4 template org add https://raw.githubusercontent.com/acme/p4n4-index/main/index.json
```

### Phase 3: Node-RED flows library (p4n4-flows)

A planned extension that extracts versioned Node-RED flows into a standalone community library with a `p4n4 flows pull <name>` command (see Section 15).

---

## 14. Design Decisions and Trade-offs

### ADR-001: Multi-repository architecture

**Decision:** 8 separate repositories instead of a monorepo.

**Trade-offs:**

| Benefit | Cost |
|---------|------|
| Stacks independently versioned and releasable | Cross-stack secrets require manual synchronisation without CLI |
| Teams can fork one stack without cloning unrelated services | PRs touching multiple stacks need coordinated merges |
| PyPI publishing is scoped to CLI repo only | Issue triage requires knowing which sub-repo owns the problem |
| Reduced CI blast radius per repo | No atomic cross-repo commits |

### CLI as integration layer

The CLI bundles Jinja2 templates internally so scaffold works fully offline. This means template changes must be released as a new CLI version. The alternative — fetching templates from `p4n4-templates` at init time — would require network access and version pinning logic. Bundling was chosen for Phase 1 simplicity; remote template resolution may be added in Phase 2.

### Docker Compose over Helm/Kubernetes

Kubernetes was explicitly ruled out for Phase 1. The target audience (makers, small teams, self-hosters) is far more familiar with `docker compose` than with Kubernetes manifests. The operational overhead of k8s on a Raspberry Pi 4 or a NUC is unjustified for the workloads p4n4 targets. This decision can be revisited in Phase 3 if large-scale/multi-node deployments become a user requirement.

### InfluxDB v2 (not v3 or TimescaleDB)

InfluxDB v2 was chosen for its native time-series semantics, Flux query language, and first-class Node-RED integration. TimescaleDB was considered but requires PostgreSQL operational familiarity that may exceed the target audience. InfluxDB v3 (IOx) was not yet broadly available with stable Docker images at project inception.

---

## 15. Future Roadmap

### Phase 2 (near-term)

- Activate MkDocs site in `p4n4-docs` and migrate documentation off the umbrella repo.
- `p4n4-lib`: common Python library mediating between stacks and the client layer.
- `p4n4-api`: REST API gateway (`:8000`) wrapping stack operations.
- `p4n4-dashboard`: web UI for project management and stack monitoring.
- TLS support in `p4n4 init` wizard (auto-generate self-signed certs or ACME/Let's Encrypt via Caddy).

### Phase 3 (medium-term)

- **p4n4-flows**: standalone Node-RED flow library with `p4n4 flows pull <name>` command.
- **Multi-site federation**: n8n workflows + Node-RED flows for aggregating data across multiple p4n4 instances.
- **Private org template indexes**: CLI already supports `p4n4 template org add`; tooling to create and validate private indexes.
- **Helm chart**: optional Kubernetes deployment path for users graduating to multi-node.
- **ARM64 / Raspberry Pi image variants**: official support and CI testing on ARM64.

---

*Document maintained in `raisga/p4n4-docs`. Open issues or PRs there for changes.*
