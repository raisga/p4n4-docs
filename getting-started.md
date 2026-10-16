# Getting Started

## Prerequisites

- Docker >= 24 with Compose v2 (`docker compose`)
- Python >= 3.11
- Git

## Installation

```bash
pip install p4n4
p4n4 --version
```

## Create a project

```bash
p4n4 init my-project                 # IoT stack only (default)
p4n4 init my-project --layer iot,ai  # multiple stacks
p4n4 init my-project --layer all     # iot + ai (with the inference runner) + dashboard
p4n4 init my-project --layer all,extras  # plus the optional extras layer
```

The interactive wizard prompts for configuration (InfluxDB organisation, timezone,
service passwords, including the Node-RED editor login — leave blank to auto-generate).
It then:

- Fetches stack files from the canonical stack repos (`p4n4-iot`, `p4n4-ai`, `p4n4-extras`),
  or from local checkouts passed with `--source-iot` / `--source-ai` / `--source-extras` /
  `--source-dashboard`.
- Generates cryptographically secure secrets and writes them to `.env`.
- Writes a `.gitignore` that keeps every `.env`, generated credentials and model files out of git.
- Creates a `.p4n4.json` project manifest.

### Project layout

A **single-layer** project keeps the stack files at the project root:

```
my-project/
├── .p4n4.json
├── .gitignore                  ← keeps .env, credentials and models out of git
├── .env
├── docker-compose.yml
├── config/
└── scripts/
```

A **multi-layer** project gives each stack its own subdirectory, so the stacks run
as separate Compose projects (the AI stack and the extras attach to the `p4n4-net` network
that the IoT stack creates):

```
my-project/
├── .p4n4.json                  ← manifest at the root, lists all layers
├── .gitignore                  ← covers every layer's .env, credentials and models
├── iot/
│   ├── docker-compose.yml
│   ├── .env
│   ├── config/
│   └── scripts/
└── ai/
    ├── docker-compose.yml
    ├── .env
    ├── config/
    ├── scripts/
    ├── runner/                 ← the inference runner (profile ei-runner)
    ├── edge-impulse/models/    ← .eim models (never committed)
    └── onnx/models/            ← .onnx models (never committed)
```

The optional extras layer (`--layer extras`) adds `extras/`, with nothing enabled until you
add a service ([Extras](stacks/extras.md)). A project made before the edge stack merged into
the AI stack has an `edge/` layer: it keeps working, and `p4n4 migrate edge` moves it
([guide](guides/migrating-from-edge.md)).

Shared values such as `INFLUXDB_TOKEN` are written identically to every layer's
`.env`. InfluxDB, Grafana and n8n keep the secrets they first start with, so change those
in the service rather than in `.env` alone (see the [Security guide](guides/security.md#secret-rotation)).

## Start the stacks

```bash
cd my-project
p4n4 up          # all enabled stacks
p4n4 up iot      # one stack only
```

With no argument, stacks start in dependency order:

1. `iot` — creates the `p4n4-net` Docker bridge network, starts Mosquitto and InfluxDB first.
2. `ai` — attaches to `p4n4-net`.
3. `extras` — attaches to `p4n4-net`; skipped while no service is enabled.

`p4n4 up` creates `p4n4-net` itself when a project has no IoT layer or the IoT stack is down.
`p4n4 down` stops the stacks in reverse order.

Stacks use fixed container names (`p4n4-influxdb`, …) and host ports, so only one project
runs on a host at a time. If another project is still up, `p4n4 up` names it and prints the
command that stops it; `p4n4 down --all` stops every p4n4 project on the host.

## Service URLs (default ports)

| Service | URL |
|---------|-----|
| Node-RED | http://localhost:1880 (log in with `NODE_RED_USER` / `NODE_RED_PASSWORD` from the IoT `.env`) |
| Grafana | http://localhost:3000 |
| InfluxDB | http://localhost:8086 |
| n8n (optional) | http://localhost:5678 (add `n8n` to `COMPOSE_PROFILES` in the AI `.env`; create the owner account on the first visit) |
| Letta (optional) | http://localhost:8283 (add `letta` to `COMPOSE_PROFILES` in the AI `.env`; localhost only) |
| Ollama | http://localhost:11434 (`OLLAMA_PORT` in the AI `.env`; change it if Ollama already runs on the host; localhost only) |
| Inference runner | http://localhost:8080/health (localhost only) |
| Dashboard (web) | http://localhost:8088 (with the `dashboard` layer) |

MQTT is on `localhost:1883` (TCP) and `:9001` (WebSocket). Credentials are in each
layer's `.env`; `p4n4 secret show` prints them masked.

## Stop everything

```bash
p4n4 down
```

## Send a test reading

Devices publish JSON to `sensors/<device-id>/<measurement>`:

```bash
mosquitto_pub -h localhost -t sensors/test-01/temperature -m '{"value": 23.5, "unit": "C"}'
```

Node-RED writes it to InfluxDB, and it shows up on Grafana's *Sensor Data* panel. With
the inference runner enabled (`ei-runner` in the AI stack's `COMPOSE_PROFILES`), publish a
feature vector to `sensors/<device-id>/raw` to get a result on `inference/<device-id>/result`
(mock mode until you deploy a model).

## Next steps

- [IoT stack reference](stacks/iot-stack.md)
- [AI stack reference](stacks/ai-stack.md)
- [Edge inference reference](stacks/edge-inference.md) (the AI stack's runner)
- [Extras](stacks/extras.md) (optional services)
- [CLI reference](reference/cli-reference.md)
- [REST API](reference/api.md) and [dashboard](reference/dashboard.md)
