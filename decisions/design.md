# p4n4 — Design Document

**Version:** 0.2  
**Date:** 2026-10-14  
**Status:** Living document

> **0.2:** the edge stack merged into the AI stack as its `ei-runner` service, and p4n4-edge
> became p4n4-extras, for optional services ([ADR-005](adr/ADR-005.md)). The sections below
> also follow the code as it is: pinned images, the CLI scaffolding from the stack repos
> through p4n4-lib, and the template registry's `template.yaml`.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Goals and Non-Goals](#2-goals-and-non-goals)
3. [System Architecture](#3-system-architecture)
4. [Component Design](#4-component-design)
   - 4.1 [IoT Stack (p4n4-iot)](#41-iot-stack-p4n4-iot)
   - 4.2 [AI Stack (p4n4-ai)](#42-ai-stack-p4n4-ai)
   - 4.3 [Edge Inference (p4n4-ai, ei-runner)](#43-edge-inference-p4n4-ai-ei-runner)
   - 4.4 [CLI (p4n4-cli)](#44-cli-p4n4-cli)
   - 4.5 [Template Registry (p4n4-templates)](#45-template-registry-p4n4-templates)
   - 4.6 [Documentation Site (p4n4-docs)](#46-documentation-site-p4n4-docs)
   - 4.7 [Extras (p4n4-extras)](#47-extras-p4n4-extras)
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

**p4n4** is a self-hosted, open-source platform for building end-to-end IoT pipelines with local AI inference. It assembles best-of-breed open-source services into two core Docker Compose stacks that can be deployed independently or together, plus an optional third for services outside the core:

| Stack | Purpose | Services |
|-------|---------|----------|
| **MING** (IoT, core) | Ingest, store, and visualise sensor data | Mosquitto · Node-RED · InfluxDB · Grafana (+ Telegraf) |
| **AI** (core) | Local LLM inference, agent memory, workflow automation, typed decisions, edge inference | Ollama (+ Letta · n8n · System 1 · inference runner: Edge Impulse `.eim` · ONNX) |
| **Extras** (optional) | Services useful next to the core but not required | None by default; one directory per extra |

Every service sits in a Compose profile of its own name; `( )` marks the ones off by default.

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
  ┌────────────────────────────────┐  ┌──────────────────────┐
  │  p4n4-ai (AI stack)            │  │ p4n4-extras          │
  │  Ollama · Letta · n8n          │  │ (optional, none by   │
  │  System 1 · runner: EI · ONNX  │  │  default)            │
  └────────────────────────────────┘  └──────────────────────┘

  ┌───────────────────────────────────────────────────────┐
  │  Client layer                                         │
  │  p4n4-cli (Python, PyPI)  ·  p4n4-api (REST :8000)   │
  │  p4n4-dashboard (Flutter app)                         │
  └───────────────────────────────────────────────────────┘
```

The Docker stacks share a single Docker bridge network (`p4n4-net`). The IoT stack owns and creates this network; the AI stack and the extras attach to it as an external network, so they stay separate Compose projects ([ADR-002](adr/ADR-002.md)).

The client layer (CLI, REST API, dashboard) communicates with stack services over `localhost` ports and with each other via `p4n4-lib`, the shared Python library.

---

## 4. Component Design

### 4.1 IoT Stack (p4n4-iot)

The foundation of every p4n4 deployment. It **owns** `p4n4-net` — no other stack can start without it (when running together).

#### Services

| Service | Image | Port | Role |
|---------|-------|------|------|
| Mosquitto | `eclipse-mosquitto:2.0` | 1883 / 9001 | MQTT broker |
| InfluxDB | `influxdb:2.9` | 8086 | Time-series storage |
| Node-RED | `nodered/node-red:5.0` | 1880 | Data routing and transformation |
| Grafana | `grafana/grafana:13.0` | 3000 | Dashboarding and alerting |
| Telegraf *(optional)* | `telegraf:1.40` | — | Host and broker metrics |

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
│   ├── mosquitto.conf    ← persistence and logging; includes the generated conf.d
│   ├── entrypoint.sh     ← writes listeners, device logins, TLS and the external-broker
│   │                       bridge (MQTT_*, MQTT_REMOTE_*) from .env at every start
│   ├── passwd.example    ← extra device accounts go in passwd, which is never committed
│   ├── acl.example       ← topic-level access rules
│   └── certs/            ← TLS certificates and keys
├── node-red/
│   ├── flows/
│   │   └── flows.json    ← version-controlled base flows
│   └── settings.js
└── grafana/
    └── provisioning/
        ├── datasources/  ← InfluxDB datasource auto-provision
        └── dashboards/   ← pre-built IoT overview dashboard
└── telegraf/             ← host and broker metrics (profile telegraf)
```

---

### 4.2 AI Stack (p4n4-ai)

Provides local LLM inference, stateful AI agents, event-driven workflow automation, fast typed decisions and edge inference. Attaches to `p4n4-net` as an external network.

#### Services

| Service | Image | Port | Role | Default |
|---------|-------|------|------|---------|
| Ollama | `ollama/ollama:0.35.1` | 11434 (localhost) | Local LLM runtime | On |
| Letta | `letta/letta:0.16.8` | 8283 (localhost) | AI agent framework with persistent memory | Off |
| n8n | `n8nio/n8n:2.41.6` | 5678 | Workflow automation and event orchestration | Off |
| System 1 | `ghcr.io/raisga/p4n4-system1:0.1.0` | — (`p4n4-system1:8000` on `p4n4-net`) | Typed decisions with laya ([ADR-004](adr/ADR-004.md)) | Off |
| ei-runner | built from `runner/` | 8080 (localhost) | Edge Impulse / ONNX inference ([4.3](#43-edge-inference-p4n4-ai-ei-runner)) | Off (on with `p4n4 init --layer all`) |

`COMPOSE_PROFILES` in the stack's `.env` lists the services that start. None depends on another, so `COMPOSE_PROFILES=ei-runner` runs edge inference alone, without an LLM.

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
| `alert-enrichment.json` | MQTT `inference/+/result` | Send low-confidence inference results to Ollama for analysis |
| `scheduled-digest.json` | Cron (hourly) | Telemetry summary in plain English |
| `device-onboarding.json` | MQTT `devices/+/register` | Auto-register new devices |
| `incident-escalation.json` | MQTT `alerts/+/critical` | Site rules, a System 1 suggestion, escalation of every alert |
| `operator-intake.json` | MQTT `operator/messages/<channel>` or a webhook | Operators' chat messages to confirmed requests, with System 1 |
| `system1-drift.json` | Webhook (Grafana alerts) | System 1 drift alerts to `system1/drift/<set>` |

#### GPU support

`docker-compose.override.yml.example` contains commented-out `deploy.resources` blocks for NVIDIA GPU passthrough to Ollama and to the inference runner. Enabling them requires the NVIDIA Container Toolkit on the host; the base compose file stays the same on a Raspberry Pi 5 or an x86 host without a GPU.

---

### 4.3 Edge Inference (p4n4-ai, ei-runner)

Runs on-device model inference inside Docker, as the AI stack's `ei-runner` service and profile. It was its own stack, p4n4-edge, until [ADR-005](adr/ADR-005.md); the service, its container (`p4n4-ei-runner`), port, MQTT topics and variables didn't change. It needs no other AI stack service, and works with or without the IoT stack (it needs an MQTT broker).

#### Services

| Service | Image | Port | Role |
|---------|-------|------|------|
| ei-runner | built from `runner/` (`python:3.11-slim`) | 8080 | Loads an Edge Impulse `.eim` or ONNX `.onnx` model (or runs in mock mode); MQTT in, MQTT + InfluxDB out; HTTP API |

The runner wraps the Edge Impulse Linux Python SDK and ONNX Runtime in one image rather than using `edgeimpulse/linux-runner`, so both model formats share one MQTT contract and one HTTP API. `MODEL_BACKEND` (`auto` | `eim` | `onnx` | `mock`) selects the backend.

#### Model lifecycle

```
1. Train a model in Edge Impulse Studio (or export any classifier to ONNX)
2. Export as a .eim Linux binary (or a .onnx file)
3. make deploy-model MODEL=...  ← copies it to edge-impulse/models/ or onnx/models/
4. Set EI_MODEL_FILE / ONNX_MODEL_FILE in the AI stack's .env; make up recreates the runner
5. Runner reads sensors/<device-id>/raw and publishes inference/<device-id>/result;
   POST :8080/api/v1/infer tests one sample without publishing
```

Model files are never committed to the repository (`.gitignore`). They are provided at deploy time by copying them into the read-only mounted model directories. `p4n4 ei deploy` will automate step 3–4; it isn't implemented yet.

#### Sensor / camera access

The AI stack's override template (`docker-compose.override.yml.example`) has `ei-runner` blocks for NVIDIA GPU access and for another broker and InfluxDB. USB cameras and serial devices are passed in by adding a `devices:` list to `ei-runner` there.

---

### 4.4 CLI (p4n4-cli)

The primary user interface. Published to PyPI as `p4n4`. `p4n4 init` copies each layer's files from its stack repo (`git clone --depth 1` of `main`), or from a local checkout with `--source-<layer>` for offline use. The shared, framework-free logic (layer registry, project layout, scaffold, validation, Compose wrappers, secrets, migrations) lives in p4n4-lib, which p4n4-api and p4n4-emu use too.

#### Tech stack

| Component | Library |
|-----------|---------|
| CLI framework | [Typer](https://typer.tiangolo.com/) + [Rich](https://rich.readthedocs.io/) |
| Interactive prompts | [Questionary](https://questionary.readthedocs.io/) |
| Shared logic | [p4n4-lib](https://github.com/raisga/p4n4-lib) (`p4n4_lib`: PyYAML, stdlib) |
| Secret generation | `secrets` module (stdlib), through `p4n4_lib.secrets` |

#### Package structure

```
p4n4/                        (p4n4-cli: presentation only)
├── cli.py                   ← Typer app; command registration
├── project.py               ← finding the project, its stack directories, legacy-layer warnings
└── commands/                ← one module per top-level command (init, lifecycle, ai, migrate, …)

p4n4_lib/                    (p4n4-lib: shared by the CLI, p4n4-api and p4n4-emu)
├── layers.py                ← layer registry: repo URLs, copy paths, required files and keys
├── layout.py                ← flat vs per-layer directories, dependency order, runner layer
├── scaffold.py              ← fetch a stack (clone or local path), copy it, write .env
├── manifest.py              ← .p4n4.json, and its dashboard block
├── validate.py · env.py · secrets.py · compose.py · migrate.py · system1.py
└── sources.yaml             ← stack repo URLs
```

#### Command surface

| Command | Description |
|---------|-------------|
| `p4n4 init NAME` | Interactive wizard — scaffold layers (`--layer iot,ai,extras,dashboard` or `all`), generate secrets, write `.p4n4.json` |
| `p4n4 up [STACK]` | Start stacks in dependency order (`--emu PROFILE` under p4n4-emu) |
| `p4n4 down [STACK]` | Stop stacks (`--all`: every p4n4 project on the host) |
| `p4n4 status` | Show container status for all stacks |
| `p4n4 logs [SERVICE] [--stack NAME]` | Stream logs |
| `p4n4 validate` | Validate manifest, required files and `.env` keys |
| `p4n4 secret show\|rotate\|generate` | Manage project secrets |
| `p4n4 ai system1 …` | System 1: enable, disable, pull, status, decide, eval, label, feedback |
| `p4n4 migrate edge` | Move a legacy edge layer into ai (or extras) |
| `p4n4 add` / `remove` / `upgrade` | Planned (stubs) |
| `p4n4 ei deploy\|run\|status` | Planned (stubs): Edge Impulse model lifecycle |
| `p4n4 template search\|install\|list` | Planned (stubs): community template registry |

#### Project manifest (.p4n4.json)

The manifest is the source of truth for a scaffolded project. It records the project name and the layers it runs; a project made from a template also names the template, and an optional `dashboard` block tells p4n4-dashboard how to present it.

```json
{
  "schema_version": 1,
  "project": "my-project",
  "layers": ["iot", "ai"],
  "created_at": "2026-10-14T00:00:00+00:00"
}
```

`p4n4 validate` checks each layer's required files and `.env` keys (an opt-in service's own, such as the inference runner's, only while its profile is enabled), and the `dashboard` block.

---

### 4.5 Template Registry (p4n4-templates)

A Git-native community registry, with no server. Each template is a directory under `projects/` that is a runnable p4n4 project on its own: one directory per layer when it has several (`iot/`, `ai/`, `extras/`), its `.p4n4.json`, a `README.md`, optionally a dashboard `theme/`, and an end-to-end smoke test.

#### Template metadata (`template.yaml`)

Each template ships a `template.yaml`, validated against `schema/template.schema.json` (required: `schema_version`, `name`, `version`, `title`, `description`, `layers`, `services`):

```yaml
schema_version: 1
name: mqtt-influx-grafana-ollama
version: 0.2.0
title: MQTT → InfluxDB → Grafana + Ollama
description: The telemetry pipeline as an iot layer, plus a local LLM in an ai layer
layers: [iot, ai]                       # iot, ai, extras
tags: [mqtt, influxdb, grafana, ollama]
services:
  - name: mqtt
    layer: iot
    description: Eclipse Mosquitto MQTT broker
    ports: [1883, 9001]
smoke_test: tests/smoke.sh
```

`scripts/validate.py` (CI, `uv run scripts/validate.py`) checks the schema, that the layers match `.p4n4.json`, that every Compose service is listed, that each layer's `.env.example` covers its compose file, and the dashboard block; CI then runs each template's smoke test. Templates ship their own files rather than rendering anything: there is no templating step.

#### Installing

`p4n4 template search|install|list` are planned (stubs today). Until then a template is copied as the project root, and each layer's `.env.example` copied to `.env` (`scripts/dev new <template> [dir]` does both in the monorepo). A remote index and private org indexes are not built yet.

---

### 4.6 Documentation Site (p4n4-docs)

Markdown in the p4n4-docs repository (`docs/` in the monorepo), read on GitHub. A static site built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) and deployed to GitHub Pages is planned (§15); the repository has no site configuration or CI yet.

Content structure:
- `README.md` — platform overview and index
- `getting-started.md` — installation and first project
- `stacks/` — `iot-stack.md`, `ai-stack.md`, `edge-inference.md` (the AI stack's runner; `edge-stack.md` points there), `extras.md`, `dashboard.md`
- `reference/` — architecture, CLI, REST API, dashboard, configuration, MQTT topics, template registry, hardware, emulator
- `guides/` — deployment, development, security, troubleshooting, migrating from the edge stack, and hosts: Raspberry Pi 5, Jetson, AWS, Google Cloud, Azure, DigitalOcean
- `decisions/` — this document, the specs, design drafts, and `adr/` (Architecture Decision Records)
- `project/` — releasing, known issues

---

### 4.7 Extras (p4n4-extras)

Optional services outside the core, in the repository that was p4n4-edge ([ADR-005](adr/ADR-005.md)). The `extras` layer is opt-in (`--layer extras`, never part of `--layer all`), starts with nothing enabled, and nothing in the IoT or AI stack depends on it; `p4n4 up` skips it while no service is enabled.

Each extra is a directory of its own (`<name>/docker-compose.yml`, `.env.example`, `config/`, `README.md`), which the layer's `docker-compose.yml` pulls in with `include:`. All extras run as one Compose project, on `p4n4-net`, with one `.env`; each sits in a Compose profile of its own name, and paths in its compose file are relative to its directory. `_template/` is the skeleton extra to copy. Rules: a pinned image, container name `p4n4-<name>`, host ports on `127.0.0.1` by default, secrets only in `.env`, and hardware settings in `docker-compose.override.yml`.

No extras ship yet. The community catalog in the [AI harness design](ai-harness.md#5-extras-community-stacks) (OBD-II, Modbus, CAN bus) builds on this layer: a manifest per extra, a validator and `p4n4 extra add` are still to come.

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

### Edge inference path (AI stack, ei-runner)

```
Sensor device
  │  feature vector on sensors/<device-id>/raw  {"values": [...]}
  ▼
Inference runner (ei-runner, p4n4-ei-runner container)
  │  .eim / .onnx inference, or mock
  ▼
Result on inference/<device-id>/result  (+ ai_events bucket in InfluxDB)
  │
  ├──► Node-RED (stores it; triggers action flows)
  └──► n8n (alert-enrichment on low confidence)

POST :8080/api/v1/infer runs one sample over HTTP without publishing it.
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

`p4n4-ai` and `p4n4-extras` declare it as external:

```yaml
networks:
  p4n4-net:
    external: true
    name: p4n4-net
```

All services on `p4n4-net` can resolve each other by container name (e.g., `http://p4n4-influxdb:8086`, `p4n4-mqtt:1883`). No additional DNS configuration is needed.

### Port exposure

| Service | Host port | Container port | Stack |
|---------|-----------|----------------|-------|
| Mosquitto MQTT | 1883 | 1883 | iot |
| Mosquitto WebSocket | 9001 | 9001 | iot |
| InfluxDB | 8086 | 8086 | iot |
| Node-RED | 1880 | 1880 | iot |
| Grafana | 3000 | 3000 | iot |
| Ollama | 11434 (`127.0.0.1`) | 11434 | ai |
| Letta | 8283 (`127.0.0.1`) | 8283 | ai |
| n8n | 5678 | 5678 | ai |
| System 1 | — | 8000 | ai |
| Inference runner (`ei-runner`) | 8080 (`127.0.0.1`) | 8080 | ai |
| p4n4 REST API | 8000 (`127.0.0.1`) | 8000 | api |
| p4n4 Dashboard (web) | 8088 | 8080 | dashboard |

Extras publish their own ports, on `127.0.0.1` by default.

### Production hardening

- Place all services behind a reverse proxy (Nginx, Caddy, Traefik) with TLS termination.
- Use firewall rules to restrict 1883 and 8086 to the local network.
- Use Mosquitto TLS (`listener 8883`) for MQTT in production.

---

## 7. Secret Management

### CLI-managed (recommended)

`p4n4 init` generates a **`.env` per stack** with cryptographically secure secrets.
Single-layer projects keep the `.env` at the project root; multi-layer projects place
one in each stack's subdirectory (see [ADR-002](adr/ADR-002.md)). Every
`docker compose` invocation runs inside the owning stack's directory, which picks up
that directory's `.env`.

```
my-project/               ← multi-layer (`--layer iot,ai`)
├── .p4n4.json
├── iot/
│   ├── .env              ← iot secrets; in .gitignore
│   └── ...
└── ai/
    ├── .env              ← ai secrets + shared InfluxDB values
    └── ...
```

Secrets are generated with Python's `secrets.token_hex()`. Cross-stack values are
written identically to every stack's `.env`. `p4n4 secret rotate` only replaces the
secrets their services read at every start (`NODE_RED_PASSWORD`, `LETTA_SERVER_PASSWORD`,
`SYSTEM1_API_KEY`, `OPERATOR_INTAKE_TOKEN`, `SYSTEM1_ALERT_TOKEN`);
InfluxDB, Grafana and n8n keep their first values, which are changed in the service (see the
[Security guide](../guides/security.md#secret-rotation)).

### Cross-stack shared secrets

| Variable | Owned by | Used by |
|----------|----------|---------|
| `INFLUXDB_TOKEN` | p4n4-iot | Node-RED, Grafana, n8n, edge runner |
| `INFLUXDB_ORG` | p4n4-iot | Node-RED, Grafana, n8n, edge runner |
| `INFLUXDB_BUCKET` | p4n4-iot | Node-RED, Grafana, n8n |
| `MQTT_USER` / `MQTT_PASSWORD` | p4n4-iot | Node-RED, edge runner |
| `GRAFANA_USER` / `GRAFANA_PASSWORD` | p4n4-iot | Grafana |
| `NODE_RED_USER` / `NODE_RED_PASSWORD` | p4n4-iot | Node-RED editor login |
| `N8N_BASIC_AUTH_USER` / `_PASSWORD` | p4n4-ai | n8n |
| `N8N_ENCRYPTION_KEY` | p4n4-ai | n8n |
| `LETTA_SERVER_PASSWORD` | p4n4-ai | Letta, Node-RED |
| `EI_API_KEY` | p4n4-ai | edge runner (optional) |

When using stacks without the CLI, these must be kept consistent manually across per-stack `.env` files.

---

## 8. Security Model

### Threat model (self-hosted, LAN deployment)

| Threat | Mitigation |
|--------|-----------|
| Default credentials exploited | CLI generates unique secrets per project; no global defaults committed |
| Secrets in version control | `.env` is in `.gitignore`; only `.env.example` (placeholders) is committed |
| Unauthenticated MQTT | Internal services use a `p4n4-net`-only listener; device listeners require a login once `MQTT_PASSWORD` is set, plus an optional ACL and TLS |
| Exposed services on public IP | Bind services to localhost or use firewall; reverse proxy with TLS for WAN |
| Model binary tampering | `.eim` files are externally sourced; integrity verification is out of scope for Phase 1 |
| Compromised container | Docker network isolation limits blast radius; no host bind-mounts for sensitive paths |

### Development vs production posture

| Setting | Development (default) | Production (hardened) |
|---------|-----------------------|-----------------------|
| MQTT device login | None (`MQTT_PASSWORD` empty) | `MQTT_USER` / `MQTT_PASSWORD`, an account per device, ACL, TLS listener |
| Service binding | Ollama, Letta, the inference runner and p4n4-api on `127.0.0.1`; the rest on `0.0.0.0` | `127.0.0.1` or reverse proxy |
| TLS | Off | Reverse proxy terminates TLS |
| Secrets | Placeholder or CLI-generated | CLI-generated; rotated periodically |
| InfluxDB exposed | Yes (`:8086`) | Firewall-restricted or internal-only |

### Vulnerability disclosure

Vulnerabilities are reported privately through a [private security advisory](https://github.com/raisga/p4n4/security/advisories/new), as the [Security guide](../guides/security.md#reporting-vulnerabilities) describes. Public issues must not contain exploit details.

---

## 9. Repository Strategy

p4n4 is split across separate repositories under the `raisga` GitHub organisation (ADR-001), each with a single, independently releasable responsibility. The `p4n4` umbrella gathers them as Git submodules, so one checkout holds the whole platform ([architecture](../reference/architecture.md)).

| Repo | Type | Ships as | Description |
|------|------|----------|-------------|
| `.github` | org meta | — | Org profile, shared workflows, issue templates |
| `p4n4` | umbrella | — | Landing page, every repo below as a submodule |
| `p4n4-iot` | stack | — | IoT Docker Compose stack; owns `p4n4-net` |
| `p4n4-ai` | stack | `ghcr.io/raisga/p4n4-system1` | AI Docker Compose stack, edge inference included |
| `p4n4-extras` | stack | — | Optional services (formerly p4n4-edge, [ADR-005](adr/ADR-005.md)) |
| `p4n4-lib` | library | PyPI `p4n4-lib` | Shared logic for the CLI, p4n4-api and p4n4-emu |
| `p4n4-cli` | tool | PyPI `p4n4` | Python CLI |
| `p4n4-api` | service | `ghcr.io/raisga/p4n4-api` | REST API gateway |
| `p4n4-dashboard` | frontend | `ghcr.io/raisga/p4n4-dashboard` | Web service and desktop/mobile apps |
| `p4n4-templates` | registry | — | Community templates |
| `p4n4-emu` | tool | — | Workstation hardware emulator |
| `p4n4-hw` | hardware | — | KiCad designs and RPi5 scripts |
| `p4n4-docs` | docs | — | This documentation |

### Why multi-repo

- Stack repos can be version-tagged, forked, and deployed independently.
- PyPI publishing for the CLI only requires access to `p4n4-cli` — no monorepo tooling needed.
- Teams can contribute templates without repository-wide write access.
- Reduces blast radius of a compromised CI token.

### Cross-repo coordination

p4n4-lib is the integration layer: its layer registry (`layers.py`, `sources.yaml`) names each stack repo, the files a project copies from it, and the files and `.env` keys a valid project needs. `p4n4 init` clones each stack's `main`, so a stack change that moves or requires files lands with a p4n4-lib release that knows about it, and [releasing.md](../project/releasing.md) orders the publishes (stacks pushed, then p4n4-lib, then its users).

---

## 10. Versioning and Compatibility

### Per-repo semver

Each repository uses independent `MAJOR.MINOR.PATCH` Git tags. The `p4n4-cli` package on PyPI follows the same scheme.

### CLI ↔ stack compatibility

There is no runtime version check. The stacks are scaffolded from `main` at `p4n4 init`, so the compatibility contract is p4n4-lib's layer registry: the CLI and p4n4-api declare the p4n4-lib versions they need (`p4n4-lib>=0.4.0`), and the stack changes a p4n4-lib release relies on are pushed before it. A layer the registry drops stays runnable as a legacy layer until a `p4n4 migrate` command moves it (`edge`, [ADR-005](adr/ADR-005.md)).

### Release flow (p4n4-cli)

```
1. Bump the version in pyproject.toml and p4n4/__init__.py
2. Update CHANGELOG.md and write release-notes/v<version>.md
3. Open PR → CI passes (lint + test on Python 3.11/3.12/3.13)
4. Merge to main
5. gh release create v<version> --notes-file release-notes/v<version>.md
6. publish.yml (on a published release) → python -m build → PyPI upload
```

p4n4-lib follows the same flow, before the CLI that needs it.

---

## 11. CI/CD Strategy

### Stack repos (p4n4-iot, p4n4-ai, p4n4-extras)

On every push and PR:

1. `docker compose config --quiet`, with `.env.example` too — validate the compose file.
2. `yamllint docker-compose.yml` — lint it.
3. `python3 scripts/check_env_example.py` — verify `.env.example` lists every variable the compose file uses (in p4n4-extras, every included extra's and the skeleton's).

p4n4-ai also validates the n8n workflows and System 1 question sets, builds the inference runner and runs its tests (`pytest tests/` on Python 3.11–3.13); the System 1 image has its own workflow. p4n4-extras also validates `_template/`.

### p4n4-cli

Matrix test across Python 3.11, 3.12, 3.13 on every PR:

1. `ruff check .` — lint.
2. `pytest --cov=p4n4` — unit tests with coverage, scaffolding from shallow clones of the stack repos.

On a published GitHub release:

1. `python -m build` — build wheel and sdist.
2. Upload to PyPI.

### p4n4-templates

On every PR:

1. `uv run scripts/validate.py` — validate every template (§4.5).
2. Each template's smoke test (`tests/smoke.sh`), in an isolated Compose project.

### p4n4-docs

None yet. When the MkDocs site lands (§15), push to `main` will build it and deploy to GitHub Pages.

---

## 12. Deployment Model

### CLI-managed (recommended path)

```bash
pip install p4n4
p4n4 init my-project     # interactive wizard
cd my-project
p4n4 up                  # starts: iot → ai → extras → dashboard (in order)
```

The CLI starts each layer's Compose project in dependency order with `docker compose up -d`. It creates `p4n4-net` when it's missing, refuses to start when another project holds the fixed `p4n4-*` container names, and skips a layer with no service enabled. Within a stack, `depends_on` with healthchecks orders the services (Node-RED and Grafana wait for a healthy InfluxDB, for example).

1. `p4n4-iot` — creates `p4n4-net`, starts Mosquitto and InfluxDB first.
2. `p4n4-ai` — attaches to `p4n4-net`; Ollama by default, the inference runner when `ei-runner` is enabled.
3. `p4n4-extras` — attaches to `p4n4-net`; skipped while nothing is enabled.
4. `p4n4-dashboard` — last, after the stacks it shows.

### Manual path (without CLI)

```bash
# Step 1: IoT stack (its Compose file creates p4n4-net)
cd stacks/iot && cp .env.example .env
# edit .env
docker compose up -d

# Step 2: AI stack (add ei-runner to COMPOSE_PROFILES for edge inference)
cd stacks/ai && cp .env.example .env
# INFLUXDB_TOKEN, INFLUXDB_ORG and MQTT credentials must match stacks/iot/.env
docker compose up -d

# Step 3: Extras (optional; nothing starts until COMPOSE_PROFILES lists an extra)
cd stacks/extras && cp .env.example .env
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

### Adding an optional service

Add it as an extra (§4.7): copy `_template/` to a directory of its own in the extras layer and include it. It stays opt-in and out of the core stacks' files.

### Adjusting a core service

Put machine-specific changes (a GPU, a device, other ports) in the stack's `docker-compose.override.yml`. The `p4n4-net` network is already available for inter-service communication.

### Custom Node-RED flows

Edit `config/node-red/flows/flows.json` directly or use the Node-RED UI at `:1880`. Deploying from the UI saves to the same file, so commit it to version control. Compose mounts the `flows/` directory rather than the file, because Node-RED saves by renaming a temporary file over `flows.json`.

### Custom Grafana dashboards

Place `.json` dashboard files in `config/grafana/provisioning/dashboards/json/`. They are auto-provisioned on container start.

### Community templates

1. Add a project directory with a `template.yaml` under `projects/` in `raisga/p4n4-templates` via pull request ([authoring guide](https://github.com/raisga/p4n4-templates/blob/main/docs/authoring.md)).
2. CI validates it and runs its smoke test.
3. Once merged, users copy it as their project root (`p4n4 template install <name>` is planned).

Private or org indexes are planned, not built.

### Phase 3: Node-RED flows library (p4n4-flows)

A planned extension that extracts versioned Node-RED flows into a standalone community library with a `p4n4 flows pull <name>` command (see Section 15).

---

## 14. Design Decisions and Trade-offs

### ADR-001: Multi-repository architecture

**Decision:** separate repositories instead of a monorepo, gathered in the `p4n4` umbrella as submodules.

**Trade-offs:**

| Benefit | Cost |
|---------|------|
| Stacks independently versioned and releasable | Cross-stack secrets require manual synchronisation without CLI |
| Teams can fork one stack without cloning unrelated services | PRs touching multiple stacks need coordinated merges |
| PyPI publishing is scoped to CLI repo only | Issue triage requires knowing which sub-repo owns the problem |
| Reduced CI blast radius per repo | No atomic cross-repo commits |

### Stack repos as the scaffold source

`p4n4 init` copies each layer from its stack repo (a shallow clone of `main`), so stack fixes reach new projects without a CLI release, and the files a project gets are the files the stack's CI tested. The cost is network access at init (`--source-<layer>` works offline from a local checkout) and keeping `main` compatible with the released p4n4-lib (§10). The CLI renders no templates (its `jinja2` dependency is unused).

### Two core stacks, extras outside them

The runner of on-device models joined the AI stack instead of keeping a stack of its own, and optional services got their own opt-in layer, so the core stays two stacks and nothing optional grows inside them ([ADR-005](adr/ADR-005.md)).

### Docker Compose over Helm/Kubernetes

Kubernetes was explicitly ruled out for Phase 1. The target audience (makers, small teams, self-hosters) is far more familiar with `docker compose` than with Kubernetes manifests. The operational overhead of k8s on a Raspberry Pi 4 or a NUC is unjustified for the workloads p4n4 targets. This decision can be revisited in Phase 3 if large-scale/multi-node deployments become a user requirement.

### InfluxDB v2 (not v3 or TimescaleDB)

InfluxDB v2 was chosen for its native time-series semantics, Flux query language, and first-class Node-RED integration. TimescaleDB was considered but requires PostgreSQL operational familiarity that may exceed the target audience. InfluxDB v3 (IOx) was not yet broadly available with stable Docker images at project inception.

---

## 15. Future Roadmap

### Phase 2 (near-term)

- Activate MkDocs site in `p4n4-docs`.
- ~~`p4n4-lib`: common Python library mediating between stacks and the client layer.~~ Done: on PyPI as `p4n4-lib`.
- `p4n4-api`: REST API gateway (`:8000`); shipped, see [REST API](../reference/api.md) for what it covers and what's next.
- `p4n4-dashboard`: Flutter app for project management and stack monitoring; shipped, see [Dashboard](../reference/dashboard.md).
- Extras catalog: a manifest per extra, a validator and `p4n4 extra add` ([AI harness design](ai-harness.md#5-extras-community-stacks)).
- TLS support in `p4n4 init` wizard (auto-generate self-signed certs or ACME/Let's Encrypt via Caddy).

### Phase 3 (medium-term)

- **p4n4-flows**: standalone Node-RED flow library with `p4n4 flows pull <name>` command.
- **Multi-site federation**: n8n workflows + Node-RED flows for aggregating data across multiple p4n4 instances.
- **Template commands and private org indexes**: `p4n4 template search|install|list`, then indexes hosted by an organisation.
- **Helm chart**: optional Kubernetes deployment path for users graduating to multi-node.
- **ARM64 / Raspberry Pi image variants**: official support and CI testing on ARM64.

---

*Document maintained in `raisga/p4n4-docs`. Open issues or PRs there for changes.*
