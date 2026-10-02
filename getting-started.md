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
p4n4 init my-project --layer all     # iot + ai + edge
```

The interactive wizard prompts for configuration (InfluxDB organisation, timezone,
service passwords, including the Node-RED editor login — leave blank to auto-generate).
It then:

- Fetches stack files from the canonical stack repos (`p4n4-iot`, `p4n4-ai`, `p4n4-edge`),
  or from local checkouts passed with `--source-iot` / `--source-ai` / `--source-edge`.
- Generates cryptographically secure secrets and writes them to `.env`.
- Creates a `.p4n4.json` project manifest.

### Project layout

A **single-layer** project keeps the stack files at the project root:

```
my-project/
├── .p4n4.json
├── .env
├── docker-compose.yml
├── config/
└── scripts/
```

A **multi-layer** project gives each stack its own subdirectory, so the stacks run
as separate Compose projects (the AI and edge stacks attach to the `p4n4-net` network that
the IoT stack creates):

```
my-project/
├── .p4n4.json                  ← manifest at the root, lists all layers
├── iot/
│   ├── docker-compose.yml
│   ├── .env
│   ├── config/
│   └── scripts/
├── ai/
│   ├── docker-compose.yml
│   ├── .env
│   ├── config/
│   └── scripts/
└── edge/
    ├── docker-compose.yml
    ├── .env
    ├── runner/
    ├── edge-impulse/models/    ← .eim models (never committed)
    └── onnx/models/            ← .onnx models (never committed)
```

Shared values such as `INFLUXDB_TOKEN` are written identically to every layer's
`.env`, and `p4n4 secret rotate` keeps them in sync.

## Start the stacks

```bash
cd my-project
p4n4 up          # all enabled stacks
p4n4 up iot      # one stack only
```

With no argument, stacks start in dependency order:

1. `iot` — creates the `p4n4-net` Docker bridge network, starts Mosquitto and InfluxDB first.
2. `ai` — attaches to `p4n4-net`.
3. `edge` — attaches to `p4n4-net`.

`p4n4 down` stops them in reverse order.

## Service URLs (default ports)

| Service | URL |
|---------|-----|
| Node-RED | http://localhost:1880 (log in with `NODE_RED_USER` / `NODE_RED_PASSWORD` from the IoT `.env`) |
| Grafana | http://localhost:3000 |
| InfluxDB | http://localhost:8086 |
| n8n | http://localhost:5678 |
| Letta | http://localhost:8283 |
| Ollama | http://localhost:11434 |
| Inference runner | http://localhost:8080/health |

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
the edge stack running, publish a feature vector to `sensors/<device-id>/raw` to get a
result on `inference/<device-id>/result` (mock mode until you deploy a model).

## Next steps

- [IoT stack reference](stacks/iot-stack.md)
- [AI stack reference](stacks/ai-stack.md)
- [Edge stack reference](stacks/edge-stack.md)
- [CLI reference](reference/cli-reference.md)
- [REST API](reference/api.md) and [dashboard](reference/dashboard.md)
