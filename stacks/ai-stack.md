# AI Stack

The AI stack (`p4n4-ai`) provides local LLM inference (Ollama), agent memory (Letta),
and workflow automation (n8n). It attaches to `p4n4-net` as an external network.

## Services

| Service | Image | Port | Role | Starts by default |
|---------|-------|------|------|-------------------|
| Ollama | `ollama/ollama:0.35` | 11434 on `127.0.0.1` | Local LLM runtime | Yes |
| Letta | `letta/letta:0.16.8` | 8283 on `127.0.0.1` | AI agent framework with memory | No |
| n8n | `n8nio/n8n:2.41.6` | 5678 | Workflow automation | No |

The images are pinned. Letta stays on 0.16: `letta/letta:latest` is now Letta's Code App
Server, a different product, while p4n4-api's Agent endpoints use the 0.16 server
(`/v1/agents/`). n8n is pinned to the version the starter workflows were tested with.

Ollama has no authentication, so Ollama and Letta are published on `127.0.0.1` only
(`OLLAMA_BIND`, `LETTA_BIND`). n8n and Letta reach Ollama as `p4n4-ollama:11434` on
`p4n4-net`, and a p4n4-api on the host reaches it on localhost. For remote access, put a
reverse proxy with TLS in front instead of changing the bind address. Letta runs with
`SECURE=true`, so every request needs `Authorization: Bearer <LETTA_SERVER_PASSWORD>`, and it
keeps agents and their memory in the Postgres bundled in its image (volume `letta-pgdata`).

Each service sits in a Compose profile of its own name. `COMPOSE_PROFILES` in `.env` lists
the ones that start, and defaults to `ollama`. Set `COMPOSE_PROFILES=ollama,letta,n8n` to run
all three; `p4n4 init` generates Letta's and n8n's secrets either way.

## Prerequisites

The IoT stack must be running (to provide `p4n4-net`) or the network must exist:

```bash
# if not using p4n4-iot; the label lets the IoT stack use this network later
docker network create --driver bridge --subnet 172.20.0.0/16 \
  --label com.docker.compose.network=p4n4-net p4n4-net
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

They use two credentials, matched by name. Create them in n8n before importing:

- **`p4n4 MQTT`** (MQTT): host `p4n4-mqtt`, port `1883`, plus the broker login from the IoT
  `.env` if it requires one.
- **`p4n4 InfluxDB`** (Header Auth, used by the Scheduled Digest): name `Authorization`,
  value `Token <INFLUXDB_TOKEN>`.

The Scheduled Digest's **Settings** node holds the InfluxDB org and bucket (`ming`,
`raw_telemetry`); change them there if the project uses others. The workflows don't read
`.env` through `$env`, because n8n blocks that by default.

Import them in the n8n UI (**Workflows → Import from File**, then publish each one), or
from the command line:

```bash
docker cp config/n8n/workflows p4n4-n8n:/tmp/workflows
docker exec p4n4-n8n n8n import:workflow --separate --input=/tmp/workflows
for id in p4n4AlertEnrichment p4n4DeviceOnboarding p4n4IncidentEscalation p4n4ScheduledDigest; do
  docker exec p4n4-n8n n8n publish:workflow --id=$id
done
docker restart p4n4-n8n
```

The workflows have fixed IDs, so importing them again updates them instead of adding
copies.

n8n asks whoever opens it first to create the owner account, so open
<http://localhost:5678> and create it as soon as the stack is up. n8n runs without usage
reports or update checks, and its schedules use `TZ` from `.env`.

## GPU support

Uncomment the `deploy.resources` block in `docker-compose.override.yml` for NVIDIA GPU.

## Environment variables

| Variable | Description |
|----------|-------------|
| `OLLAMA_PORT` | Host port for the Ollama API (default `11434`). Change it when Ollama already runs on the host; containers still use `p4n4-ollama:11434`, and a p4n4-api on the host needs `P4N4_API_OLLAMA_URL` to match |
| `OLLAMA_BIND` | Address the Ollama API is published on (default `127.0.0.1`) |
| `OLLAMA_MAX_LOADED_MODELS` / `OLLAMA_NUM_PARALLEL` | Models kept in memory and requests served at once (default `1` / `1`; raise them on hosts with memory or a GPU to spare) |
| `LETTA_BIND` | Address the Letta API is published on (default `127.0.0.1`) |
| `N8N_BASIC_AUTH_USER` / `_PASSWORD` | Still passed to n8n, but n8n uses the owner account created on the first visit |
| `N8N_HOST` | Hostname in webhook URLs (default `localhost`) |
| `TZ` | Timezone for n8n's schedules and date expressions (default `UTC`) |
| `N8N_ENCRYPTION_KEY` | n8n data encryption key |
| `LETTA_SERVER_PASSWORD` | Letta API password |
| `INFLUXDB_TOKEN` / `INFLUXDB_ORG` / `INFLUXDB_BUCKET` | Shared with IoT stack (must match) |
