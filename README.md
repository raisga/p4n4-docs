# p4n4

> EdgeAI + GenAI Integration Platform
> Version 0.1 | 2026

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Background & Motivation](#2-background--motivation)
3. [System Overview](#3-system-overview)
4. [IoT Stack Components](#4-iot-stack-components-p4n4-iot)
5. [GenAI Stack Components](#5-genai-stack-components-p4n4-ai)
6. [Integration Architecture](#6-integration-architecture)
7. [Deployment & Operations](#7-deployment--operations)
8. [Security](#8-security)
9. [p4n4 CLI — Project Scaffolding Tool](#9-p4n4-cli--project-scaffolding-tool)
10. [p4n4 Template Registry](#10-p4n4-template-registry)
11. [Roadmap](#11-roadmap)
12. [Appendix](#12-appendix)

---

## 1. Executive Summary

**p4n4** (pronounced *"pana"*) is a Docker Compose-based open platform stack for small-to-medium IoT deployments augmented with a first-class Generative AI layer. It takes direct inspiration from the IoTStack TIGUITTO pattern — Telegraf, InfluxDB, Grafana, Mosquitto — but replaces Telegraf with **Node-RED** as the primary data-flow orchestrator, and adds a dedicated **GenAI stack** comprising **n8n**, **Letta**, and **Ollama**.

The name *p4n4* reflects the two complementary four-service stacks at the heart of the platform.

### Design Philosophy

| Principle | Description |
|-----------|-------------|
| **Open Source First** | Every component is freely available and self-hostable |
| **Composable** | Stacks can be deployed independently or together |
| **Docker-native** | Single `docker-compose.yml` per stack with clean overrides |
| **Edge & Cloud Ready** | Runs on Raspberry Pi class hardware up to cloud VMs |
| **AI-Augmented IoT** | Sensor data and AI reasoning exist in the same data fabric |

---

## 2. Background & Motivation

### 2.1 IoTStack and the TIGUITTO Pattern

IoTStack popularized the **TIGUITTO** stack — Telegraf, InfluxDB, Grafana, and Mosquitto — as a proven, containerized IoT data pipeline. The pattern is elegant: MQTT devices publish sensor data to Mosquitto; Telegraf subscribes and writes to InfluxDB; Grafana visualises. It is battle-tested, lightweight, and declarative.

However, TIGUITTO has a central limitation: **Telegraf is a metrics-scraping agent, not a general-purpose automation runtime.** It has no built-in conditional logic, HTTP call-outs, device actuation, or AI integration. As IoT projects grow in complexity — particularly when combining sensor telemetry with intelligent reasoning — the gap between "data collection" and "intelligent response" becomes critical.

### 2.2 Why Node-RED Replaces Telegraf

Node-RED is a flow-based, visual programming environment built on Node.js. Originally developed at IBM for IoT wiring, it is now a de facto standard in the maker and industrial IoT communities.

**What Node-RED adds over Telegraf:**

- Native MQTT subscribe/publish with full topic routing control
- Visual flow editor accessible via browser — no code required for simple pipelines
- Hundreds of community nodes: HTTP, WebSocket, OPC-UA, Modbus, BACnet, InfluxDB, and more
- JavaScript function nodes for arbitrary transformation logic
- **Bidirectional** data flow — ingest sensor data *and* actuate devices
- Built-in dashboard capability (`node-red-dashboard`) for lightweight UIs
- Direct HTTP calls to the GenAI stack from within flows

Node-RED can replicate everything Telegraf does (MQTT → InfluxDB), while additionally enabling conditional logic, multi-protocol bridging, device actuation, API integration, and direct AI calls — all in a single runtime.

### 2.3 The GenAI Stack Opportunity

The rapid maturation of local LLM inference (Ollama), agentic memory frameworks (Letta), and AI workflow automation (n8n) creates a compelling opportunity: **IoT telemetry can now be semantically understood, summarised, and acted upon by AI agents running entirely on-premise.**

p4n4 treats the GenAI stack not as a separate product but as an integral layer of the IoT platform — connected through Node-RED as the shared data and control plane.

---

## 3. System Overview

### 3.1 Stack Topology

p4n4 is organized into two cooperating stacks sharing a Docker bridge network (`p4n4-net`):

```
┌─────────────────────────────┐    ┌─────────────────────────────┐
│      IoT Stack (p4n4-iot)   │    │    GenAI Stack (p4n4-ai)    │
│                             │    │                             │
│  🔴 Mosquitto  MQTT Broker  │    │  🔴 Ollama    LLM Inference │
│  🟠 Node-RED   Orchestrator │◄──►│  🟠 Letta     Agent Memory  │
│  🔵 InfluxDB   Time-Series  │    │  🔵 n8n       AI Workflows  │
│  🟢 Grafana    Dashboards   │    │                             │
└─────────────────────────────┘    └─────────────────────────────┘
              │                                  │
              └──────────── p4n4-net ────────────┘
```

Node-RED sits at the intersection of both stacks. It subscribes to Mosquitto, normalises and routes data to InfluxDB, and can simultaneously call n8n webhooks or directly query Ollama/Letta APIs when AI reasoning is needed.

### 3.2 High-Level Data Flow

```
IoT Devices
    │  MQTT publish
    ▼
Mosquitto (broker)
    │  MQTT subscribe
    ▼
Node-RED (flow engine)
    ├──► InfluxDB          (write telemetry)
    ├──► Ollama            (real-time AI inference)
    ├──► Letta             (agent event logging)
    └──► n8n               (trigger complex workflows)
          │
          ├──► InfluxDB    (historical queries)
          ├──► Ollama      (reasoning chains)
          └──► Letta       (agent memory read/write)

Grafana ◄──── InfluxDB     (dashboard queries)
Grafana ────► n8n          (alert webhooks)
```

---

## 4. IoT Stack Components (p4n4-iot)

### 4.1 Mosquitto — MQTT Broker

Eclipse Mosquitto is the message backbone of the IoT stack, implementing MQTT 3.1.1 and 5.0 as the central publish/subscribe hub for all device telemetry, commands, and status events.

| Parameter | Value | Notes |
|-----------|-------|-------|
| Docker Image | `eclipse-mosquitto:2.x` | Stable LTS release |
| MQTT Port | `1883` (TCP) | Standard unencrypted MQTT |
| MQTT/TLS Port | `8883` (TCP) | TLS-enabled, cert-based auth |
| WebSocket Port | `9001` (WS) | Browser-based MQTT clients |
| Config Volume | `/mosquitto/config/` | `mosquitto.conf`, `passwd`, `acl` |
| Data Volume | `/mosquitto/data/` | Persistent message store |
| Log Volume | `/mosquitto/log/` | Operational logs |

**Key configuration decisions:**

- Authentication via password file (`mosquitto_passwd`) with ACL rules per topic namespace
- Retained messages enabled for device last-will and status topics
- QoS 1 minimum required for all telemetry topics (at-least-once delivery)
- Topic convention: `{site}/{device_type}/{device_id}/{measurement}`  
  e.g. `factory/temp_sensor/T001/celsius`

---

### 4.2 Node-RED — Flow Orchestrator

Node-RED is the operational heart of p4n4. It replaces Telegraf as the primary data-movement engine, adding visual programmability, conditional routing, and AI bridge capability. Flows are defined in JSON and version-controlled alongside the stack.

| Parameter | Value | Notes |
|-----------|-------|-------|
| Docker Image | `nodered/node-red:latest` | Alpine-based, minimal footprint |
| UI Port | `1880` (HTTP) | Flow editor and dashboard |
| Data Volume | `/data/` | `flows.json`, settings, credentials |
| Base Nodes | `mqtt`, `influxdb`, `http`, `function`, `dashboard` | Core capability set |
| Community Nodes | `node-red-contrib-ollama`, `node-red-contrib-letta` | AI integration |
| Network | `p4n4-net` (bridge) | Shared with all stack services |

**Core flow patterns:**

```
Telemetry Ingest:
  [MQTT In] → [Parse/Transform] → [InfluxDB Out]

Device Command:
  [HTTP In] → [Validate] → [MQTT Out]

Anomaly Detection:
  [MQTT In] → [Threshold Node] → [Ollama HTTP] → [Alert MQTT Out]

AI Digest:
  [Timer] → [InfluxDB Query] → [n8n Webhook] → [Summary to Dashboard]

Agent Interaction:
  [HTTP In from UI] → [Letta API] → [HTTP Response]
```

**Security:** Node-RED editor access is protected by username/password in `settings.js`. Credentials (MQTT passwords, API keys) are stored in the encrypted Node-RED credential store. Optionally exposed via Nginx reverse proxy with TLS termination.

---

### 4.3 InfluxDB — Time-Series Database

InfluxDB v2 provides persistent time-series storage for all sensor data ingested by Node-RED. It offers a purpose-built query language (Flux), push-down aggregations, data retention policies, and a REST API consumed by both Grafana and the GenAI stack.

| Parameter | Value | Notes |
|-----------|-------|-------|
| Docker Image | `influxdb:2.x` | InfluxDB v2 OSS |
| API Port | `8086` (HTTP) | Flux queries, write API, UI |
| Data Volume | `/var/lib/influxdb2/` | Persistent bucket storage |
| Config Volume | `/etc/influxdb2/` | `influxdb.conf` |
| Auth | Token-based (Bearer) | Per-bucket read/write tokens |
| Query Language | Flux | Time-series pipeline language |

**Bucket design:**

| Bucket | Retention | Purpose |
|--------|-----------|---------|
| `raw_telemetry` | 30 days | All inbound sensor readings |
| `processed_metrics` | 365 days | Downsampled/aggregated data |
| `ai_events` | Indefinite | AI-generated annotations, anomaly flags, agent logs |
| `system_health` | 7 days | Node-RED and stack component health metrics |

---

### 4.4 Grafana — Visualisation & Alerting

Grafana serves as the primary human-facing dashboard layer. It queries InfluxDB using Flux, renders time-series panels, and fires alerts via email, Slack, webhooks, or n8n. Pre-provisioned with datasources and a base dashboard via config volumes.

| Parameter | Value | Notes |
|-----------|-------|-------|
| Docker Image | `grafana/grafana-oss:latest` | Open Source edition |
| UI Port | `3000` (HTTP) | Dashboard and administration |
| Data Volume | `/var/lib/grafana/` | Persistent dashboards, users |
| Provisioning | `/etc/grafana/provisioning/` | Auto-configure datasources & dashboards |
| Alerting | Grafana Unified Alerting | Webhook to n8n supported |
| Plugins | `grafana-clock-panel`, `marcusolsson-json-datasource` | Extended panel types |

Grafana alert rules fire webhooks into n8n, enabling AI-enriched notifications — for example, prompting Ollama to generate a human-readable summary of an anomaly before notifying on-call staff.

---

## 5. GenAI Stack Components (p4n4-ai)

The GenAI stack provides local AI inference, persistent agent memory, and workflow automation. All three services are self-hosted, privacy-preserving, and network-isolated except for explicit bridges to the IoT stack via Node-RED and n8n.

---

### 5.1 Ollama — Local LLM Inference

Ollama provides an OpenAI-compatible local inference runtime for large language models. It handles model downloading, quantisation, and serving via a simple REST API. In p4n4, Ollama is the AI reasoning engine that Node-RED and Letta communicate with for natural language generation, summarisation, classification, and anomaly explanation.

| Parameter | Value | Notes |
|-----------|-------|-------|
| Docker Image | `ollama/ollama:latest` | CPU + GPU (CUDA/ROCm) builds available |
| API Port | `11434` (HTTP) | OpenAI-compatible `/api/generate`, `/api/chat` |
| Model Volume | `/root/.ollama/` | Persistent model cache (can be 10s of GB) |
| GPU Support | NVIDIA via `--gpus all` / AMD ROCm | Optional; falls back to CPU |
| Default Models | `llama3`, `mistral`, `phi3` | Configurable via pull scripts |

**Recommended models by hardware profile:**

| Model | Parameters | Best For |
|-------|-----------|----------|
| `phi3:mini` | 3.8B | Raspberry Pi 5, Jetson Nano (edge) |
| `mistral:7b` | 7B | Mid-range x86 servers |
| `llama3:8b` | 8B | Cloud VMs, GPU workstations |
| `nomic-embed-text` | — | Embedding pipeline for semantic search |

---

### 5.2 Letta — Agent Memory & Personas

Letta (formerly MemGPT) is an open-source agent framework providing persistent, structured memory for AI agents. Unlike a stateless LLM call, Letta agents maintain a memory hierarchy:

- **Core memory** — always in context (device identity, site config, operator preferences)
- **Archival memory** — vector-searchable long-term store (maintenance history, anomaly patterns)
- **Recall memory** — conversation and event history

In p4n4, Letta agents serve as persistent *"site intelligence"* entities that remember device histories, maintenance events, operator interactions, and learned anomaly patterns.

| Parameter | Value | Notes |
|-----------|-------|-------|
| Docker Image | `letta/letta:latest` | Self-hosted server mode |
| API Port | `8283` (HTTP) | REST API for agent management & messaging |
| UI | `8283` (HTTP) | Letta ADE (Agent Development Environment) |
| Database | PostgreSQL + pgvector | Archival memory vector store |
| LLM Backend | Ollama (local) or OpenAI API | Configurable per agent |
| Auth | Bearer token | Per-deployment token |

**Agent personas in p4n4:**

| Agent | Role |
|-------|------|
| **Site Monitor** | One per physical site; tracks device health, acknowledges alerts, logs maintenance history |
| **Anomaly Analyst** | Called by Node-RED when readings exceed thresholds; provides natural-language explanation |
| **Operator Assistant** | User-facing chatbot with memory of past conversations and site context |
| **Report Agent** | Triggered weekly by n8n; queries InfluxDB summary, generates management report |

---

### 5.3 n8n — AI Workflow Automation

n8n is an open-source, node-based workflow automation platform — fully self-hosted and with deep AI integration. In p4n4, n8n serves as the high-level workflow orchestrator, handling complex multi-step automations that involve external APIs, notifications, scheduling, and AI reasoning chains.

It complements Node-RED: **Node-RED owns real-time sensor data flow; n8n owns business logic workflows triggered by IoT events.**

| Parameter | Value | Notes |
|-----------|-------|-------|
| Docker Image | `n8nio/n8n:latest` | Community edition |
| UI Port | `5678` (HTTP) | Workflow editor and execution log |
| Data Volume | `/home/node/.n8n/` | Workflows, credentials, execution history |
| Database | SQLite (default) / PostgreSQL (production) | Workflow metadata store |
| Trigger Types | Webhook, Cron, MQTT, HTTP, Email | Rich trigger ecosystem |
| AI Nodes | Langchain, OpenAI, Ollama (via HTTP) | Built-in AI chain support |

**Key workflow patterns:**

```
Alert Enrichment:
  [Grafana Webhook]
    → [InfluxDB: query recent readings]
    → [Ollama: explain anomaly in plain English]
    → [Slack/Email notification with AI context]

Scheduled Digest:
  [Cron: daily 08:00]
    → [InfluxDB: 24h summary query]
    → [Letta Agent: generate management report]
    → [Email to operators]

Device Onboarding:
  [HTTP trigger from Node-RED]
    → [Create Letta Agent persona for new device]
    → [Register MQTT topics]
    → [Confirm back to Node-RED]

Incident Escalation:
  [n8n: detect repeated alert]
    → [Letta: log incident to archival memory]
    → [n8n: create ticket in external ITSM]
```

---

## 6. Integration Architecture

### 6.1 Network Design

All containers share a single Docker bridge network: `p4n4-net`. Service discovery uses Docker DNS (container name as hostname). External access is controlled via port mapping and an optional Nginx reverse proxy.

```yaml
networks:
  p4n4-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

Internal DNS examples:
- `http://influxdb:8086`
- `http://ollama:11434`
- `http://letta:8283`
- `http://n8n:5678`

GenAI services have no direct inbound ports from the internet — accessible only via Node-RED or n8n on the internal network.

### 6.2 Service Communication Matrix

| From | To | Protocol | Purpose |
|------|----|----------|---------|
| IoT Devices | Mosquitto | MQTT :1883 | Publish sensor telemetry, subscribe commands |
| Node-RED | Mosquitto | MQTT :1883 | Subscribe all telemetry topics |
| Node-RED | InfluxDB | HTTP :8086 | Write processed metrics (Flux write API) |
| Node-RED | Ollama | HTTP :11434 | Direct LLM inference for real-time flows |
| Node-RED | Letta | HTTP :8283 | Send events to persistent agents |
| Node-RED | n8n | HTTP :5678 | Trigger complex multi-step workflows |
| Grafana | InfluxDB | HTTP :8086 | Dashboard queries (Flux) |
| Grafana | n8n | HTTP :5678 | Alert webhook trigger |
| n8n | InfluxDB | HTTP :8086 | Historical data queries for workflows |
| n8n | Ollama | HTTP :11434 | LLM reasoning in workflow chains |
| n8n | Letta | HTTP :8283 | Agent memory read/write in workflows |
| Letta | Ollama | HTTP :11434 | LLM backend for agent inference |

---

## 7. Deployment & Operations

### 7.1 Repository Structure

```
p4n4/
├── docker-compose.iot.yml          # IoT stack
├── docker-compose.ai.yml           # GenAI stack
├── docker-compose.override.yml     # Local overrides (GPU, ports, dev)
├── .env                            # Secrets & config (never commit)
├── .env.example                    # Template for .env
│
├── mosquitto/
│   └── config/
│       ├── mosquitto.conf
│       ├── passwd
│       └── acl
│
├── node-red/
│   ├── flows.json                  # Version-controlled flow definitions
│   └── settings.js
│
├── influxdb/
│   └── config/influxdb.conf
│
├── grafana/
│   └── provisioning/
│       ├── datasources/influxdb.yaml
│       └── dashboards/p4n4-base.json
│
├── ollama/
│   └── models/                     # Model pull scripts
│
├── letta/
│   └── config/letta.conf
│
└── n8n/
    └── workflows/                  # Exportable workflow JSON files
```

### 7.2 Quick Start

```bash
# 1. Clone and configure
git clone https://github.com/your-org/p4n4.git && cd p4n4
cp .env.example .env
# Edit .env with your credentials, tokens, and secrets

# 2. Start IoT stack
docker compose -f docker-compose.iot.yml up -d

# 3. Start GenAI stack
docker compose -f docker-compose.ai.yml up -d

# 4. Pull Ollama models
docker exec ollama ollama pull phi3:mini

# 5. Import Node-RED flows
# Access http://localhost:1880 → Import → node-red/flows.json

# 6. Activate n8n workflows
# Access http://localhost:5678 → Import from n8n/workflows/
```

**Service URLs after startup:**

| Service | URL | Default Credentials |
|---------|-----|---------------------|
| Node-RED | http://localhost:1880 | Set in `settings.js` |
| Grafana | http://localhost:3000 | `admin` / set in `.env` |
| InfluxDB | http://localhost:8086 | Token in `.env` |
| n8n | http://localhost:5678 | Set in `.env` |
| Letta ADE | http://localhost:8283 | Token in `.env` |
| Ollama API | http://localhost:11434 | No auth (internal only) |

### 7.3 Hardware Profiles

| Profile | Hardware | Stacks | Recommended Models |
|---------|----------|--------|--------------------|
| Edge Minimal | Raspberry Pi 5 (8GB) | IoT only | N/A |
| Edge AI | Raspberry Pi 5 + Coral/Hailo | IoT + GenAI (CPU) | `phi3:mini` |
| Mid-range Server | Intel NUC / x86 (16GB RAM) | Both | `mistral:7b` |
| GPU Workstation | NVIDIA RTX 3080+ (16GB VRAM) | Both | `llama3:8b` |
| Cloud VM | AWS t3.xlarge / GCP n2-standard-4 | Both | `mistral:7b` or API |

---

## 8. Security

### 8.1 Authentication & Access Control

| Service | Auth Method |
|---------|------------|
| Mosquitto | Username/password (`mosquitto_passwd`) + ACL file |
| Node-RED | Bcrypt-hashed admin credentials in `settings.js`; encrypted credential store |
| InfluxDB | Token-based; separate read-only and read-write tokens per bucket |
| Grafana | Local admin + optional OAuth2/LDAP |
| n8n | `N8N_BASIC_AUTH_*` env vars; webhook URLs include secret tokens |
| Ollama | No built-in auth — network-isolated on `p4n4-net`, not exposed externally |
| Letta | Bearer token; API not exposed externally |

### 8.2 Network Security

- All inter-service communication is on the Docker bridge network (`p4n4-net`) — no public IP exposure by default
- External access to Node-RED, Grafana, and n8n should be via **Nginx reverse proxy with TLS termination**
- MQTT over TLS (port 8883) recommended for any device communicating over untrusted networks
- Secrets managed via `.env` file — never committed to version control; use a secrets manager in production

### 8.3 Nginx Reverse Proxy (recommended)

```nginx
server {
    listen 443 ssl;
    server_name p4n4.example.com;

    location /nodered/  { proxy_pass http://localhost:1880/; }
    location /grafana/   { proxy_pass http://localhost:3000/; }
    location /n8n/       { proxy_pass http://localhost:5678/; }
}
```

---

## 9. p4n4 CLI — Project Scaffolding Tool

### 9.1 Overview

`p4n4` is a command-line tool that scaffolds, configures, and manages a p4n4 project on any target machine. It is the primary entry point for operators and developers — replacing the need to manually copy files, edit YAML, and remember Docker Compose invocation order.

It is intentionally modelled on the developer experience of tools like `create-react-app`, `cookiecutter`, and `ansible-pull`: answer a few questions, get a working project directory, bring it up with a single command.

The CLI is implemented in **Python** (single-file install via `pipx` or `pip`) with no runtime dependencies beyond Docker and Docker Compose on the host.

```
$ pip install p4n4
$ p4n4 --help

Usage: p4n4 [OPTIONS] COMMAND [ARGS]...

  p4n4 — IoT + GenAI + Edge ML platform scaffolding tool.

Options:
  --version   Show version and exit.
  --help      Show this message and exit.

Commands:
  init        Scaffold a new p4n4 project interactively.
  add         Add a stack or service to an existing project.
  remove      Remove a stack or service from an existing project.
  up          Start one or more stacks.
  down        Stop one or more stacks.
  status      Show running services and health.
  logs        Tail logs for a service.
  secret      Generate and rotate secrets in .env.
  ei          Manage Edge Impulse models and inference containers.
  validate    Check project config for errors before starting.
  upgrade     Pull latest service images and restart.
  template    Pull, push, search, and manage project templates.
```

---

### 9.2 Design Principles

| Principle | Implementation |
|-----------|---------------|
| **Interactive by default, scriptable on demand** | `init` uses prompts; all prompts can be bypassed with flags for CI/CD use |
| **Never overwrite user files** | Regenerating config merges changes; existing files are backed up with `.bak` suffix |
| **Secrets are never printed** | `p4n4 secret` writes directly to `.env`; values are not echoed to stdout |
| **Composable stacks** | Each stack is independently addable/removable; the tool manages cross-stack dependencies |
| **Idempotent** | Running the same command twice leaves the project in the same state |
| **No hidden state** | All configuration lives in the project directory; the tool is stateless |

---

### 9.3 `p4n4 init` — Interactive Scaffolding

The primary command. Walks the operator through a guided wizard that produces a complete, ready-to-run project directory.

```
$ p4n4 init

  ██████╗ ██╗  ██╗███╗  ██╗██╗  ██╗
  ██╔══██╗██║  ██║████╗ ██║██║  ██║
  ██████╔╝███████║██╔██╗██║███████║
  ██╔═══╝ ╚════██║██║╚████║╚════██║
  ██║           ██║██║ ╚███║      ██║
  ╚═╝           ╚═╝╚═╝  ╚══╝      ╚═╝
  Platform Stack Scaffolding Tool v0.2

? Project name:  my-factory-stack
? Project directory [./my-factory-stack]:
? Site identifier (used in MQTT topics):  factory-floor-1

─── Stacks ───────────────────────────────────────────────────────────
? Include IoT stack? (Mosquitto, Node-RED, InfluxDB, Grafana)  [Y/n]: Y
? Include GenAI stack? (Ollama, Letta, n8n)  [y/N]: y
? Include Edge Impulse inference stack?  [y/N]: y

─── IoT Stack ────────────────────────────────────────────────────────
? MQTT username:  p4n4mqtt
? Node-RED admin username:  admin
? InfluxDB organisation:  p4n4
? InfluxDB primary bucket [raw_telemetry]:

─── GenAI Stack ──────────────────────────────────────────────────────
? Default Ollama model:
  ❯ phi3:mini    (3.8B — recommended for edge/Pi)
    mistral:7b   (7B — mid-range server)
    llama3:8b    (8B — GPU workstation / cloud)
    skip         (pull manually later)
? GPU acceleration for Ollama?  [y/N]: n

─── Edge Impulse Stack ───────────────────────────────────────────────
? Add an Edge Impulse inference model?  [Y/n]: Y
? Model name (used as container name):  vibration-fault
? Model source:
  ❯ .eim file path  (recommended for production, no cloud dependency)
    Edge Impulse API key  (fetches latest model at container start)
? Path to .eim file:  ./models/motor-fault-v3.eim
? Host port for inference HTTP server [1337]:
? Add another model?  [y/N]: N

─── Secrets ──────────────────────────────────────────────────────────
  Generating secrets...  ✓ MQTT password
                         ✓ InfluxDB admin token
                         ✓ Grafana admin password
                         ✓ n8n encryption key
                         ✓ Letta server token

─── Scaffold ─────────────────────────────────────────────────────────
  Writing project files...

  my-factory-stack/
  ├── .env                            ✓
  ├── .env.example                    ✓
  ├── .gitignore                      ✓
  ├── docker-compose.iot.yml          ✓
  ├── docker-compose.ai.yml           ✓
  ├── docker-compose.edge.yml         ✓
  ├── mosquitto/config/               ✓  (mosquitto.conf, passwd, acl)
  ├── node-red/flows.json             ✓  (base IoT + EI flows)
  ├── node-red/settings.js            ✓
  ├── influxdb/config/influxdb.conf   ✓
  ├── grafana/provisioning/           ✓  (datasource + base dashboard)
  ├── letta/config/letta.conf         ✓
  ├── n8n/workflows/                  ✓  (alert enrichment, digest)
  └── edge-impulse/
      └── models/motor-fault-v3.eim   ✓  (copied from source)

─── Done ─────────────────────────────────────────────────────────────
  ✓ Project scaffolded at ./my-factory-stack

  Next steps:
    cd my-factory-stack
    p4n4 up --all          # bring up all stacks
    p4n4 status            # verify services are healthy
```

#### Non-interactive / scriptable mode

All prompts can be supplied as flags for use in CI/CD pipelines or automated provisioning:

```bash
p4n4 init \
  --name my-factory-stack \
  --site factory-floor-1 \
  --stacks iot,ai,edge \
  --ollama-model phi3:mini \
  --ei-model vibration-fault:./models/motor-fault-v3.eim@1337 \
  --no-gpu \
  --non-interactive
```

---

### 9.4 `p4n4 add` — Add a Stack or Service

Add a stack or individual service to an existing project without touching already-configured files.

```bash
# Add the GenAI stack to a project that was initialised with IoT only
p4n4 add stack ai

# Add a second Edge Impulse inference model
p4n4 add ei-model \
  --name audio-anomaly \
  --eim ./models/audio-anomaly-v1.eim \
  --port 1338

# Add a custom Node-RED community node to the container build
p4n4 add nodered-node node-red-contrib-modbus
```

The `add` command regenerates only the affected compose files and appends to `.env.example`. It never touches `.env` directly unless `--apply-secrets` is passed.

---

### 9.5 `p4n4 up` / `p4n4 down` — Stack Lifecycle

Wraps `docker compose` with awareness of stack interdependencies and correct startup order.

```bash
# Start all stacks (correct order: iot → ai → edge)
p4n4 up --all

# Start specific stacks only
p4n4 up iot ai

# Start with live log output (does not detach)
p4n4 up --all --follow

# Stop all stacks
p4n4 down --all

# Stop and remove volumes (destructive — prompts for confirmation)
p4n4 down --all --volumes
```

**Startup order enforcement:** the IoT stack (specifically Mosquitto and InfluxDB) must be healthy before the AI stack starts. The Edge Impulse stack can start independently. `p4n4 up` handles this sequencing automatically using Docker healthcheck polling rather than relying on `depends_on` alone.

---

### 9.6 `p4n4 status` — Health Overview

Prints a unified health view across all stacks and their services.

```
$ p4n4 status

  p4n4 — my-factory-stack @ factory-floor-1
  ─────────────────────────────────────────────────────────────────

  IoT Stack (p4n4-iot)
  ┌────────────────────┬──────────┬────────────┬─────────────────┐
  │ Service            │ Status   │ Uptime     │ Port(s)         │
  ├────────────────────┼──────────┼────────────┼─────────────────┤
  │ mosquitto          │ ✓ healthy │ 2d 14h     │ 1883, 8883      │
  │ node-red           │ ✓ healthy │ 2d 14h     │ 1880            │
  │ influxdb           │ ✓ healthy │ 2d 14h     │ 8086            │
  │ grafana            │ ✓ healthy │ 2d 14h     │ 3000            │
  └────────────────────┴──────────┴────────────┴─────────────────┘

  GenAI Stack (p4n4-ai)
  ┌────────────────────┬──────────┬────────────┬─────────────────┐
  │ Service            │ Status   │ Uptime     │ Port(s)         │
  ├────────────────────┼──────────┼────────────┼─────────────────┤
  │ ollama             │ ✓ healthy │ 2d 14h     │ 11434           │
  │ letta              │ ✓ healthy │ 2d 14h     │ 8283            │
  │ n8n                │ ✓ healthy │ 2d 14h     │ 5678            │
  └────────────────────┴──────────┴────────────┴─────────────────┘

  Edge ML Stack (p4n4-edge)
  ┌────────────────────┬──────────┬────────────┬─────────────────┐
  │ Service            │ Status   │ Uptime     │ Port(s)         │
  ├────────────────────┼──────────┼────────────┼─────────────────┤
  │ ei-vibration-fault │ ✓ healthy │ 2d 14h     │ 1337            │
  │   model: motor-fault-v3.eim   │ classes: normal, fault, unknown │
  └────────────────────┴──────────┴────────────┴─────────────────┘

  Access points:
    Node-RED  → http://localhost:1880
    Grafana   → http://localhost:3000
    n8n       → http://localhost:5678
    Letta ADE → http://localhost:8283
```

---

### 9.7 `p4n4 ei` — Edge Impulse Model Management

A dedicated subcommand for managing Edge Impulse inference containers and models.

```bash
# List all configured EI inference containers and their model info
p4n4 ei list

# Test inference on a model with a feature vector
p4n4 ei infer vibration-fault --features "0.12,-0.05,1.01,0.33,..."

# Test inference on a model with an image file
p4n4 ei infer vision-presence --image ./test-frame.jpg

# Update a model to a new .eim file (pulls container down, swaps file, restarts)
p4n4 ei update vibration-fault --eim ./models/motor-fault-v4.eim

# Fetch latest model from Edge Impulse Studio via API key
p4n4 ei update vibration-fault --api-key ei_abc123... --pull-latest

# Show model metadata (classes, input shape, DSP config)
p4n4 ei info vibration-fault
```

```
$ p4n4 ei list

  Edge Impulse Models
  ┌──────────────────────┬────────────────────────┬────────┬───────────────────────────┐
  │ Container            │ Model File             │ Port   │ Classes                   │
  ├──────────────────────┼────────────────────────┼────────┼───────────────────────────┤
  │ ei-vibration-fault   │ motor-fault-v3.eim     │ 1337   │ normal, fault, unknown    │
  │ ei-audio-anomaly     │ audio-anomaly-v1.eim   │ 1338   │ anomaly score (float)     │
  └──────────────────────┴────────────────────────┴────────┴───────────────────────────┘

$ p4n4 ei infer vibration-fault --features "0.12,-0.05,1.01,0.33"

  Inference result (ei-vibration-fault @ localhost:1337)
  ┌─────────────┬───────────┐
  │ Label       │ Score     │
  ├─────────────┼───────────┤
  │ normal      │ 0.964     │
  │ fault       │ 0.031     │
  │ unknown     │ 0.005     │
  ├─────────────┼───────────┤
  │ anomaly     │ -0.18     │
  └─────────────┴───────────┘
  Timing: dsp=2ms  classification=1ms
```

---

### 9.8 `p4n4 secret` — Secret Management

Generates cryptographically random secrets and writes them directly to `.env`. Secrets are never printed to the terminal.

```bash
# Generate all missing secrets (safe to run on an existing .env)
p4n4 secret generate

# Rotate a specific secret (restarts affected services)
p4n4 secret rotate influxdb-token --restart

# Show which secrets are set vs missing (values are masked)
p4n4 secret status
```

```
$ p4n4 secret status

  Secret Status — my-factory-stack/.env
  ┌───────────────────────────────┬────────┬──────────────────────────┐
  │ Key                           │ Status │ Used By                  │
  ├───────────────────────────────┼────────┼──────────────────────────┤
  │ MQTT_PASSWORD                 │ ✓ set  │ mosquitto, node-red      │
  │ INFLUXDB_ADMIN_TOKEN          │ ✓ set  │ influxdb, node-red, n8n  │
  │ GF_SECURITY_ADMIN_PASSWORD    │ ✓ set  │ grafana                  │
  │ N8N_ENCRYPTION_KEY            │ ✓ set  │ n8n                      │
  │ LETTA_SERVER_PASSWORD         │ ✓ set  │ letta                    │
  │ EI_API_KEY                    │ ─ unset│ edge-impulse (optional)  │
  └───────────────────────────────┴────────┴──────────────────────────┘
```

---

### 9.9 `p4n4 validate` — Pre-flight Checks

Validates the project configuration before attempting to start, catching common mistakes early.

```
$ p4n4 validate

  Validating my-factory-stack...

  ✓ docker available (27.3.1)
  ✓ docker compose available (v2.29.1)
  ✓ .env present and readable
  ✓ All required secrets set
  ✓ docker-compose.iot.yml — valid YAML, images resolve
  ✓ docker-compose.ai.yml  — valid YAML, images resolve
  ✓ docker-compose.edge.yml — valid YAML, images resolve
  ✓ Edge Impulse models present:
      motor-fault-v3.eim  (1.2 MB, aarch64)
      audio-anomaly-v1.eim (0.9 MB, aarch64)
  ✓ Port conflicts: none
  ✓ Node-RED flows.json — valid JSON
  ✓ Mosquitto passwd file — present and non-empty
  ✓ InfluxDB config — valid

  All checks passed. Ready to run: p4n4 up --all
```

---

### 9.10 `p4n4 upgrade` — Rolling Image Updates

Pulls latest images for all services and performs a rolling restart, minimising downtime.

```bash
# Upgrade all services
p4n4 upgrade --all

# Upgrade a specific stack
p4n4 upgrade iot

# Preview what would change without applying
p4n4 upgrade --all --dry-run
```

---

### 9.11 Implementation Details

#### Technology

| Aspect | Choice | Rationale |
|--------|--------|-----------|
| Language | Python 3.11+ | Widely available, excellent subprocess/YAML/JSON support |
| CLI framework | [Typer](https://typer.tiangolo.com/) + [Rich](https://github.com/Textualize/rich) | Ergonomic CLI with styled terminal output |
| Interactive prompts | [questionary](https://github.com/tmbo/questionary) | Arrow-key menus, validation, defaults |
| Template engine | [Jinja2](https://jinja.palletsprojects.com/) | Renders `docker-compose.yml`, `.env.example`, configs from templates |
| Secret generation | `secrets` (stdlib) | Cryptographically secure random token generation |
| Install method | `pipx install p4n4` | Isolated install, no virtualenv management for users |
| Package distribution | PyPI | `pip install p4n4` also works |

#### Project Template System

All scaffolded files are rendered from Jinja2 templates bundled with the CLI package. The template context is built from the answers collected in `p4n4 init` and stored in a `.p4n4.json` project manifest:

```json
{
  "version": "0.2",
  "project_name": "my-factory-stack",
  "site_id": "factory-floor-1",
  "stacks": ["iot", "ai", "edge"],
  "ollama_model": "phi3:mini",
  "gpu": false,
  "ei_models": [
    {
      "name": "vibration-fault",
      "eim": "edge-impulse/models/motor-fault-v3.eim",
      "port": 1337
    }
  ]
}
```

This manifest is the source of truth for subsequent `p4n4 add`, `p4n4 remove`, and `p4n4 upgrade` commands. It is checked into version control alongside the rest of the project.

#### Repository Layout (CLI package)

```
p4n4-cli/
├── p4n4/
│   ├── __init__.py
│   ├── cli.py              # Typer app, command registration
│   ├── commands/
│   │   ├── init.py         # p4n4 init wizard
│   │   ├── add.py          # p4n4 add
│   │   ├── lifecycle.py    # p4n4 up / down / status / logs
│   │   ├── ei.py           # p4n4 ei subcommands
│   │   ├── secret.py       # p4n4 secret
│   │   ├── validate.py     # p4n4 validate
│   │   ├── upgrade.py      # p4n4 upgrade
│   │   └── template.py     # p4n4 template subcommands
│   ├── scaffold/
│   │   ├── manifest.py     # .p4n4.json read/write
│   │   └── renderer.py     # Jinja2 template rendering
│   └── templates/
│       ├── docker-compose.iot.yml.j2
│       ├── docker-compose.ai.yml.j2
│       ├── docker-compose.edge.yml.j2
│       ├── env.j2
│       ├── env.example.j2
│       ├── mosquitto/
│       ├── node-red/
│       ├── grafana/
│       └── n8n/
├── tests/
├── pyproject.toml
└── README.md
```

---

### 9.12 Usage Examples — End to End

**Scenario A: Minimal IoT-only deployment on a Raspberry Pi 5**

```bash
pip install p4n4
p4n4 init --name factory-pi --stacks iot --non-interactive
cd factory-pi
p4n4 up iot
p4n4 status
```

**Scenario B: Full stack with Edge Impulse, scripted for automated provisioning**

```bash
p4n4 init \
  --name plant-monitor \
  --site plant-a \
  --stacks iot,ai,edge \
  --ollama-model phi3:mini \
  --ei-model vibration-fault:./models/motor-fault-v3.eim@1337 \
  --ei-model audio-anomaly:./models/audio-anomaly-v1.eim@1338 \
  --non-interactive

cd plant-monitor
p4n4 validate
p4n4 up --all
p4n4 status
```

**Scenario C: Adding a new EI model to a running project**

```bash
cd plant-monitor
p4n4 ei update vibration-fault --eim ./models/motor-fault-v4.eim
# → pulls container down, swaps .eim, restarts, confirms health
p4n4 ei info vibration-fault
```

---

## 10. p4n4 Template Registry

### 10.1 Overview

A p4n4 **template** is a versioned, shareable snapshot of a complete or partial project — compose files, Node-RED flows, n8n workflows, Grafana dashboards, Mosquitto configs, and Edge Impulse model references — packaged together and published to a Git repository. Templates are the primary mechanism for knowledge reuse across sites, teams, and organisations.

The template system is deliberately built on top of plain Git rather than a proprietary registry. This means any Git host works — GitHub, GitLab, Gitea, Bitbucket, a self-hosted bare repo — with no central server required. An **optional organisation index** layer sits on top, providing short-name resolution (`acme/factory-baseline`) for teams that want a curated catalogue without managing full URLs.

Templates are pulled and applied via `p4n4 template pull`, and published via `p4n4 template push`. They integrate directly with `p4n4 init` as a `--template` source, replacing the built-in defaults with the template's content.

---

### 10.2 Template Anatomy

A valid p4n4 template is a Git repository (or a subdirectory of one) containing a `p4n4-template.toml` manifest at the root. Everything else in the repository is scaffolding material.

```
my-factory-template/
├── p4n4-template.toml          ← required: template manifest
├── docker-compose.iot.yml.j2   ← Jinja2 template (rendered on pull)
├── docker-compose.ai.yml.j2
├── docker-compose.edge.yml.j2
├── mosquitto/
│   └── config/
│       ├── mosquitto.conf.j2
│       └── acl.j2
├── node-red/
│   └── flows.json              ← static file (copied as-is)
├── grafana/
│   └── provisioning/
│       └── dashboards/
│           └── factory-base.json
├── n8n/
│   └── workflows/
│       └── alert-enrichment.json
└── edge-impulse/
    └── models/
        └── .gitkeep            ← .eim files listed in manifest, not committed
```

#### `p4n4-template.toml` — Template Manifest

```toml
[template]
name        = "factory-baseline"
version     = "1.2.0"
description = "Full IoT + GenAI stack for discrete manufacturing. Includes vibration fault detection and operator assistant agent."
author      = "Acme Corp Platform Team"
license     = "MIT"
tags        = ["manufacturing", "vibration", "edge-impulse", "genai"]

[template.requires]
p4n4_cli    = ">=0.2.0"        # minimum CLI version
stacks      = ["iot", "ai", "edge"]

[template.variables]
# Variables exposed to the init wizard when this template is used.
# Each becomes a prompt unless passed via --var key=value.
site_id         = { prompt = "Site identifier (used in MQTT topics)", default = "site-1" }
ollama_model    = { prompt = "Ollama model", choices = ["phi3:mini", "mistral:7b", "llama3:8b"], default = "phi3:mini" }
influxdb_org    = { prompt = "InfluxDB organisation", default = "p4n4" }

[template.ei_models]
# Edge Impulse models referenced by this template.
# Operators are prompted for .eim paths or API keys on pull.
[template.ei_models.vibration-fault]
description = "Motor vibration fault classifier (normal / fault / unknown)"
port        = 1337
required    = true

[template.ei_models.audio-anomaly]
description = "Machine sound anomaly detector"
port        = 1338
required    = false

[template.files]
# Files rendered via Jinja2 (template variables available)
render = [
  "docker-compose.iot.yml.j2",
  "docker-compose.ai.yml.j2",
  "docker-compose.edge.yml.j2",
  "mosquitto/config/mosquitto.conf.j2",
  "mosquitto/config/acl.j2",
]
# Files copied verbatim
copy = [
  "node-red/flows.json",
  "grafana/provisioning/dashboards/factory-base.json",
  "n8n/workflows/alert-enrichment.json",
]
```

---

### 10.3 Name Resolution

`p4n4 template` accepts three forms of template identifier, resolved in this order:

```
1. Short name (org-scoped)     acme/factory-baseline
2. Full Git URL                https://github.com/acme/p4n4-template-factory.git
3. Local path                  ./my-local-template
```

#### Short Name Resolution

Short names are resolved against an **organisation index** — a lightweight JSON file hosted in a known Git repository. The CLI ships with the official p4n4 community index pre-configured, and teams can add their own private index.

```
Short name:   acme/factory-baseline
              │     │
              │     └── template name within the org index
              └──────── organisation slug (maps to an index URL)
```

If no org prefix is given, the CLI searches the community index:

```
factory-baseline        →  searches community index at
                           https://github.com/p4n4-org/templates/index.json
```

#### Organisation Index Format

An org index is a single `index.json` file in a Git repository:

```json
{
  "org": "acme",
  "index_version": "1",
  "templates": {
    "factory-baseline": {
      "repo": "https://github.com/acme/p4n4-template-factory.git",
      "description": "Full IoT + GenAI stack for discrete manufacturing",
      "tags": ["manufacturing", "vibration", "edge-impulse"],
      "latest": "1.2.0"
    },
    "warehouse-minimal": {
      "repo": "https://github.com/acme/p4n4-template-warehouse.git",
      "description": "Minimal IoT-only stack for warehouse monitoring",
      "tags": ["warehouse", "temperature", "humidity"],
      "latest": "0.4.1"
    }
  }
}
```

#### Adding an Organisation Index

```bash
# Register an org index (stored in ~/.p4n4/config.toml)
p4n4 template org add acme https://github.com/acme/p4n4-index.git

# List registered orgs
p4n4 template org list

# Remove an org
p4n4 template org remove acme
```

`~/.p4n4/config.toml`:

```toml
[orgs]
acme      = "https://github.com/acme/p4n4-index.git"
community = "https://github.com/p4n4-org/templates.git"   # default, always present
```

---

### 10.4 `p4n4 template pull` — Download and Apply a Template

Clones the template repository, renders Jinja2 files with project-specific variables, and writes output into the project directory (or the current directory for `init` integration).

```bash
# Pull by short name (community index)
p4n4 template pull factory-baseline

# Pull by org-scoped short name
p4n4 template pull acme/factory-baseline

# Pull a specific version tag
p4n4 template pull acme/factory-baseline@1.2.0

# Pull from a full Git URL
p4n4 template pull https://github.com/acme/p4n4-template-factory.git

# Pull from a full Git URL at a specific branch or tag
p4n4 template pull https://github.com/acme/p4n4-template-factory.git@v1.2.0

# Pull from a local directory (useful for template development)
p4n4 template pull ./my-local-template

# Pull and supply variables non-interactively
p4n4 template pull acme/factory-baseline \
  --var site_id=plant-a \
  --var ollama_model=mistral:7b \
  --non-interactive

# Pull into an existing project directory (merges, does not overwrite)
p4n4 template pull acme/factory-baseline --output ./my-existing-project
```

#### Pull Walkthrough

```
$ p4n4 template pull acme/factory-baseline

  Resolving acme/factory-baseline...
  ✓ Found in acme org index → https://github.com/acme/p4n4-template-factory.git
  ✓ Latest version: 1.2.0

  Cloning template...  ✓  (factory-baseline@1.2.0)

  Template: factory-baseline v1.2.0
  ─────────────────────────────────────────────────────────────────────
  Full IoT + GenAI stack for discrete manufacturing.
  Includes vibration fault detection and operator assistant agent.
  Author: Acme Corp Platform Team  |  License: MIT
  Tags:   manufacturing, vibration, edge-impulse, genai
  Stacks: iot, ai, edge

─── Variables ────────────────────────────────────────────────────────
? Site identifier (used in MQTT topics) [site-1]:  plant-a
? Ollama model:
  ❯ phi3:mini    (3.8B — recommended for edge/Pi)
    mistral:7b   (7B — mid-range server)
    llama3:8b    (8B — GPU workstation / cloud)
? InfluxDB organisation [p4n4]:

─── Edge Impulse Models ──────────────────────────────────────────────
  This template requires 1 Edge Impulse model.

? vibration-fault — Motor vibration fault classifier
  Model source:
  ❯ .eim file path
    Edge Impulse API key
? Path to .eim file:  ./models/motor-fault-v3.eim

  audio-anomaly — Machine sound anomaly detector  [optional, skipping]

─── Output ───────────────────────────────────────────────────────────
? Output directory [./factory-baseline]:  ./plant-a-stack

  Rendering templates...
  Copying static files...

  plant-a-stack/
  ├── p4n4-template.toml          ✓  (locked: acme/factory-baseline@1.2.0)
  ├── .p4n4.json                  ✓
  ├── .env                        ✓  (secrets generated)
  ├── .env.example                ✓
  ├── docker-compose.iot.yml      ✓
  ├── docker-compose.ai.yml       ✓
  ├── docker-compose.edge.yml     ✓
  ├── mosquitto/config/           ✓
  ├── node-red/flows.json         ✓
  ├── grafana/provisioning/       ✓
  ├── n8n/workflows/              ✓
  └── edge-impulse/models/        ✓

  ✓ Template applied at ./plant-a-stack

  Next steps:
    cd plant-a-stack
    p4n4 validate
    p4n4 up --all
```

#### Merge Behaviour on Existing Projects

When `--output` points to an existing project directory, the pull command performs a **merge**, not an overwrite:

| Situation | Behaviour |
|-----------|-----------|
| File exists, unchanged from previous template | Overwritten with new version |
| File exists, modified by operator | Skipped; diff saved to `<file>.template-update` |
| File exists in project, not in template | Left untouched |
| New file added in template | Written to project |
| Secret in `.env` already set | Never touched |

---

### 10.5 `p4n4 template push` — Publish a Template

Packages and pushes the current project (or a specified directory) as a template to a Git remote. Secrets and `.env` are always excluded. The `p4n4-template.toml` manifest must exist before pushing.

```bash
# Initialise a template manifest in the current project
p4n4 template init-manifest

# Push current project as a template to a Git remote
p4n4 template push https://github.com/acme/p4n4-template-factory.git

# Push and tag with a version
p4n4 template push https://github.com/acme/p4n4-template-factory.git --tag 1.3.0

# Push to a Git remote already configured in .p4n4.json
p4n4 template push --remote origin --tag 1.3.0

# Dry-run: show what would be pushed without pushing
p4n4 template push https://github.com/acme/p4n4-template-factory.git --dry-run
```

#### What Gets Pushed

The push command constructs a clean export of the project for use as a template:

- All files tracked by the project, **excluding**: `.env`, `*.eim`, `*.bak`, `__pycache__`, Docker runtime state
- `.gitignore` is applied — nothing ignored in the project is included in the template
- `.env` values are scrubbed; `.env.example` is always included
- `.eim` model files are listed by name in `p4n4-template.toml` under `[template.ei_models]` but the binaries are not pushed (operators provide them on pull)
- `flows.json` is included but any Node-RED credentials block is stripped

```
$ p4n4 template push https://github.com/acme/p4n4-template-factory.git --tag 1.3.0

  Preparing template export from ./plant-a-stack...

  Checking p4n4-template.toml...  ✓
  Scrubbing secrets from .env...  ✓  (8 values removed)
  Stripping Node-RED credentials... ✓
  Checking for uncommitted .eim files... ✓  (none found)
  Building file list...  21 files

  Files to push:
    p4n4-template.toml
    docker-compose.iot.yml.j2
    docker-compose.ai.yml.j2
    docker-compose.edge.yml.j2
    .env.example
    .gitignore
    mosquitto/config/mosquitto.conf.j2
    mosquitto/config/acl.j2
    node-red/flows.json
    grafana/provisioning/datasources/influxdb.yaml
    grafana/provisioning/dashboards/factory-base.json
    n8n/workflows/alert-enrichment.json
    n8n/workflows/digest.json
    ... (8 more)

? Confirm push to https://github.com/acme/p4n4-template-factory.git@1.3.0?  [Y/n]: Y

  Pushing...  ✓
  Tagging 1.3.0...  ✓

  ✓ Template published: acme/p4n4-template-factory @ 1.3.0
  → https://github.com/acme/p4n4-template-factory

  To register in your org index, run:
    p4n4 template org publish acme factory-baseline 1.3.0
```

---

### 10.6 `p4n4 template search` — Discover Templates

Searches across all registered org indexes and the community index.

```bash
# Search by keyword
p4n4 template search manufacturing

# Search by tag
p4n4 template search --tag edge-impulse

# Search within a specific org
p4n4 template search --org acme

# List all templates in the community index
p4n4 template search --all
```

```
$ p4n4 template search manufacturing

  Searching community index...  ✓
  Searching acme index...  ✓

  Results for "manufacturing"
  ─────────────────────────────────────────────────────────────────────────────
  acme/factory-baseline      v1.3.0   Full IoT + GenAI + EI for manufacturing
                                      Tags: manufacturing, vibration, edge-impulse
  acme/warehouse-minimal     v0.4.1   Minimal IoT-only warehouse monitoring
                                      Tags: warehouse, temperature, humidity
  community/smart-factory    v0.9.0   Community template: smart factory starter
                                      Tags: manufacturing, opc-ua, grafana
  ─────────────────────────────────────────────────────────────────────────────
  3 result(s) found.

  Pull a template:  p4n4 template pull <name>
```

---

### 10.7 `p4n4 template list` and `p4n4 template info`

```bash
# List all locally cached templates (previously pulled)
p4n4 template list

# Show detailed info about a template (reads from index, does not clone)
p4n4 template info acme/factory-baseline

# Show info about a template from a URL without pulling it
p4n4 template info https://github.com/acme/p4n4-template-factory.git
```

```
$ p4n4 template info acme/factory-baseline

  acme / factory-baseline  v1.2.0
  ─────────────────────────────────────────────────────────────────
  Full IoT + GenAI stack for discrete manufacturing.
  Author:   Acme Corp Platform Team
  License:  MIT
  Repo:     https://github.com/acme/p4n4-template-factory.git
  Tags:     manufacturing, vibration, edge-impulse, genai

  Stacks:         iot, ai, edge
  CLI required:   p4n4 >= 0.2.0

  Variables:
    site_id        Site identifier (used in MQTT topics)  [default: site-1]
    ollama_model   Ollama model                           [default: phi3:mini]
    influxdb_org   InfluxDB organisation                  [default: p4n4]

  Edge Impulse models:
    vibration-fault   Motor vibration fault classifier    [required]  :1337
    audio-anomaly     Machine sound anomaly detector      [optional]  :1338

  Versions:  1.0.0  1.1.0  1.2.0 (latest)

  Pull this template:
    p4n4 template pull acme/factory-baseline
```

---

### 10.8 `p4n4 template upgrade` — Update an Applied Template

When a template has been applied to a project (the source is locked in `.p4n4.json`), `template upgrade` checks for a newer version and applies the delta using the same merge rules as `pull`.

```bash
# Check if the applied template has a newer version
p4n4 template upgrade --check

# Apply the upgrade (interactive merge for modified files)
p4n4 template upgrade

# Upgrade to a specific version
p4n4 template upgrade --to 1.3.0
```

```
$ p4n4 template upgrade --check

  Current template:  acme/factory-baseline @ 1.2.0
  Latest available:  acme/factory-baseline @ 1.3.0

  Changelog (1.2.0 → 1.3.0):
    + Added n8n workflow: OTA model update notification
    + Updated Grafana dashboard: EI classification overlay panel
    ~ Changed: docker-compose.edge.yml — new healthcheck config
    ~ Changed: mosquitto/config/acl.j2 — added inference topic namespace

  Run `p4n4 template upgrade` to apply.
```

---

### 10.9 Integration with `p4n4 init`

When a template is specified in `p4n4 init`, the wizard merges template-defined variables into the standard init flow, replacing the built-in defaults. The template source is recorded in `.p4n4.json` for future `template upgrade` support.

```bash
# Init from a community template
p4n4 init --template factory-baseline

# Init from an org-scoped template
p4n4 init --template acme/factory-baseline

# Init from a Git URL
p4n4 init --template https://github.com/acme/p4n4-template-factory.git

# Init from a template at a specific version
p4n4 init --template acme/factory-baseline@1.2.0

# Init from a local template directory (useful during template development)
p4n4 init --template ./my-local-template
```

When `--template` is provided, `p4n4 init` replaces the built-in stack selection wizard with the template's `[template.variables]` prompts. Stack selection is inferred from `[template.requires].stacks`.

---

### 10.10 `.p4n4.json` — Template Lock Record

After a template pull or template-backed init, the applied template reference is recorded in the project's `.p4n4.json` manifest under a `template` key:

```json
{
  "version": "0.2",
  "project_name": "plant-a-stack",
  "site_id": "plant-a",
  "stacks": ["iot", "ai", "edge"],
  "ollama_model": "phi3:mini",
  "gpu": false,
  "ei_models": [
    { "name": "vibration-fault", "eim": "edge-impulse/models/motor-fault-v3.eim", "port": 1337 }
  ],
  "template": {
    "name": "factory-baseline",
    "org": "acme",
    "repo": "https://github.com/acme/p4n4-template-factory.git",
    "version": "1.2.0",
    "applied_at": "2026-03-04T14:22:00Z"
  }
}
```

This lock record enables `p4n4 template upgrade` to know exactly what was applied and at which version.

---

### 10.11 Security Considerations

| Concern | Mitigation |
|---------|-----------|
| Malicious template content | Templates are plain Git repos; operators review before applying. HTTPS URLs enforced by default. SSH supported for internal repos. |
| Secret leakage on push | `p4n4 template push` always scrubs `.env` and Node-RED credential blocks; refuses to push if `.env` has no corresponding `.env.example` |
| Tampered template on re-pull | Version is pinned in `.p4n4.json`; CLI warns if the upstream repo has force-pushed to a previously used tag |
| Private template repos | Full Git URL with SSH (`git@github.com:acme/...`) supported; CLI delegates auth to the system Git credential store |
| Org index tampering | Index repos can be pinned to a specific commit in `~/.p4n4/config.toml` for high-security environments |

---

### 10.12 Usage Examples

**Pull a community template and bring the stack up:**

```bash
p4n4 template pull factory-baseline --output ./my-plant
cd my-plant
p4n4 validate
p4n4 up --all
```

**Bootstrap a new template from an existing project:**

```bash
cd my-plant
p4n4 template init-manifest          # creates p4n4-template.toml interactively
p4n4 template push git@github.com:acme/p4n4-template-factory.git --tag 1.0.0
p4n4 template org publish acme factory-baseline 1.0.0
```

**Clone a private internal template using SSH:**

```bash
p4n4 template pull git@internal-git.acme.corp:iot/p4n4-template-plant.git@v2.1.0 \
  --var site_id=plant-b \
  --non-interactive
```

**Keep a project in sync with a shared organisational baseline:**

```bash
# Check for updates every week as part of a cron job or CI step
p4n4 template upgrade --check
# → shows changelog diff, exits non-zero if update available
```

---

## 11. Roadmap

### Phase 1 — Foundation (v0.1)
- [ ] Core IoT stack functional: Mosquitto + Node-RED + InfluxDB + Grafana
- [ ] Base GenAI stack: Ollama + Letta + n8n deployed and integrated via Node-RED
- [ ] Reference flows and workflows documented and exported
- [ ] Hardware-tested on Raspberry Pi 5 and x86 server
- [ ] `p4n4 init`, `up`, `down`, `status`, `validate` CLI commands (IoT stack only)

### Phase 2 — Intelligence Layer (v0.2)
- [ ] Pre-built Letta agent personas: Site Monitor, Anomaly Analyst, Operator Assistant
- [ ] Node-RED AI palette: curated nodes for Ollama/Letta integration
- [ ] Grafana AI panel: chat widget backed by Letta Operator Assistant agent
- [ ] n8n workflow library: alert enrichment, scheduled digest, incident escalation
- [ ] `p4n4 init` full three-stack wizard including GenAI and Edge Impulse
- [ ] `p4n4 ei` subcommands: list, infer, update, info
- [ ] `p4n4 secret` generation and rotation
- [ ] `p4n4 template pull` — Git URL and local path support
- [ ] `p4n4 template push` — publish to Git remote with secret scrubbing
- [ ] `p4n4 template search` / `info` — community index integration
- [ ] First official community templates published to `p4n4-org/templates`
- [ ] Published to PyPI as `p4n4`

### Phase 3 — Scale & Extensibility (v0.3+)
- [ ] Multi-site federation: replicated Node-RED → central InfluxDB + Grafana with site tagging
- [ ] OpenTelemetry integration for stack observability
- [ ] Embedding pipeline: sensor event descriptions embedded via `nomic-embed-text` into Letta archival memory
- [ ] Optional Kafka/NATS layer for high-throughput deployments replacing Mosquitto
- [ ] Community marketplace: shareable Node-RED flows and n8n workflows
- [ ] `p4n4 upgrade` with dry-run and rolling restart support
- [ ] `p4n4 add` / `p4n4 remove` for granular service management
- [ ] `p4n4 template upgrade` — delta merge with changelog
- [ ] Org index support: `p4n4 template org add/list/publish`
- [ ] Shell completion for bash, zsh, fish

---

## 12. Appendix

### 12.1 Port Reference

| Service | Stack | Port(s) | Exposure |
|---------|-------|---------|----------|
| Mosquitto | IoT | 1883, 8883, 9001 | IoT devices, Node-RED |
| Node-RED | IoT / Bridge | 1880 | Operators, internal services |
| InfluxDB | IoT | 8086 | Node-RED, Grafana, n8n |
| Grafana | IoT | 3000 | Operators (browser) |
| Ollama | GenAI | 11434 | Node-RED, Letta, n8n (internal only) |
| Letta | GenAI | 8283 | Node-RED, n8n (internal only) |
| n8n | GenAI / Bridge | 5678 | Operators, Grafana webhooks |

### 12.2 TIGUITTO vs p4n4 Comparison

| Capability | TIGUITTO (IoTStack) | p4n4 |
|------------|---------------------|------|
| Data ingestion | Telegraf (metrics agent) | Node-RED (visual flow engine) |
| MQTT broker | Eclipse Mosquitto | Eclipse Mosquitto |
| Time-series DB | InfluxDB | InfluxDB |
| Visualisation | Grafana | Grafana |
| Conditional logic | Limited (Telegraf processors) | Full (Node-RED function nodes) |
| Device actuation | ✗ | ✓ (MQTT out, HTTP out) |
| Protocol bridging | MQTT only | MQTT, HTTP, WebSocket, Modbus, OPC-UA… |
| AI integration | ✗ | ✓ Ollama, Letta, n8n |
| Workflow automation | ✗ | ✓ n8n |
| Agent memory | ✗ | ✓ Letta |
| Local LLM | ✗ | ✓ Ollama (phi3, mistral, llama3) |
| Visual programming | ✗ | ✓ Node-RED editor |
| Edge deployment | ✓ (Pi compatible) | ✓ (Pi 5 + optional GPU) |

### 12.3 Key Environment Variables (`.env`)

```bash
# Mosquitto
MQTT_USER=p4n4mqtt
MQTT_PASSWORD=changeme

# InfluxDB
INFLUXDB_ADMIN_TOKEN=your-influx-token
INFLUXDB_ORG=p4n4
INFLUXDB_BUCKET=raw_telemetry

# Grafana
GF_SECURITY_ADMIN_PASSWORD=changeme

# n8n
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=changeme
N8N_ENCRYPTION_KEY=your-encryption-key

# Letta
LETTA_SERVER_PASSWORD=changeme

# Ollama (GPU)
OLLAMA_NUM_GPU=1
```
