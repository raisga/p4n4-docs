# p4n4 — Specifications Roadmap

**Version:** 0.1-draft
**Status:** Living document
**Maintained in:** [raisga/p4n4-docs](https://github.com/raisga/p4n4-docs)

---

## Table of Contents

1. [Purpose and Scope](#1-purpose-and-scope)
2. [Platform Version Matrix](#2-platform-version-matrix)
3. [Feature Dependency Map](#3-feature-dependency-map)
4. [Phase 1: Foundation (v0.1) — PLANNED](#4-phase-1-foundation-v01--planned)
5. [Phase 2: Intelligence Layer (v0.2) — FUTURE](#5-phase-2-intelligence-layer-v02--future)
6. [Phase 3: Scale & Extensibility (v0.3+) — FUTURE](#6-phase-3-scale--extensibility-v03--future)
7. [v1.0 Production Stability — FUTURE](#7-v10-production-stability--future)
8. [Technical Specifications](#8-technical-specifications)
9. [Repository & CI Reference](#9-repository--ci-reference)
10. [Appendix](#10-appendix)

---

## 1. Purpose and Scope

This document is the canonical specifications roadmap for the p4n4 platform. It defines what each release version of p4n4 delivers, the acceptance criteria each feature must meet, the interface contracts between components, and the test requirements that gate each milestone.

**Audience:** contributors, maintainers, and external collaborators who need to understand the implementation plan at a technical level — not the product narrative. That lives in [`design.md`](design.md).

**How to read this document:**

- Acceptance criteria under `PLANNED` phases are binding requirements for that release.
- Items under `FUTURE` phases are architectural intent; details will be refined as prior phases complete.
- This document does not replace architectural rationale (see [`design.md`](design.md)) or multi-repo structure decisions (see [`../reference/architecture.md`](../reference/architecture.md)).

**Relationship to other documents:**

| Document | What it covers |
|---|---|
| [`design.md`](design.md) | Goals, architecture, system design, design decisions |
| [`../reference/architecture.md`](../reference/architecture.md) | Multi-repo layout, cross-repo dependencies, CI YAML, versioning policy |
| [`cli-reference.md`](../reference/cli-reference.md) | Full CLI command reference with examples |
| [`adr/ADR-001.md`](adr/ADR-001.md) | Multi-repository architecture decision record |
| [`adr/ADR-002.md`](adr/ADR-002.md) | Per-layer subdirectories in multi-layer projects |
| [`../index.md`](../index.md) | Platform landing page and narrative roadmap |
| **This document** | Feature specifications, acceptance criteria, interface contracts, test requirements |

---

## 2. Platform Version Matrix

### 2.1 Release Summary

| Version | Status  | Theme                    | Primary Repos Touched                          |
|---------|---------|--------------------------|------------------------------------------------|
| v0.1    | PLANNED | Foundation               | p4n4-cli, p4n4-iot, p4n4-ai, p4n4-edge        |
| v0.2    | FUTURE  | Intelligence Layer       | p4n4-cli, p4n4-ai, p4n4-templates              |
| v0.3    | FUTURE  | Scale & Extensibility    | p4n4-cli, p4n4-iot, p4n4-ai, p4n4-helm (new)  |
| v1.0    | FUTURE  | Production Stability     | all repos                                      |

> **Current priority:** Deliver a functional `p4n4` CLI installable via `pip install p4n4` that can scaffold, start, and stop the IoT, GenAI, and Edge AI stacks. Everything in Phase 1 is in scope for v0.1. Nothing has been built yet — all features start from zero.

### 2.2 CLI-to-Stack Compatibility Matrix

Each stack repo uses independent semver. The `p4n4-cli` `compat.py` module enforces minimum stack versions at runtime. The table below reflects the compatibility contract per CLI release.

| CLI Version | p4n4-iot | p4n4-ai  | p4n4-edge |
|-------------|----------|----------|-----------|
| 0.1.x       | >=0.1.0  | >=0.1.0  | >=0.1.0   |
| 0.2.x       | >=0.1.0  | >=0.2.0  | >=0.1.0   |
| 0.3.x       | >=0.3.0  | >=0.3.0  | >=0.1.0   |
| 1.0.x       | >=1.0.0  | >=1.0.0  | >=1.0.0   |

**Versioning policy:** "Platform version" refers to the CLI's PyPI release version (`p4n4` on PyPI), which is the primary integration point. Each stack repo publishes its own patch releases between CLI milestones. The CLI will warn — not fail — when a stack version is within the compatible range but below the latest known patch.

---

## 3. Feature Dependency Map

### 3.1 Directed Dependency Graph

```
v0.1 Foundation (planned — no external dependencies)
├── [F-0.1.1] p4n4-iot compose stack
├── [F-0.1.2] p4n4-ai compose stack ──────────── requires: F-0.1.1 (p4n4-net bridge)
├── [F-0.1.3] p4n4-edge compose stack ─────────── requires: F-0.1.1 (p4n4-net bridge)
├── [F-0.1.4] p4n4-cli core (init/up/down/status/validate)
├── [F-0.1.5] Secret management (p4n4 secret)
├── [F-0.1.6] Project manifest (.p4n4.json)
└── [F-0.1.7] PyPI publish

v0.2 Intelligence Layer
├── [F-0.2.1] Letta agent personas ─────────────── requires: F-0.1.2
├── [F-0.2.2] Node-RED AI palette ──────────────── requires: F-0.1.1, F-0.1.2
├── [F-0.2.3] n8n workflow library expansion ────── requires: F-0.1.2
├── [F-0.2.4] p4n4 ei subcommands ─────────────── requires: F-0.1.3, F-0.1.4
├── [F-0.2.5] Template registry v2 ────────────── requires: F-0.1.6
└── [F-0.2.6] Official community templates ──────── requires: F-0.2.5

v0.3 Scale & Extensibility
├── [F-0.3.1] Multi-site federation ────────────── requires: F-0.2.3, F-0.2.1
├── [F-0.3.2] OpenTelemetry integration ──────────── requires: F-0.1.1, F-0.1.2
├── [F-0.3.3] Embedding pipeline ──────────────── requires: F-0.2.1, F-0.3.2
├── [F-0.3.4] Optional Kafka/NATS layer ─────────── requires: F-0.1.1
├── [F-0.3.5] Kubernetes/Helm chart ───────────────── requires: F-0.1.1, F-0.1.2, F-0.1.3
├── [F-0.3.6] p4n4-flows standalone library ──────── requires: F-0.2.2
└── [F-0.3.7] Shell completions ─────────────────── requires: F-0.1.4
```

### 3.2 Dependency Table

| Feature ID | Name                          | Depends On              | Blocks                  |
|------------|-------------------------------|-------------------------|-------------------------|
| F-0.1.1    | p4n4-iot stack                | —                       | F-0.1.2, F-0.1.3, F-0.2.2, F-0.3.2, F-0.3.4, F-0.3.5 |
| F-0.1.2    | p4n4-ai stack                 | F-0.1.1                 | F-0.2.1, F-0.2.2, F-0.2.3, F-0.3.2, F-0.3.5 |
| F-0.1.3    | p4n4-edge stack               | F-0.1.1                 | F-0.2.4, F-0.3.5        |
| F-0.1.4    | CLI core                      | —                       | F-0.2.4, F-0.3.7        |
| F-0.1.5    | Secret management             | F-0.1.4                 | —                       |
| F-0.1.6    | Project manifest              | F-0.1.4                 | F-0.2.5                 |
| F-0.1.7    | PyPI publish                  | F-0.1.4                 | —                       |
| F-0.2.1    | Letta agent personas          | F-0.1.2                 | F-0.3.1, F-0.3.3        |
| F-0.2.2    | Node-RED AI palette           | F-0.1.1, F-0.1.2        | F-0.3.6                 |
| F-0.2.3    | n8n workflow library          | F-0.1.2                 | F-0.3.1                 |
| F-0.2.4    | p4n4 ei subcommands           | F-0.1.3, F-0.1.4        | —                       |
| F-0.2.5    | Template registry v2          | F-0.1.6                 | F-0.2.6                 |
| F-0.2.6    | Official community templates  | F-0.2.5                 | —                       |
| F-0.3.1    | Multi-site federation         | F-0.2.3, F-0.2.1        | —                       |
| F-0.3.2    | OpenTelemetry integration     | F-0.1.1, F-0.1.2        | F-0.3.3                 |
| F-0.3.3    | Embedding pipeline            | F-0.2.1, F-0.3.2        | —                       |
| F-0.3.4    | Optional Kafka/NATS layer     | F-0.1.1                 | —                       |
| F-0.3.5    | Kubernetes/Helm chart         | F-0.1.1, F-0.1.2, F-0.1.3 | —                    |
| F-0.3.6    | p4n4-flows library            | F-0.2.2                 | —                       |
| F-0.3.7    | Shell completions             | F-0.1.4                 | —                       |

---

## 4. Phase 1: Foundation (v0.1) — PLANNED

**Priority order within this phase:** The CLI package (F-0.1.4, F-0.1.6, F-0.1.7) and the three compose stacks (F-0.1.1–F-0.1.3) are the core deliverables. Secret management (F-0.1.5) ships alongside the CLI. Nothing in this phase exists yet.

### 4.1 Release Summary

| Feature ID | Name                       | Repo(s)           | Priority | Status  |
|------------|----------------------------|-------------------|----------|---------|
| F-0.1.4    | CLI core commands          | p4n4-cli          | 1 — High | Planned |
| F-0.1.6    | Project manifest           | p4n4-cli          | 1 — High | Planned |
| F-0.1.7    | PyPI publish               | p4n4-cli          | 1 — High | Planned |
| F-0.1.1    | p4n4-iot compose stack     | p4n4-iot          | 2 — High | Planned |
| F-0.1.2    | p4n4-ai compose stack      | p4n4-ai           | 2 — High | Planned |
| F-0.1.3    | p4n4-edge compose stack    | p4n4-edge         | 2 — High | Planned |
| F-0.1.5    | Secret management          | p4n4-cli          | 3 — Med  | Planned |

---

### F-0.1.4 — CLI Core Commands

**Repo:** `p4n4-cli`
**Description:** The primary user interface for p4n4. Built with Typer, Rich, Questionary, and Jinja2. Bundles Jinja2 templates for offline scaffolding. Published to PyPI as `p4n4`.

**Command surface:**

| Command                     | Description                                              |
|-----------------------------|----------------------------------------------------------|
| `p4n4 init [PATH]`          | Interactive scaffold wizard; generates stack files       |
| `p4n4 add STACK`            | Add a stack to an existing project                       |
| `p4n4 remove STACK`         | Remove a stack from a project                            |
| `p4n4 up [STACK]`           | Start one or all stacks                                  |
| `p4n4 down [STACK]`         | Stop one or all stacks                                   |
| `p4n4 status`               | Show container status for all stacks                     |
| `p4n4 logs STACK`           | Tail logs for a stack                                    |
| `p4n4 validate`             | Validate `.env` completeness and compose config          |

**Startup order enforced by `p4n4 up` (no `STACK` argument):**

1. Start `p4n4-iot` → poll Docker healthchecks until Mosquitto and InfluxDB are healthy.
2. Start `p4n4-ai` → poll until Ollama and Letta respond to HTTP health endpoints.
3. Start `p4n4-edge` → no dependency polling required.

**Acceptance Criteria:**

- [ ] `p4n4 init PATH` scaffolds all selected stack `docker-compose.yml` and `.env.example` files using bundled Jinja2 templates without any network access.
- [ ] `p4n4 init` prompts for project name, selects stacks interactively, and writes a valid `.p4n4.json` manifest.
- [ ] `p4n4 up` without arguments starts stacks in the defined order, polling Docker healthchecks between stages.
- [ ] `p4n4 down --volumes` removes named volumes as well as containers.
- [ ] `p4n4 status` prints a Rich table with columns: container name, image, status, ports.
- [ ] `p4n4 validate` exits non-zero and prints actionable error messages for each missing required `.env` variable.
- [ ] `p4n4 logs STACK` streams docker compose logs for the specified stack.
- [ ] All commands exit `0` on success and non-zero on failure, with human-readable Rich error output.

---

### F-0.1.5 — Secret Management

**Repo:** `p4n4-cli`
**Description:** CLI subcommand surface for managing secrets across stack `.env` files. Generates cryptographically secure values using Python's `secrets` module.

**Command interface:**

```bash
p4n4 secret show          # Masked table: variable name, masked value, source stack
p4n4 secret rotate        # Regenerates all secrets in .env files; prompts for confirmation
p4n4 secret generate      # Prints fresh secrets to stdout; does NOT write .env
```

**Acceptance Criteria:**

- [ ] `p4n4 secret show` prints a Rich table with columns: variable name, masked value (`****`), stack, and whether the variable is a cross-stack shared secret.
- [ ] `p4n4 secret rotate` prompts for confirmation before overwriting any `.env` file.
- [ ] `p4n4 secret rotate` generates all secrets using `secrets.token_urlsafe(32)` or equivalent.
- [ ] `p4n4 secret generate` writes nothing to disk; all output goes to stdout only.
- [ ] `.env` files are never committed (confirmed by `.gitignore` entries in scaffolded projects).

---

### F-0.1.6 — Project Manifest

**Repo:** `p4n4-cli`
**Description:** The `.p4n4.json` manifest records which stacks are enabled, the project name, and the CLI version used for scaffolding. It is the machine-readable project identity file.

**Schema (`manifest-v1.json`):**

```json
{
  "$schema": "https://raisga.github.io/p4n4-docs/schemas/manifest-v1.json",
  "name": "<string, required>",
  "version": "<string, semver, required>",
  "cli_version": "<string, semver, required>",
  "stacks": ["<array of: iot | ai | edge>"],
  "created_at": "<string, ISO 8601 datetime, required>",
  "template": "<string, optional — template name if installed via p4n4 template>"
}
```

**Acceptance Criteria:**

- [ ] Every `p4n4 init` run writes a `.p4n4.json` at the project root that validates against `manifest-v1.json` schema.
- [ ] `p4n4 add STACK` appends the stack name to the `stacks` array and is idempotent.
- [ ] `p4n4 remove STACK` removes the stack name from the `stacks` array.
- [ ] `p4n4 validate` fails with exit code `2` if `.p4n4.json` is absent.
- [ ] `cli_version` is set to the installed CLI version at scaffold time and never auto-updated on subsequent commands.

---

### F-0.1.7 — PyPI Publish

**Repo:** `p4n4-cli`
**Description:** The `p4n4` package is published to PyPI via the `publish.yml` GitHub Actions workflow, triggered on `git tag v*` pushes.

**Acceptance Criteria:**

- [ ] `pip install p4n4` on Python 3.11, 3.12, and 3.13 succeeds.
- [ ] `p4n4 --version` returns the correct version string matching `pyproject.toml`.
- [ ] The `publish.yml` workflow triggers only on tag pushes matching `v*`.
- [ ] The package bundles all Jinja2 templates as package data (accessible offline after install).

---

### F-0.1.1 — p4n4-iot Compose Stack

**Repo:** `p4n4-iot`
**Description:** Docker Compose stack delivering the MING tier: Mosquitto (MQTT broker), Node-RED (flow orchestrator), InfluxDB v2 (time-series store), and Grafana (dashboards). This stack creates and owns the `p4n4-net` Docker bridge network that all other stacks attach to.

**Acceptance Criteria:**

- [ ] `docker compose up` in `p4n4-iot/` starts all four services (Mosquitto, Node-RED, InfluxDB, Grafana) without errors.
- [ ] Mosquitto binds to port 1883 (TCP) and 9001 (WebSocket) on the host.
- [ ] Node-RED UI is reachable at `http://localhost:1880`.
- [ ] InfluxDB UI is reachable at `http://localhost:8086`.
- [ ] Grafana UI is reachable at `http://localhost:3000`.
- [ ] Docker bridge network `p4n4-net` with subnet `172.20.0.0/16` is created when the stack starts.
- [ ] All services are joined to `p4n4-net` and can reach each other by Docker DNS name.
- [ ] A `.env.example` is committed with all required variable names and placeholder values; no real secrets are committed.

---

### F-0.1.2 — p4n4-ai Compose Stack

**Repo:** `p4n4-ai`
**Description:** Docker Compose stack delivering the GenAI tier: Ollama (local LLM inference), Letta (stateful agent memory framework), and n8n (workflow automation). This stack attaches to `p4n4-net` as an external network.

**Acceptance Criteria:**

- [ ] `docker compose up` in `p4n4-ai/` starts all three services (Ollama, Letta, n8n) without errors when `p4n4-net` exists.
- [ ] Ollama API is reachable at `http://localhost:11434`.
- [ ] Letta API is reachable at `http://localhost:8283`.
- [ ] n8n UI is reachable at `http://localhost:5678`.
- [ ] All services resolve `influxdb`, `mosquitto`, and `nodered` hostnames via Docker DNS.
- [ ] The stack declares `p4n4-net` as `external: true` and fails with a descriptive error if the network does not exist.
- [ ] Four starter n8n workflow JSON files are bundled in `config/n8n/workflows/`: `alert-enrichment.json`, `scheduled-digest.json`, `device-onboarding.json`, `incident-escalation.json`.

---

### F-0.1.3 — p4n4-edge Compose Stack

**Repo:** `p4n4-edge`
**Description:** Docker Compose stack running the Edge Impulse Linux Runner. Fully independent; attaches to `p4n4-net` but does not require any other p4n4 stack to function.

**Acceptance Criteria:**

- [ ] `docker compose up` in `p4n4-edge/` starts the Edge Impulse runner without errors when `p4n4-net` exists.
- [ ] Runner HTTP API is reachable at `http://localhost:8080`.
- [ ] A bind-mount at `./models/` is available for `.eim` model files.
- [ ] The stack attaches to `p4n4-net` as `external: true`.
- [ ] `.env.example` includes `EI_API_KEY` placeholder.

---

### 4.2 v0.1 Known Limitations

The following are explicit non-goals for v0.1. Their absence is not a defect.

- No TLS termination wizard; users must configure a reverse proxy manually.
- No multi-site federation; single-node deployment only.
- No Kubernetes or Helm chart support.
- No shell tab-completion.
- No embedding or semantic search pipeline.
- `p4n4 ei infer` and `p4n4 ei list` are not yet implemented (only `ei deploy`, `ei run`, `ei status`).
- Template registry is read-only; no `p4n4 template push` command.

---

### 4.3 v0.1 Test Requirements

| Test ID  | Scope              | Type        | Command / Assertion                              |
|----------|--------------------|-------------|--------------------------------------------------|
| T-0.1.1  | CLI init           | Unit        | `pytest test_init.py`                            |
| T-0.1.2  | Manifest read/write| Unit        | `pytest test_manifest.py`                        |
| T-0.1.3  | Secret generation  | Unit        | `pytest test_secret.py`                          |
| T-0.1.4  | Validate command   | Unit        | `pytest test_validate.py`                        |
| T-0.1.5  | Compose valid      | Integration | `docker compose config --quiet` (CI, all stacks) |
| T-0.1.6  | YAML lint          | Static      | `yamllint .` (CI, all stack repos)               |
| T-0.1.7  | .env coverage      | Static      | `check_env_example.py` (CI, all stack repos)     |
| T-0.1.8  | PyPI install smoke | Smoke       | `pip install p4n4 && p4n4 --version`             |

---

## 5. Phase 2: Intelligence Layer (v0.2) — FUTURE

### 5.1 Release Summary

| Feature ID | Name                          | Repo(s)                   | Status |
|------------|-------------------------------|---------------------------|--------|
| F-0.2.1    | Letta agent personas          | p4n4-ai, p4n4-cli         | Future |
| F-0.2.2    | Node-RED AI palette           | p4n4-iot, p4n4-cli        | Future |
| F-0.2.3    | n8n workflow library          | p4n4-ai                   | Future |
| F-0.2.4    | p4n4 ei subcommands           | p4n4-cli                  | Future |
| F-0.2.5    | Template registry v2          | p4n4-cli, p4n4-templates  | Future |
| F-0.2.6    | Official community templates  | p4n4-templates            | Future |

---

### F-0.2.1 — Pre-built Letta Agent Personas

**Repo:** `p4n4-ai`, `p4n4-cli` (scaffold templates updated)
**Description:** Three named Letta agents are shipped as importable configuration artifacts in `p4n4-ai/letta/agents/`. Each persona defines a system prompt, memory configuration, and preferred model. The CLI gains `p4n4 ai agent` subcommands for managing them.

**Persona catalog:**

| Persona ID          | Display Name        | Primary Function                                    |
|---------------------|---------------------|-----------------------------------------------------|
| `site-monitor`      | Site Monitor        | Receives all sensor events; tracks device uptime    |
| `anomaly-analyst`   | Anomaly Analyst     | Watches telemetry trends; fires alerts on deviation |
| `operator-assistant`| Operator Assistant  | Answers natural-language queries about the deployment |

**CLI additions:**

```bash
p4n4 ai agent list              # Rich table: name, status, model, agent ID
p4n4 ai agent init <persona>    # Create agent via Letta REST API
p4n4 ai agent delete <name>     # Delete agent by name
p4n4 ai agent chat <name>       # Interactive chat with an agent
```

**Agent configuration schema (Letta API POST /v1/agents body):**

```json
{
  "name": "<string>",
  "persona": "<string — system prompt template>",
  "human": "<string — user persona context>",
  "model": "<string — ollama model name, e.g. mistral:7b>",
  "model_endpoint_type": "ollama",
  "model_endpoint": "http://ollama:11434",
  "context_window": 8192,
  "embedding_model": "nomic-embed-text",
  "embedding_endpoint_type": "ollama"
}
```

**Acceptance Criteria:**

- [ ] `p4n4 ai agent list` prints a Rich table with columns: name, status (running/stopped), model, agent ID.
- [ ] `p4n4 ai agent init site-monitor` creates the agent via `POST /v1/agents` on the Letta API and verifies HTTP 200.
- [ ] Each persona's system prompt template is stored as a TOML file in `p4n4-ai/letta/agents/<persona-id>.toml` and version-controlled.
- [ ] Each persona's system prompt is ≤ 2,048 tokens for the default model context window.
- [ ] Agents persist across a `docker compose restart letta` (data stored in Letta's volume-backed SQLite store).

---

### F-0.2.2 — Node-RED AI Palette

**Repo:** `p4n4-iot` (flows.json updates), `p4n4-cli` (bundled template updated)
**Description:** A set of pre-built Node-RED subflows wrapping Ollama and Letta HTTP APIs. Delivered as importable JSON subflow definitions embedded in the base `flows.json` template. Not an NPM package in v0.2 (extraction to `p4n4-flows` is F-0.3.6).

**Subflow catalog:**

| Subflow Name         | Input (msg fields)       | Output (msg fields)   | HTTP Call                          |
|----------------------|--------------------------|-----------------------|------------------------------------|
| `ollama-generate`    | `payload` (string prompt)| `payload` (response)  | `POST ollama:11434/api/generate`   |
| `ollama-chat`        | `messages` (array)       | `payload` (reply)     | `POST ollama:11434/api/chat`       |
| `letta-send-event`   | `agent_id`, `payload`    | `payload` (response)  | `POST letta:8283/v1/agents/:id/messages` |
| `letta-get-memory`   | `agent_id`               | `payload` (memory obj)| `GET letta:8283/v1/agents/:id/memory` |
| `influx-flux-query`  | `payload` (Flux string)  | `payload` (rows array)| `POST influxdb:8086/api/v2/query`  |

**Acceptance Criteria:**

- [ ] Each subflow is importable via the Node-RED UI (Import > Clipboard) without error on Node-RED 3.x.
- [ ] `ollama-generate` accepts a `msg.payload` string and returns `msg.payload` as the LLM response string within 60 seconds for a 7B model on reference hardware (8 GB RAM).
- [ ] All subflows set `msg.error` and nullify `msg.payload` on HTTP error; they do not throw uncaught exceptions.
- [ ] All subflows are included in the bundled `flows.json` Jinja2 template in `p4n4-cli`.
- [ ] A smoke-test flow `ai-integration-smoke.json` is committed to `p4n4-iot/config/node-red/` that wires an inject node → `ollama-generate` → debug node and verifies a non-empty response.

---

### F-0.2.3 — n8n Workflow Library Expansion

**Repo:** `p4n4-ai`
**Description:** Build a library of eight n8n workflows covering the primary AI-IoT integration patterns. Four workflows serve as the baseline IoT integration tier; four are AI-augmented extensions.

**Workflow library (v0.2 target):**

| File                          | Trigger               | AI Component      | Output                       |
|-------------------------------|-----------------------|-------------------|------------------------------|
| `alert-enrichment.json`       | MQTT alert topic      | Ollama            | Enriched alert message       |
| `scheduled-digest.json`       | Cron                  | Ollama            | Digest via webhook           |
| `device-onboarding.json`      | MQTT new-device event | (none)            | Device registry entry        |
| `incident-escalation.json`    | Threshold exceeded    | Ollama            | Routed escalation            |
| `anomaly-investigation.json`  | Letta alert event     | Ollama + Letta    | Root-cause memo              |
| `sensor-qa.json`              | Webhook (NL query)    | Ollama + InfluxDB | Natural language answer      |
| `model-drift-report.json`     | Cron (weekly)         | Ollama            | Drift summary (Markdown)     |
| `operator-briefing.json`      | Cron (daily)          | Letta (persona)   | Morning status brief         |

**Acceptance Criteria:**

- [ ] All 8 workflow JSON files pass a round-trip import via `docker compose exec n8n n8n import:workflow --input=<file>` without error.
- [ ] All workflows reference credentials by type/name only; no hardcoded tokens or passwords.
- [ ] Each workflow file includes a top-level `"notes"` field describing purpose, trigger condition, and expected outputs.
- [ ] `anomaly-investigation.json` sends a Letta event, receives a Letta agent response, then calls Ollama to synthesise a root-cause note — all within a single n8n workflow execution without manual intervention.

---

### F-0.2.4 — `p4n4 ei` Subcommand Expansion

**Repo:** `p4n4-cli`
**Description:** Extend the `p4n4 ei` command surface to cover the full Edge Impulse model lifecycle, including listing local models, running inference, and querying runner metadata.

**Complete command surface (v0.2 target):**

| Command                   | Description                                                  |
|---------------------------|--------------------------------------------------------------|
| `p4n4 ei deploy MODEL`    | Copy `.eim` to models dir; restart runner container          |
| `p4n4 ei run`             | Start the Edge Impulse runner container                      |
| `p4n4 ei status`          | Print runner container status                                |
| `p4n4 ei list`            | List `.eim` model files in `edge-impulse/models/`            |
| `p4n4 ei infer FILE`      | Send a sample file to runner `:8080/api/v1/infer`; print result |
| `p4n4 ei update`          | Pull latest Edge Impulse Linux Runner Docker image           |
| `p4n4 ei info`            | Print runner version, loaded model filename, and health status |

**Inference API contract (Edge Impulse runner at `:8080`):**

```
POST /api/v1/infer
Content-Type: multipart/form-data

Form fields:
  file: <binary — raw sensor CSV or image frame>

Response 200 (application/json):
{
  "result": {
    "classification": {
      "<label>": <float — confidence 0.0 to 1.0>,
      ...
    }
  },
  "timing": {
    "dsp": <int — ms>,
    "classification": <int — ms>,
    "anomaly": <int — ms>
  }
}
```

**Acceptance Criteria:**

- [ ] `p4n4 ei list` prints a Rich table with columns: filename, file size, last modified; returns a zero-row table with an informational message when no `.eim` files are found.
- [ ] `p4n4 ei infer sample.csv` sends the file to `:8080/api/v1/infer` and prints classification labels ranked by confidence score.
- [ ] `p4n4 ei info` prints runner container image tag, currently mounted `.eim` filename, and HTTP health check status.
- [ ] `p4n4 ei update` wraps `docker pull edgeimpulse/linux-runner:latest` and prints the pulled image digest.
- [ ] All `p4n4 ei` subcommands exit non-zero and print a descriptive error when the edge stack is not running.

---

### F-0.2.5 — Template Registry v2

**Repo:** `p4n4-cli`, `p4n4-templates`
**Description:** Extend the template command surface. `p4n4 template install` (v0.1 alias) becomes `p4n4 template pull` (aligned with Git/container vocabulary). Adds `push` for template submission and `search` with tag filtering. Registry index schema is bumped to v2.

**Complete command surface:**

| Command                          | Description                                                  |
|----------------------------------|--------------------------------------------------------------|
| `p4n4 template search [QUERY]`   | List registry templates; filter by keyword or `--tag`        |
| `p4n4 template pull NAME`        | Clone template into current project                          |
| `p4n4 template push PATH`        | Validate local template; output GitHub PR stub URL           |
| `p4n4 template list`             | Show templates applied to current project                    |
| `p4n4 template info NAME`        | Print template metadata from index                           |
| `p4n4 template org add URL`      | Register a private org index URL                             |
| `p4n4 template org list`         | List registered org index URLs                               |

**`index.json` v2 schema:**

```json
{
  "org": "<string>",
  "index_version": "2",
  "templates": {
    "<name>": {
      "repo": "<string — git URL>",
      "subdir": "<string — path within repo>",
      "description": "<string>",
      "tags": ["<string>"],
      "latest": "<string — semver>",
      "author": "<string>",
      "license": "<string — SPDX identifier>",
      "min_hardware": "<string — e.g. rpi4 | rpi5 | nuc | gpu-workstation>",
      "requires_stacks": ["iot", "ai", "edge"]
    }
  }
}
```

**Acceptance Criteria:**

- [ ] `p4n4 template search` fetches `index.json` from the configured registry URL and displays results in a Rich table with columns: name, description, tags, version, author.
- [ ] `p4n4 template search --tag manufacturing` filters results to templates containing that tag.
- [ ] `p4n4 template pull factory-baseline` clones the template subdir, renders Jinja2 templates, and merges generated files into the project directory without overwriting an existing `.env`.
- [ ] `p4n4 template push ./my-template` validates `p4n4-template.toml`, runs the same validation script used in CI, and prints a pre-filled GitHub PR URL for the user to complete submission.
- [ ] `p4n4 template org add <url>` appends the URL to the `template_orgs` array in `.p4n4.json`.
- [ ] Registry fetch failures (timeout, DNS error) produce a clear error message with a suggestion to use `--offline` to load a cached index snapshot.

---

### F-0.2.6 — Official Community Templates

**Repo:** `p4n4-templates`
**Description:** Publish at least three community-quality templates that exercise v0.2 features and serve as reference implementations for template authors.

**Initial official templates:**

| Template Name      | Stacks Required  | Description                                              |
|--------------------|------------------|----------------------------------------------------------|
| `factory-baseline` | iot + ai + edge  | Discrete manufacturing: vibration/temp anomaly detection |
| `iot-minimal`      | iot              | Minimal sensor monitoring with no AI components          |
| `greenhouse-control`| iot + ai        | Soil/humidity sensors + AI-generated grow cycle reports  |

**Acceptance Criteria:**

- [ ] All three templates pass `validate_templates.py` CI without errors.
- [ ] Each template's `p4n4-template.toml` declares `requires.cli = ">=0.2.0"`.
- [ ] The `greenhouse-control` template includes at least one pre-built Node-RED flow that uses the `ollama-generate` subflow from F-0.2.2.
- [ ] All template `README.md` files document expected MQTT topics, required environment variables, and (if applicable) the required Edge Impulse model.

---

### 5.2 Milestone: v0.2.0 "Intelligence Layer"

**Milestone is closed when ALL of the following are true:**

- [ ] All six features (F-0.2.1 through F-0.2.6) have all acceptance criteria met.
- [ ] `p4n4-cli`: `__version__` and `pyproject.toml` bumped to `0.2.0`.
- [ ] `p4n4-cli`: `CHANGELOG.md` updated.
- [ ] `p4n4-cli`: `compat.py` updated with `"ai": ">=0.2.0"`.
- [ ] `p4n4-ai`: tagged `v0.2.0`.
- [ ] `p4n4-templates`: `index.json` updated to `index_version: "2"`.
- [ ] All test requirements in Section 5.3 pass on Python 3.11, 3.12, and 3.13.
- [ ] All v0.1 test requirements (T-0.1.1 through T-0.1.8) still pass.
- [ ] `p4n4-docs`: this document updated — v0.2 status changed to `COMPLETE`.
- [ ] GitHub Release created with notes.

**v0.2.x patch releases** address bugs that do not require new features. `compat.py` is only updated if a stack patch introduces a breaking configuration change.

---

### 5.3 v0.2 Test Requirements

| Test ID   | Scope                   | Type        | Command / Assertion                                      |
|-----------|-------------------------|-------------|----------------------------------------------------------|
| T-0.2.1   | Letta persona init      | Integration | `pytest test_agent_personas.py` (mocked Letta API)       |
| T-0.2.2   | ollama-generate subflow | Integration | Node-RED test harness; assert `msg.payload` non-empty    |
| T-0.2.3   | n8n workflow import     | Integration | `n8n import:workflow`; validate all 8 JSON files         |
| T-0.2.4   | `p4n4 ei list` empty    | Unit        | `pytest test_ei.py::test_list_empty`                     |
| T-0.2.5   | `p4n4 ei infer` mock    | Unit        | `pytest test_ei.py::test_infer_mock_runner`              |
| T-0.2.6   | Template search output  | Unit        | `pytest test_template.py::test_search_returns_table`     |
| T-0.2.7   | Template pull no env    | Unit        | `pytest test_template.py::test_pull_no_env_overwrite`    |
| T-0.2.8   | index.json v2 schema    | Static      | `validate_index.py` (CI on p4n4-templates)               |
| T-0.2.9   | p4n4-template.toml      | Static      | `validate_templates.py` (CI on all 3 new templates)      |
| T-0.2.10  | v0.1 full suite         | Unit        | `pytest tests/` (all T-0.1.x must still pass)            |

---

## 6. Phase 3: Scale & Extensibility (v0.3+) — FUTURE

Phase 3 feature specifications are written as detailed briefs. Precise acceptance criteria will be finalized as Phase 2 completes. Interface contracts are defined here to guide architectural decisions made during Phase 2 implementation.

### 6.1 Release Summary

| Feature ID | Name                           | Repo(s)                       | Status |
|------------|--------------------------------|-------------------------------|--------|
| F-0.3.1    | Multi-site federation          | p4n4-ai, p4n4-templates       | Future |
| F-0.3.2    | OpenTelemetry integration      | p4n4-iot, p4n4-ai             | Future |
| F-0.3.3    | Embedding pipeline             | p4n4-ai, p4n4-cli             | Future |
| F-0.3.4    | Optional Kafka/NATS layer      | p4n4-iot, p4n4-cli            | Future |
| F-0.3.5    | Kubernetes/Helm chart          | p4n4-helm (new)               | Future |
| F-0.3.6    | p4n4-flows standalone library  | p4n4-flows (new)              | Future |
| F-0.3.7    | Shell completions              | p4n4-cli                      | Future |

**Milestone: v0.3.0 "Scale & Extensibility"** — Deliverable: Multi-site federation, OpenTelemetry support, embedding pipeline, optional Kafka/NATS, Kubernetes/Helm chart, `p4n4-flows` library, shell completions. Integration tests pass on at least one ARM64 target (Raspberry Pi 5 or equivalent).

---

### F-0.3.1 — Multi-Site Federation

**Repo:** `p4n4-ai`, `p4n4-templates`
**Description:** Enable multiple p4n4 deployments ("sites") to share telemetry and AI insights. Implemented as a set of n8n workflows and Node-RED flows that bridge MQTT topics and InfluxDB instances across sites via a designated "federation hub" deployment. Federation is opt-in and does not affect standard single-site deployments.

**Federation topology:**

```
   Site A (spoke)               Federation Hub               Site B (spoke)
   ┌──────────────┐             ┌───────────────┐            ┌──────────────┐
   │ Mosquitto    │──MQTT──────►│ n8n federate  │◄──MQTT─────│ Mosquitto    │
   │ InfluxDB     │◄──Flux──────│ InfluxDB sync │────────────►│ InfluxDB     │
   │ Letta agent  │             │ Letta (global)│             │ Letta agent  │
   └──────────────┘             └───────────────┘             └──────────────┘
```

**`.p4n4.json` federation extension (v0.3):**

```json
{
  "federation": {
    "role": "hub | spoke",
    "hub_url": "<string — MQTT URL of hub, required for spokes>",
    "site_id": "<string — unique per site>",
    "topics_forwarded": ["sensors/#", "alerts/#"]
  }
}
```

**Draft acceptance criteria:**

- [ ] A spoke instance publishes all topics matching `topics_forwarded` to the hub's Mosquitto broker.
- [ ] The hub's InfluxDB stores telemetry tagged with `site_id`.
- [ ] A unified Grafana dashboard at the hub visualises metrics from all connected sites.
- [ ] `p4n4 federation status` prints connected spokes, last-seen timestamp, and forwarded topic count.
- [ ] Federation is opt-in; absent a `federation` key in `.p4n4.json`, all behaviour is identical to v0.2.

---

### F-0.3.2 — OpenTelemetry Integration

**Repo:** `p4n4-iot`, `p4n4-ai`
**Description:** Export stack-level observability signals (traces, metrics, logs) to an OpenTelemetry Collector service. Primarily for operators running p4n4 in semi-production environments needing inter-service latency visibility and error rate tracking.

**Signals scope:**

| Service    | Signal Type | What is exported                          |
|------------|-------------|-------------------------------------------|
| Node-RED   | Traces      | Flow execution time per flow ID           |
| Node-RED   | Metrics     | Messages/sec per topic, error rate        |
| n8n        | Traces      | Workflow execution duration               |
| n8n        | Metrics     | Workflow success/failure counts           |
| Ollama     | Metrics     | Inference latency (p50/p95), tokens/s     |
| Letta      | Logs        | Agent event log forwarded to OTEL         |

**OTEL Collector compose service (added to `p4n4-iot` override):**

```yaml
otel-collector:
  image: otel/opentelemetry-collector-contrib:latest
  ports:
    - "4317:4317"   # OTLP gRPC
    - "4318:4318"   # OTLP HTTP
  volumes:
    - ./config/otel/collector.yaml:/etc/otelcol-contrib/config.yaml
  networks:
    - p4n4-net
```

**Draft acceptance criteria:**

- [ ] OTEL Collector is enabled via `p4n4 up --otel` and disabled by default.
- [ ] Node-RED emits trace spans for each flow execution via `node-red-contrib-otel`.
- [ ] Grafana is pre-configured with a Tempo datasource to visualise OTEL traces.

---

### F-0.3.3 — Embedding Pipeline

**Repo:** `p4n4-ai`, `p4n4-cli`
**Description:** Enable the Anomaly Analyst and Operator Assistant Letta personas to perform semantic search over historical telemetry. InfluxDB data is periodically embedded via `nomic-embed-text` (running in Ollama) and inserted into Letta archival memory. This extends the agent's reasoning context without expanding the active LLM context window.

**Pipeline:**

```
InfluxDB (hourly cron in n8n)
   │  Flux query → JSON rows
   ▼
n8n: embedding-pipeline.json
   │  stringify rows → text chunk
   ▼
Ollama POST /api/embeddings (model: nomic-embed-text)
   │  768-dim embedding vector
   ▼
Letta POST /v1/agents/:id/archival/insert
   │  text + embedding stored in archival memory
   ▼
Operator Assistant
   └── semantic search on archival memory during query handling
```

**Draft acceptance criteria:**

- [ ] `nomic-embed-text` model is pulled automatically when the AI stack starts (v0.3+).
- [ ] The embedding pipeline n8n workflow (`embedding-pipeline.json`) runs on a configurable cron schedule via `EMBED_CRON` env var (default: hourly).
- [ ] `p4n4 ai memory stats` prints: agent name, archival entry count, oldest/newest entry date.
- [ ] Archival memory for the Operator Assistant persona is queryable via natural language through `p4n4 ai agent chat operator-assistant`.

---

### F-0.3.4 — Optional Kafka/NATS Layer

**Repo:** `p4n4-iot`, `p4n4-cli`
**Description:** For high-throughput deployments (>10,000 messages/second), provide an optional message queue tier between MQTT ingestion and InfluxDB writes. The standard direct path (Mosquitto → Node-RED → InfluxDB) remains the default and is unchanged.

**Topology option A — Kafka:**

```
Mosquitto → Kafka Connect (MQTT source) → Kafka topic
                                        → Node-RED (Kafka consumer) → InfluxDB
```

**Topology option B — NATS JetStream:**

```
Mosquitto → NATS bridge → NATS JetStream subject
                        → Node-RED (NATS consumer) → InfluxDB
```

**Draft acceptance criteria:**

- [ ] Either mode is activated by `p4n4 up --messaging kafka` or `--messaging nats`.
- [ ] Standard mode (no flag) is unaffected and default.
- [ ] Kafka/NATS layer sustains 10,000 messages/second under benchmark conditions with 0 message drops.
- [ ] `p4n4 status --messaging` includes queue lag and throughput metrics when a messaging layer is active.

---

### F-0.3.5 — Kubernetes/Helm Chart

**Repo:** `p4n4-helm` (new repository)
**Description:** Helm chart for deploying the full p4n4 platform on Kubernetes. Phase 3 scope is a functional chart for single-namespace deployment. Multi-namespace and multi-cluster are Phase 4+.

**Chart structure:**

```
p4n4-helm/
├── Chart.yaml
├── values.yaml                  # defaults mirror compose stack defaults
├── templates/
│   ├── namespace.yaml
│   ├── p4n4-net-networkpolicy.yaml
│   ├── mosquitto/
│   ├── influxdb/
│   ├── nodered/
│   ├── grafana/
│   ├── ollama/
│   ├── letta/
│   ├── n8n/
│   └── edge-impulse/
└── ci/
    └── values-ci.yaml
```

**Draft acceptance criteria:**

- [ ] `helm install p4n4 ./p4n4-helm` deploys all services in a `p4n4` namespace on k3s or Kubernetes 1.28+.
- [ ] All services pass readiness probes within 5 minutes on reference hardware.
- [ ] `values.yaml` exposes all configuration variables present in compose stack `.env` files.
- [ ] `helm uninstall p4n4` cleanly removes all Kubernetes resources.
- [ ] CI runs `helm lint` and `helm template` on every PR to `p4n4-helm`.

---

### F-0.3.6 — `p4n4-flows` Standalone Node-RED Library

**Repo:** `p4n4-flows` (new repository)
**Description:** Extract the Node-RED AI palette subflows (from F-0.2.2) and base IoT flows into a versioned community library. Flows are distributed as importable JSON collections and are installable via `p4n4 flows pull <name>`.

**Flows registry schema (`flows-index.json`):**

```json
{
  "index_version": "1",
  "flows": {
    "<name>": {
      "repo": "<string — git URL>",
      "subdir": "<string — path within repo>",
      "description": "<string>",
      "node_red_min": "<string — semver, e.g. 3.0>",
      "tags": ["<string>"]
    }
  }
}
```

**Draft acceptance criteria:**

- [ ] `p4n4 flows pull ollama-generate` imports the subflow JSON into the current project's `config/node-red/` directory.
- [ ] `p4n4 flows list` fetches `flows-index.json` and displays available flows in a Rich table.
- [ ] All subflows from F-0.2.2 are published to `p4n4-flows` and reference their minimum Node-RED version.

---

### F-0.3.7 — Shell Completions

**Repo:** `p4n4-cli`
**Description:** Auto-generate shell completion scripts for bash, zsh, and fish via Typer's built-in completion mechanism. No hand-maintained completion files.

**Interface:**

```bash
p4n4 --install-completion bash
p4n4 --install-completion zsh
p4n4 --install-completion fish
```

**Draft acceptance criteria:**

- [ ] Completions for all top-level commands and their subcommands are generated correctly.
- [ ] Stack names (`iot`, `ai`, `edge`) are completed for `p4n4 up`, `p4n4 down`, `p4n4 logs`, `p4n4 add`, `p4n4 remove`.
- [ ] Completions are automatically generated by Typer; no manually maintained completion file exists in the repo.
- [ ] Installation instructions are documented in `cli-reference.md`.

---

### 6.2 v0.3 Test Requirements

| Test ID   | Scope                    | Type        | Assertion                                            |
|-----------|--------------------------|-------------|------------------------------------------------------|
| T-0.3.1   | Federation spoke/hub     | Integration | Two p4n4 instances exchange MQTT and InfluxDB data   |
| T-0.3.2   | OTEL trace export        | Integration | Trace spans appear in OTEL Collector log             |
| T-0.3.3   | Embedding pipeline       | Integration | Archival memory entry count increases after pipeline run |
| T-0.3.4   | Kafka/NATS throughput    | Load        | 10,000 msgs/s benchmark; 0 messages dropped          |
| T-0.3.5   | Helm chart install       | Integration | All pods reach Ready state within 5 min on k3s       |
| T-0.3.6   | `p4n4 flows pull`        | Unit        | `pytest test_flows.py`                               |
| T-0.3.7   | Shell completion bash    | Smoke       | Install completion; source; tab-complete `p4n4 up <TAB>` |

---

## 7. v1.0 Production Stability — FUTURE

v1.0 is the first release designated "production-ready." The following criteria define what that means for p4n4.

### 7.1 Production Readiness Criteria

| Criterion                   | Requirement                                                       |
|-----------------------------|-------------------------------------------------------------------|
| Test coverage               | ≥ 80% line coverage on `p4n4-cli` test suite                     |
| Reference uptime            | IoT stack runs 30 days without manual intervention in CI soak     |
| Security                    | No known critical CVEs in bundled service images                  |
| Documentation completeness  | All commands, config variables, and API endpoints documented      |
| OS compatibility matrix     | Ubuntu 22.04, Debian 12, Raspberry Pi OS 64-bit, macOS 14        |
| Python matrix               | Passes on Python 3.11, 3.12, 3.13                                 |
| Breaking change policy      | Semver strictly observed; deprecation warnings for 1 minor release before removal |
| Release notes               | Full `CHANGELOG.md` entry for every release                       |
| ARM64 CI                    | Official CI matrix includes `ubuntu-24.04-arm` runner            |

### 7.2 v1.0 Feature Additions

These features are not in v0.3 and are required for the v1.0 designation:

| Feature                   | Description                                                             |
|---------------------------|-------------------------------------------------------------------------|
| TLS wizard                | `p4n4 init` option to auto-generate self-signed certs or configure ACME via Caddy |
| `p4n4 doctor`             | Environment diagnostic: Docker version, RAM, GPU presence, network reachability |
| `p4n4-lib` Python package | Shared client code extracted from CLI; published separately to PyPI    |
| `p4n4-api` REST gateway   | HTTP API at `:8000` exposing all p4n4 lifecycle operations             |
| `p4n4-dashboard` web UI   | Basic project management and stack health visualisation                |
| ARM64 CI testing          | GitHub Actions matrix includes `ubuntu-24.04-arm` for all test suites |

---

## 8. Technical Specifications

### 8.1 Project Manifest Schema (`.p4n4.json`)

Full JSON Schema (draft-07) for `manifest-v1.json`, covering fields through v0.2 with the v0.3 federation extension marked as optional:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://raisga.github.io/p4n4-docs/schemas/manifest-v1.json",
  "title": "p4n4 Project Manifest",
  "type": "object",
  "required": ["name", "version", "cli_version", "stacks", "created_at"],
  "additionalProperties": false,
  "properties": {
    "$schema":      { "type": "string" },
    "name":         { "type": "string", "minLength": 1 },
    "version":      { "type": "string", "pattern": "^\\d+\\.\\d+\\.\\d+$" },
    "cli_version":  { "type": "string", "pattern": "^\\d+\\.\\d+\\.\\d+$" },
    "stacks": {
      "type": "array",
      "items": { "type": "string", "enum": ["iot", "ai", "edge"] },
      "uniqueItems": true
    },
    "created_at":   { "type": "string", "format": "date-time" },
    "template":     { "type": "string" },
    "template_orgs": {
      "type": "array",
      "items": { "type": "string", "format": "uri" }
    },
    "federation": {
      "type": "object",
      "required": ["role", "site_id"],
      "properties": {
        "role":             { "type": "string", "enum": ["hub", "spoke"] },
        "hub_url":          { "type": "string", "format": "uri" },
        "site_id":          { "type": "string", "minLength": 1 },
        "topics_forwarded": { "type": "array", "items": { "type": "string" } }
      }
    }
  }
}
```

### 8.2 Template Descriptor Schema (`p4n4-template.toml`)

```toml
# Required fields
[template]
name        = "<string>"           # unique identifier, kebab-case
version     = "<string>"           # semver
description = "<string>"           # one-line description
author      = "<string>"
license     = "<string>"           # SPDX identifier, e.g. "MIT"
tags        = ["<string>"]         # list of searchable tags

[template.requires]
cli         = ">=0.2.0"            # minimum p4n4 CLI version
stacks      = ["iot", "ai"]        # stacks this template provisions

# Optional fields
[template.hardware]
min_ram_gb  = 4                    # minimum RAM in GB
min_disk_gb = 10                   # minimum disk in GB
gpu         = false                # whether a GPU is required

[template.variables]
# Template-specific Jinja2 variables with defaults
# DEVICE_TOPIC = { description = "Root MQTT topic prefix", default = "sensors" }
```

### 8.3 Template Index Schema (`index.json`)

**v1 (current):**

```json
{
  "org": "<string>",
  "index_version": "1",
  "templates": {
    "<name>": {
      "repo": "<string — git URL>",
      "subdir": "<string>",
      "description": "<string>",
      "tags": ["<string>"],
      "latest": "<string — semver>"
    }
  }
}
```

**v2 (target for v0.2, adds `author`, `license`, `min_hardware`, `requires_stacks`):**

```json
{
  "org": "<string>",
  "index_version": "2",
  "templates": {
    "<name>": {
      "repo": "<string>",
      "subdir": "<string>",
      "description": "<string>",
      "tags": ["<string>"],
      "latest": "<string>",
      "author": "<string>",
      "license": "<string>",
      "min_hardware": "<string — e.g. rpi4 | rpi5 | nuc | gpu-workstation>",
      "requires_stacks": ["iot", "ai", "edge"]
    }
  }
}
```

### 8.4 CLI Exit Codes

| Code | Meaning                                                      |
|------|--------------------------------------------------------------|
| 0    | Success                                                      |
| 1    | General error (unexpected exception)                         |
| 2    | User error (invalid arguments, missing `.p4n4.json`)         |
| 3    | Docker error (compose config invalid, Docker daemon not running) |
| 4    | Network error (`p4n4-net` missing, service unreachable)      |
| 5    | Validation error (missing required `.env` variables)         |
| 6    | Compatibility error (stack version outside compatible range) |

### 8.5 Service API Endpoints Referenced by CLI and Flows

| Service    | Method | Path                                  | Used By               | Purpose                   |
|------------|--------|---------------------------------------|-----------------------|---------------------------|
| Ollama     | POST   | `/api/generate`                       | Node-RED subflow      | LLM text generation       |
| Ollama     | POST   | `/api/chat`                           | Node-RED subflow      | Chat completion           |
| Ollama     | POST   | `/api/embeddings`                     | n8n embedding wf      | Text embedding            |
| Ollama     | GET    | `/api/tags`                           | `p4n4 ai agent init`  | List available models     |
| Letta      | POST   | `/v1/agents`                          | `p4n4 ai agent init`  | Create agent              |
| Letta      | GET    | `/v1/agents`                          | `p4n4 ai agent list`  | List agents               |
| Letta      | POST   | `/v1/agents/:id/messages`             | Node-RED subflow      | Send message to agent     |
| Letta      | GET    | `/v1/agents/:id/memory`               | Node-RED subflow      | Read agent memory         |
| Letta      | POST   | `/v1/agents/:id/archival/insert`      | n8n embedding wf      | Insert archival entry     |
| InfluxDB   | POST   | `/api/v2/write`                       | Node-RED              | Write line protocol data  |
| InfluxDB   | POST   | `/api/v2/query`                       | Node-RED, n8n         | Execute Flux query        |
| EI Runner  | POST   | `/api/v1/infer`                       | `p4n4 ei infer`       | Run inference on sample   |
| EI Runner  | GET    | `/api/v1/info`                        | `p4n4 ei info`        | Runner metadata and model |

### 8.6 MQTT Topic Conventions

Standard topic structure used across all official p4n4 templates and flows:

| Topic Pattern                        | Direction       | Purpose                          |
|--------------------------------------|-----------------|----------------------------------|
| `sensors/<device-id>/<measurement>`  | device → broker | Telemetry publish                |
| `commands/<device-id>/<action>`      | broker → device | Actuation commands               |
| `alerts/<device-id>/<alert-type>`    | Node-RED → n8n  | Alert events                     |
| `status/<device-id>/online`          | device → broker | Heartbeat (retained message)     |
| `inference/<device-id>/result`       | EI runner → NR  | Edge Impulse classification result |
| `federation/<site-id>/sensors/#`     | spoke → hub     | Federated telemetry (v0.3)       |

### 8.7 Environment Variable Reference

All stack environment variables, their owning stack, and whether they are cross-stack shared secrets:

| Variable                    | Owner Stack | Used By                       | Cross-Stack Secret |
|-----------------------------|-------------|-------------------------------|-------------------|
| `INFLUXDB_ADMIN_TOKEN`      | p4n4-iot    | Node-RED, Grafana, n8n        | Yes               |
| `INFLUXDB_ORG`              | p4n4-iot    | Node-RED, Grafana, n8n        | No                |
| `INFLUXDB_BUCKET`           | p4n4-iot    | Node-RED, Grafana, n8n        | No                |
| `INFLUXDB_ADMIN_USER`       | p4n4-iot    | InfluxDB                      | No                |
| `INFLUXDB_ADMIN_PASSWORD`   | p4n4-iot    | InfluxDB                      | No                |
| `MQTT_USER`                 | p4n4-iot    | Node-RED, Mosquitto ACL       | Yes               |
| `MQTT_PASSWORD`             | p4n4-iot    | Node-RED, Mosquitto ACL       | Yes               |
| `GF_SECURITY_ADMIN_PASSWORD`| p4n4-iot    | Grafana                       | No                |
| `N8N_BASIC_AUTH_USER`       | p4n4-ai     | n8n                           | No                |
| `N8N_BASIC_AUTH_PASSWORD`   | p4n4-ai     | n8n                           | No                |
| `N8N_ENCRYPTION_KEY`        | p4n4-ai     | n8n                           | No                |
| `LETTA_SERVER_PASSWORD`     | p4n4-ai     | Letta, Node-RED               | Yes               |
| `EI_API_KEY`                | p4n4-edge   | Edge Impulse Runner           | No                |

---

## 9. Repository & CI Reference

### 9.1 Per-Repository CI Matrix

| Repository        | Trigger              | CI Jobs                                                   |
|-------------------|----------------------|-----------------------------------------------------------|
| `p4n4-iot`        | push, PR             | `docker compose config --quiet`, `yamllint .`, `check_env_example.py` |
| `p4n4-ai`         | push, PR             | `docker compose config --quiet`, `yamllint .`, `check_env_example.py` |
| `p4n4-edge`       | push, PR             | `docker compose config --quiet`, `yamllint .`, `check_env_example.py` |
| `p4n4-cli`        | push, PR             | `ruff check .`, `pytest` (Python 3.11, 3.12, 3.13 matrix) |
| `p4n4-cli`        | tag push `v*`        | `python -m build`, `pypi-publish` (trusted publisher)     |
| `p4n4-templates`  | push, PR             | `validate_index.py`, `validate_templates.py`              |
| `p4n4-docs`       | push to `main`       | `mkdocs gh-deploy`                                        |
| `p4n4-helm`       | push, PR (v0.3+)     | `helm lint`, `helm template`                              |

### 9.2 Release Checklist

Use this checklist for each platform release. Instantiate it in the GitHub Release notes.

```
Pre-release
[ ] All feature acceptance criteria for this version are verified
[ ] All test requirements for this version pass on Python 3.11/3.12/3.13
[ ] No regressions in prior-version test suite

CLI repo (p4n4-cli)
[ ] __version__ bumped in pyproject.toml and __init__.py
[ ] CHANGELOG.md updated with release notes
[ ] compat.py updated if stack compatibility range changed
[ ] All CI checks green on main

Stack repos (as applicable)
[ ] p4n4-iot tagged vX.Y.Z
[ ] p4n4-ai tagged vX.Y.Z
[ ] p4n4-edge tagged vX.Y.Z

Registry (as applicable)
[ ] p4n4-templates index.json updated (schema version bump if applicable)
[ ] New template README files reviewed

Publish
[ ] git tag vX.Y.Z && git push origin vX.Y.Z
[ ] PyPI publish.yml workflow triggered and completed successfully
[ ] pip install p4n4==X.Y.Z smoke-tested

Post-release
[ ] GitHub Release created with changelog notes
[ ] p4n4-docs specs.md status column updated to COMPLETE
[ ] p4n4-docs mkdocs deploy triggered
[ ] Announcement posted on relevant channels
```

---

## 10. Appendix

### 10.1 Glossary

| Term | Definition |
|---|---|
| `.eim` | Edge Impulse model binary format; the compiled, deployable artifact for the Linux Runner |
| `p4n4-net` | Docker bridge network (`172.20.0.0/16`) created by `p4n4-iot`; all stacks share it |
| MING | Acronym for the IoT stack services: Mosquitto, InfluxDB, Node-RED, Grafana |
| GenAI stack | The `p4n4-ai` stack: Ollama, Letta, n8n |
| Edge AI stack | The `p4n4-edge` stack: Edge Impulse Linux Runner |
| Letta archival memory | Long-term, semantically searchable memory storage in a Letta agent; distinct from in-context memory |
| Flux | InfluxDB's functional query language; used for all InfluxDB reads in p4n4 |
| Subflow | A reusable, named group of nodes in Node-RED that can be instantiated anywhere in a flow |
| Community template | A shareable project scaffold distributed via `p4n4-templates` registry and installable via `p4n4 template pull` |
| Org index | A private `index.json`-compatible registry hosted by an organisation; registered via `p4n4 template org add` |
| Federation hub | A p4n4 deployment configured as the central aggregation point for multi-site topologies |
| Federation spoke | A p4n4 deployment that forwards telemetry and events to a federation hub |
| OTEL | OpenTelemetry; the CNCF observability framework for traces, metrics, and logs |
| `compat.py` | Module in `p4n4-cli` that defines the minimum stack versions compatible with each CLI release |

---

### 10.2 Feature ID Index

| Feature ID | Name                             | Section | Phase | Status  |
|------------|----------------------------------|---------|-------|---------|
| F-0.1.4    | CLI core commands                | 4       | 1     | Planned |
| F-0.1.6    | Project manifest                 | 4       | 1     | Planned |
| F-0.1.7    | PyPI publish                     | 4       | 1     | Planned |
| F-0.1.1    | p4n4-iot compose stack           | 4       | 1     | Planned |
| F-0.1.2    | p4n4-ai compose stack            | 4       | 1     | Planned |
| F-0.1.3    | p4n4-edge compose stack          | 4       | 1     | Planned |
| F-0.1.5    | Secret management                | 4       | 1     | Planned |
| F-0.2.1    | Letta agent personas             | 5       | 2     | Future  |
| F-0.2.2    | Node-RED AI palette              | 5       | 2     | Future  |
| F-0.2.3    | n8n workflow library             | 5       | 2     | Future  |
| F-0.2.4    | p4n4 ei subcommands              | 5       | 2     | Future  |
| F-0.2.5    | Template registry v2             | 5       | 2     | Future  |
| F-0.2.6    | Official community templates     | 5       | 2     | Future  |
| F-0.3.1    | Multi-site federation            | 6       | 3     | Future   |
| F-0.3.2    | OpenTelemetry integration        | 6       | 3     | Future   |
| F-0.3.3    | Embedding pipeline               | 6       | 3     | Future   |
| F-0.3.4    | Optional Kafka/NATS layer        | 6       | 3     | Future   |
| F-0.3.5    | Kubernetes/Helm chart            | 6       | 3     | Future   |
| F-0.3.6    | p4n4-flows standalone library    | 6       | 3     | Future   |
| F-0.3.7    | Shell completions                | 6       | 3     | Future   |

---

### 10.3 Test ID Index

| Test ID   | Name                          | Section | Phase | Type        |
|-----------|-------------------------------|---------|-------|-------------|
| T-0.1.1   | CLI init                      | 4.3     | 1     | Unit        |
| T-0.1.2   | Manifest read/write           | 4.3     | 1     | Unit        |
| T-0.1.3   | Secret generation             | 4.3     | 1     | Unit        |
| T-0.1.4   | Validate command              | 4.3     | 1     | Unit        |
| T-0.1.5   | Compose valid                 | 4.3     | 1     | Integration |
| T-0.1.6   | YAML lint                     | 4.3     | 1     | Static      |
| T-0.1.7   | .env coverage                 | 4.3     | 1     | Static      |
| T-0.1.8   | PyPI install smoke            | 4.3     | 1     | Smoke       |
| T-0.2.1   | Letta persona init            | 5.3     | 2     | Integration |
| T-0.2.2   | ollama-generate subflow       | 5.3     | 2     | Integration |
| T-0.2.3   | n8n workflow import           | 5.3     | 2     | Integration |
| T-0.2.4   | `p4n4 ei list` empty          | 5.3     | 2     | Unit        |
| T-0.2.5   | `p4n4 ei infer` mock          | 5.3     | 2     | Unit        |
| T-0.2.6   | Template search output        | 5.3     | 2     | Unit        |
| T-0.2.7   | Template pull no env override | 5.3     | 2     | Unit        |
| T-0.2.8   | index.json v2 schema          | 5.3     | 2     | Static      |
| T-0.2.9   | p4n4-template.toml validation | 5.3     | 2     | Static      |
| T-0.2.10  | v0.1 regression suite         | 5.3     | 2     | Unit        |
| T-0.3.1   | Federation spoke/hub          | 6.2     | 3     | Integration |
| T-0.3.2   | OTEL trace export             | 6.2     | 3     | Integration |
| T-0.3.3   | Embedding pipeline            | 6.2     | 3     | Integration |
| T-0.3.4   | Kafka/NATS throughput         | 6.2     | 3     | Load        |
| T-0.3.5   | Helm chart install            | 6.2     | 3     | Integration |
| T-0.3.6   | `p4n4 flows pull`             | 6.2     | 3     | Unit        |
| T-0.3.7   | Shell completion bash         | 6.2     | 3     | Smoke       |

---

### 10.4 Document Revision History

| Version    | Author  | Change                                                              |
|------------|---------|---------------------------------------------------------------------|
| 0.1-draft  | raisga  | Initial draft — all phases, v0.1 through v1.0                      |
| 0.1-draft2 | raisga  | Reset to greenfield: v0.1 PLANNED, v0.2/v0.3 FUTURE; CLI-first priority; all acceptance criteria open |
