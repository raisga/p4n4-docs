# Configuration

Every p4n4 layer is configured by its own `.env`, next to its `docker-compose.yml`:
`iot/.env`, `ai/.env` and `extras/.env` in a multi-layer project, or `.env` at the root of a
single-layer one ([ADR-002](../decisions/adr/ADR-002.md)). `p4n4 init` writes them from
each stack's `.env.example`, with generated secrets. This page lists every key; the stack
pages explain the services behind them.

Apply a change with `p4n4 up` (Compose recreates the containers whose settings changed).
Keys marked **setup-only** are read once, when the service first creates its data: changing
them in `.env` afterwards locks clients out (see [Secret rotation](../guides/security.md#secret-rotation)).

## Shared across layers

These keys appear in more than one layer's `.env` and must hold the **same value** in each.
`p4n4 init`, `p4n4 secret rotate` and `p4n4 ai system1 enable` keep them in step; when you
change one by hand, change it everywhere.

| Key | Layers | Why |
|-----|--------|-----|
| `INFLUXDB_TOKEN` | iot, ai | The ai layer's n8n and inference runner write to the iot layer's InfluxDB |
| `INFLUXDB_ORG` | iot, ai | Same |
| `INFLUXDB_BUCKET`, `INFLUXDB_BUCKET_AI_EVENTS` | iot, ai | Where n8n reads readings and the runner writes results |
| `SYSTEM1_API_KEY` | iot, ai | Node-RED's `p4n4-system1` subflow calls the ai layer's System 1 with it |
| `SYSTEM1_ALERT_TOKEN` | iot, ai | Grafana's System 1 drift alerts (iot) send it to n8n (ai) |
| `TZ` | iot, ai, extras | Not required to match, but `p4n4 init` writes one value to all |

## Common keys

In every layer:

| Key | Default | Meaning |
|-----|---------|---------|
| `COMPOSE_PROFILES` | per layer (below) | Services that start: each service is a Compose profile of its own name |
| `RESTART_POLICY` | `unless-stopped` | Every container's restart policy: `unless-stopped`, `always`, `on-failure` or `no` |
| `TZ` | `UTC` | Container timezone; n8n's schedules and date expressions use it |

## iot layer

[IoT stack](../stacks/iot-stack.md). `COMPOSE_PROFILES` default:
`mqtt,influxdb,node-red,grafana`; also available: `telegraf`, `loki`, `alloy` (needs `loki`)
and `mqtt-ui`.

### General

| Key | Default | Meaning |
|-----|---------|---------|
| `LOG_MAX_SIZE` / `LOG_MAX_FILE` | `10m` / `3` | Docker's log rotation per container: rotated at the size, this many files kept |
| `P4N4_SUBNET` | `172.20.0.0/16` | Subnet of `p4n4-net`. Change it when the range overlaps a VPN or another network, then recreate the network. p4n4 (CLI) 0.4.0 ignores it when it creates the network; 0.4.1 reads it ([Troubleshooting](../guides/troubleshooting.md)) |

### Ports

| Key | Default | Meaning |
|-----|---------|---------|
| `MQTT_BIND` | `0.0.0.0` | Host address the three device listeners are published on |
| `INFLUXDB_PORT` / `INFLUXDB_BIND` | `8086` / `0.0.0.0` | InfluxDB's host port and address |
| `NODE_RED_PORT` / `NODE_RED_BIND` | `1880` / `0.0.0.0` | Node-RED's |
| `GRAFANA_PORT` / `GRAFANA_BIND` | `3000` / `0.0.0.0` | Grafana's |
| `MQTT_UI_PORT` / `MQTT_UI_BIND` | `8081` / `0.0.0.0` | MQTTX Web's (`mqtt-ui` profile) |

`127.0.0.1` keeps a service on this machine. The device listeners' ports are under
Mosquitto below.

### Mosquitto

| Key | Default | Meaning |
|-----|---------|---------|
| `MQTT_USER` | *(empty)* | Device account; always has access to every topic |
| `MQTT_PASSWORD` | *(empty)* | Set: the device listeners require a login (`MQTT_USER` and the accounts in `config/mosquitto/passwd`). Empty: they accept anonymous clients |
| `MQTT_PORT` | `1883` | Host port of the device listener (container port 1884) |
| `MQTT_WS_PORT` | `9001` | Host port of the device WebSocket listener |
| `MQTT_TLS_PORT` | `8883` | Host port of the TLS listener |
| `MQTT_TLS_CERT_FILE` / `MQTT_TLS_KEY_FILE` | *(empty)* | Server certificate (chain) and key, by name in `config/mosquitto/certs/` or as an absolute path; both set: the TLS listener starts |

The stack's own services use the internal listener, `mqtt:1883` on `p4n4-net`, which is
never published and never asks for a login. See
[Mosquitto authentication](../guides/security.md#mosquitto-authentication).

### Bridge to an external broker

Inbound only: the local broker subscribes on another broker and republishes locally.

| Key | Default | Meaning |
|-----|---------|---------|
| `MQTT_REMOTE_HOST` | *(empty: off)* | External broker host |
| `MQTT_REMOTE_PORT` | `8883` with TLS, else `1883` | Its port |
| `MQTT_REMOTE_USER` / `MQTT_REMOTE_PASSWORD` | *(empty)* | Login on it. The password is external: `p4n4 secret show` masks it, but never generates or rotates it |
| `MQTT_REMOTE_TOPICS` | `sensors/#` | Comma-separated topic filters to pull in |
| `MQTT_REMOTE_PREFIX` | *(empty)* | Prepended to each topic locally, e.g. `remote/` |
| `MQTT_REMOTE_QOS` | `0` | Subscription QoS, 0–2 |
| `MQTT_REMOTE_CLIENT_ID` | `p4n4-bridge-<container id>` | Client ID on the external broker |
| `MQTT_REMOTE_TLS` | `false` | Connect over TLS |
| `MQTT_REMOTE_CA_FILE` | system CA bundle | CA file in `config/mosquitto/certs/` |
| `MQTT_REMOTE_CERT_FILE` / `MQTT_REMOTE_KEY_FILE` | *(empty)* | Client certificate and key for mutual TLS |

### InfluxDB

| Key | Default | Meaning |
|-----|---------|---------|
| `INFLUXDB_USERNAME` / `INFLUXDB_PASSWORD` | `admin` / `adminpassword` | Admin login. Password: generated, **setup-only** |
| `INFLUXDB_ORG` | `ming` | Organisation (shared) |
| `INFLUXDB_TOKEN` | `p4n4-stack-token` | All-access API token (shared). Generated, **setup-only** |
| `INFLUXDB_BUCKET` / `INFLUXDB_RAW_RETENTION` | `raw_telemetry` / `30d` | Primary bucket and its retention (first start only) |
| `INFLUXDB_BUCKET_PROCESSED` / `INFLUXDB_PROCESSED_RETENTION` | `processed_metrics` / `365d` | Hourly rollups of the primary bucket |
| `INFLUXDB_BUCKET_AI_EVENTS` / `INFLUXDB_AI_EVENTS_RETENTION` | `ai_events` / `0` (forever) | Inference results, System 1 decisions, AI annotations |
| `INFLUXDB_BUCKET_HEALTH` / `INFLUXDB_HEALTH_RETENTION` | `system_health` / `7d` | Telegraf's host and broker metrics |
| `INFLUXDB_SANDBOX_BUCKET` / `INFLUXDB_SANDBOX_RETENTION` | `sandbox` / `30d` | Readings published under `sandbox/` |

### Grafana

| Key | Default | Meaning |
|-----|---------|---------|
| `GRAFANA_USER` / `GRAFANA_PASSWORD` | `admin` / `adminpassword` | Admin login. Password: generated, **setup-only** |
| `GRAFANA_ALLOW_EMBEDDING` | `false` | Lets p4n4-dashboard frame Grafana; `p4n4 init` sets it with the dashboard layer |
| `GRAFANA_SUB_PATH` | `/` | `/grafana/` when p4n4-dashboard proxies Grafana on its own origin |

### Node-RED

| Key | Default | Meaning |
|-----|---------|---------|
| `NODE_RED_USER` / `NODE_RED_PASSWORD` | `admin` / `adminpassword` | Editor and Admin API login; an empty password refuses every login. Generated, rotated by `p4n4 secret rotate` |
| `NODE_RED_CREDENTIAL_SECRET` | *(empty)* | Key for credentials saved in flows. Empty: generated inside the Node-RED volume and lost with it. **Setup-only**: changing it makes saved credentials unreadable |
| `NODE_RED_PACKAGES` | *(empty)* | npm packages installed at start, comma-separated, e.g. `@flowfuse/node-red-dashboard@1` |
| `SYSTEM1_API_KEY` | *(empty)* | The `p4n4-system1` subflow's bearer token (shared). Empty: the subflow answers with `msg.error` |
| `SYSTEM1_ALERT_TOKEN` | *(empty)* | What Grafana's System 1 drift alerts send to n8n (shared). Empty: n8n refuses them |

### Telegraf

| Key | Default | Meaning |
|-----|---------|---------|
| `TELEGRAF_HOSTNAME` | `p4n4` | The `host` tag on its metrics |
| `TELEGRAF_INTERVAL` | `10s` | Collection interval |

### Loki

| Key | Default | Meaning |
|-----|---------|---------|
| `LOKI_RETENTION` | `168h` | How long container logs are kept |

## ai layer

[AI stack](../stacks/ai-stack.md). `COMPOSE_PROFILES` default: `ollama`; also available:
`letta`, `n8n`, `system1`, `ei-runner`. `p4n4 init --layer all` or `--ei-runner` adds
`ei-runner`; `p4n4 ai system1 enable` adds `system1`.

### Ollama

| Key | Default | Meaning |
|-----|---------|---------|
| `OLLAMA_PORT` | `11434` | Host port; change it when Ollama already runs on the host. Containers always use `p4n4-ollama:11434` |
| `OLLAMA_BIND` | `127.0.0.1` | Address the API is published on. It has no authentication |
| `OLLAMA_MAX_LOADED_MODELS` | `1` | Models kept in memory at once |
| `OLLAMA_NUM_PARALLEL` | `1` | Requests served at once |

### Letta

| Key | Default | Meaning |
|-----|---------|---------|
| `LETTA_SERVER_PASSWORD` | `lettapassword` | Clients send it as `Authorization: Bearer <password>`. Generated, rotated |
| `LETTA_BIND` | `127.0.0.1` | Address the API is published on |

### n8n

| Key | Default | Meaning |
|-----|---------|---------|
| `N8N_ENCRYPTION_KEY` | `change-me-32-char-encryption-key` | Encrypts stored credentials; 32+ characters. Generated, **setup-only**: keep it once n8n has credentials |
| `N8N_HOST` | `localhost` | Hostname in webhook URLs; your server's name or IP in production |
| `N8N_BASIC_AUTH_USER` / `N8N_BASIC_AUTH_PASSWORD` | `admin` / `adminpassword` | Unused: n8n dropped basic auth in 1.0. Its login is the owner account created on the first visit |

### System 1

| Key | Default | Meaning |
|-----|---------|---------|
| `SYSTEM1_API_KEY` | *(empty)* | Callers' bearer token, 32+ characters; empty: the service refuses to start (shared). Generated, rotated |
| `SYSTEM1_IMAGE` | `ghcr.io/raisga/p4n4-system1:0.1.0` | The service image; it pins the checkpoints' hashes |
| `SYSTEM1_REPO` / `SYSTEM1_REVISION` | `convaiinnovations/laya` / a 40-character sha | Model repository on huggingface.co and its commit; the image refuses a revision it doesn't pin |
| `SYSTEM1_CHECKPOINT` | `multilingual` | `multilingual` or `english` |
| `SYSTEM1_THREADS` | `2` | torch threads (the container is limited to 2 CPUs) |
| `SYSTEM1_CALIBRATION` | *(empty)* | Site calibration file in `config/system1/calibration/`; changing it means evaluating every set again |
| `OPERATOR_INTAKE_TOKEN` | *(empty)* | Channel adapters' token for n8n's operator-intake webhook. Generated, rotated |
| `SYSTEM1_ALERT_TOKEN` | *(empty)* | Grafana's drift alerts' token for n8n (shared). Generated, rotated |

### Inference runner (`ei-runner`)

| Key | Default | Meaning |
|-----|---------|---------|
| `MODEL_BACKEND` | `auto` | `auto` (`.eim`, else `.onnx`, else mock), `eim`, `onnx` or `mock`. A selected model that doesn't load stops the runner |
| `EI_MODEL_FILE` | `model.eim` | File in `edge-impulse/models/` |
| `ONNX_MODEL_FILE` / `ONNX_LABELS` | `model.onnx` / *(empty)* | File in `onnx/models/`, and its class labels in output order |
| `MAX_FEATURES` | `65536` | Most values one sample may carry |
| `EI_RUNNER_BIND` | `127.0.0.1` | Address the HTTP API (port 8080) is published on. It has no authentication |
| `EI_API_KEY` | *(empty)* | Edge Impulse cloud features only |
| `MQTT_HOST` / `MQTT_PORT` | `p4n4-mqtt` / `1883` | Broker the runner uses: the iot layer's internal listener |
| `MQTT_USER` / `MQTT_PASSWORD` | *(empty)* | Only for a broker other than the stack's own; the internal listener needs no login |
| `MQTT_TOPIC_INPUT` / `MQTT_TOPIC_RESULTS` | `sensors/+/raw` / `inference/{device}/result` | See [MQTT topics](mqtt-topics.md) |

### InfluxDB (shared with iot)

`INFLUXDB_TOKEN`, `INFLUXDB_ORG`, `INFLUXDB_BUCKET` (n8n's starter workflows read readings
from it) and `INFLUXDB_BUCKET_AI_EVENTS` (the runner writes results to it): the same values
as the iot layer's.

## extras layer

[Extras](../stacks/extras.md). `COMPOSE_PROFILES` default: empty; each extra you add is a
profile of its own name, with its keys in this same `.env`. Only `RESTART_POLICY` and `TZ`
ship.

## p4n4-api and p4n4-dashboard

Both have their own `.env` and settings:

- **p4n4-api:** `P4N4_PROJECT_DIR`, `P4N4_API_*` (bind, port, auth, CORS, the runner's
  URL…). See the [REST API reference](api.md#configuration).
- **p4n4-dashboard (web):** `DASHBOARD_*`, the upstreams it proxies (`GRAFANA_UPSTREAM`,
  the API…) and basic auth. See [Dashboard service](../stacks/dashboard.md#environment-variables).

## Secrets at a glance

| | Keys | `p4n4 secret rotate` |
|-|------|----------------------|
| **Rotated** | `NODE_RED_PASSWORD`, `LETTA_SERVER_PASSWORD`, `SYSTEM1_API_KEY`, `OPERATOR_INTAKE_TOKEN`, `SYSTEM1_ALERT_TOKEN` | Replaces them in every layer; the next `p4n4 up` applies them |
| **Setup-only** | `INFLUXDB_PASSWORD`, `INFLUXDB_TOKEN`, `GRAFANA_PASSWORD`, `N8N_ENCRYPTION_KEY` | Leaves them alone: change them in the service, then in `.env` |
| **External** | `MQTT_REMOTE_PASSWORD` | Never generated or rotated: the remote broker issued it |

`MQTT_PASSWORD` isn't generated by `p4n4 init` yet; set it yourself to require device
logins. `p4n4 secret show` lists the generated and external secrets, masked.
