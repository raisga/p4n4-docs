# AI Stack

The AI stack (`p4n4-ai`) provides local LLM inference (Ollama), agent memory (Letta),
and workflow automation (n8n). It attaches to `p4n4-net` as an external network.

## Services

| Service | Image | Port | Role | Starts by default |
|---------|-------|------|------|-------------------|
| Ollama | `ollama/ollama:latest` | 11434 | Local LLM runtime | Yes |
| Letta | `letta/letta:latest` | 8283 | AI agent framework with memory | No |
| n8n | `n8nio/n8n:latest` | 5678 | Workflow automation | No |

Each service sits in a Compose profile of its own name. `COMPOSE_PROFILES` in `.env` lists
the ones that start, and defaults to `ollama`. Set `COMPOSE_PROFILES=ollama,letta,n8n` to run
all three; `p4n4 init` generates Letta's and n8n's secrets either way.

## Prerequisites

The IoT stack must be running (to provide `p4n4-net`) or the network must exist:

```bash
docker network create p4n4-net  # if not using p4n4-iot
```

Or start everything with: `p4n4 up` (starts all enabled stacks in dependency order, so `p4n4-iot` creates the network first)

## Pull models

```bash
./scripts/pull-models.sh llama3.2
# or
docker exec p4n4-ollama ollama pull llama3.2
```

## n8n workflows

Starter workflows are in `config/n8n/workflows/`:

| Workflow | Description |
|----------|-------------|
| `alert-enrichment.json` | Subscribes to `inference/+/result`; sends low-confidence results to Ollama for analysis |
| `scheduled-digest.json` | Hourly telemetry summary from InfluxDB via Ollama |
| `device-onboarding.json` | Listens on `devices/+/register`; registers new devices and confirms over MQTT |
| `incident-escalation.json` | Listens on `alerts/+/critical`; classifies severity via Ollama and publishes to `alerts/escalated` |

Import them via the n8n UI or mount the directory as a volume.

## GPU support

Uncomment the `deploy.resources` block in `docker-compose.override.yml` for NVIDIA GPU.

## Environment variables

| Variable | Description |
|----------|-------------|
| `OLLAMA_PORT` | Host port for the Ollama API (default `11434`). Change it when Ollama already runs on the host; containers still use `p4n4-ollama:11434`, and a p4n4-api on the host needs `P4N4_API_OLLAMA_URL` to match |
| `N8N_BASIC_AUTH_USER` / `_PASSWORD` | n8n UI credentials |
| `N8N_ENCRYPTION_KEY` | n8n data encryption key |
| `LETTA_SERVER_PASSWORD` | Letta API password |
| `INFLUXDB_TOKEN` / `INFLUXDB_ORG` / `INFLUXDB_BUCKET` | Shared with IoT stack (must match) |
