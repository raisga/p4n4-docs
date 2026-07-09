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
p4n4 init my-project --layer all     # everything
```

The interactive wizard prompts for configuration (InfluxDB organisation, timezone,
service passwords — leave blank to auto-generate). It then:

- Fetches stack files from the canonical stack repos (`p4n4-iot`, `p4n4-ai`).
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
as separate Compose projects (the AI stack attaches to the `p4n4-net` network that
the IoT stack creates):

```
my-project/
├── .p4n4.json                  ← manifest at the root, lists all layers
├── iot/
│   ├── docker-compose.yml
│   ├── .env
│   ├── config/
│   └── scripts/
└── ai/
    ├── docker-compose.yml
    ├── .env
    ├── config/
    └── scripts/
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
| Node-RED | http://localhost:1880 |
| Grafana | http://localhost:3000 |
| InfluxDB | http://localhost:8086 |
| n8n | http://localhost:5678 |
| Letta | http://localhost:8283 |
| Ollama | http://localhost:11434 |

## Stop everything

```bash
p4n4 down
```

## Next steps

- [IoT stack reference](stacks/iot-stack.md)
- [AI stack reference](stacks/ai-stack.md)
- [CLI reference](reference/cli-reference.md)
