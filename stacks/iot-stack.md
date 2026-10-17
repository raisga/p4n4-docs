# IoT Stack

The IoT stack (`p4n4-iot`) is the foundation of every p4n4 deployment. It owns the
`p4n4-net` Docker bridge network that all other stacks attach to.

## Services

| Service | Container | Image | Port | Role | Profile (default) |
|---------|-----------|-------|------|------|-------------------|
| Mosquitto | `p4n4-mqtt` | `eclipse-mosquitto:2.0` | 1883 (TCP) / 9001 (WebSocket) / 8883 (TLS, optional) | MQTT broker | `mqtt` (on) |
| InfluxDB | `p4n4-influxdb` | `influxdb:2.9` | 8086 | Time-series database | `influxdb` (on) |
| Node-RED | `p4n4-node-red` | `nodered/node-red:5.0` | 1880 | Flow-based data routing | `node-red` (on) |
| Grafana | `p4n4-grafana` | `grafana/grafana:13.0` | 3000 | Dashboards | `grafana` (on) |
| Telegraf | `p4n4-telegraf` | `telegraf:1.40` | — | Host and broker metrics into `system_health`, for the **Stack Health** dashboard | `telegraf` (off) |
| Loki | `p4n4-loki` | `grafana/loki:3.7` | — | Stores container logs, for the **Stack Logs** dashboard and Explore | `loki` (off) |
| Alloy | `p4n4-alloy` | `grafana/alloy:v1.20.1` | — | Collects the logs of every `p4n4-*` container into Loki (needs `loki`) | `alloy` (off) |
| MQTTX Web | `p4n4-mqtt-ui` | `emqx/mqttx-web:v1.13.1` | 8081 | MQTT web client over the broker's WebSocket port | `mqtt-ui` (off) |

Images are pinned to a minor version: `docker compose pull` brings in patch releases, but
new features don't arrive unannounced. Grafana uses `grafana/grafana`, the OSS image
(`grafana/grafana-oss` stopped at 13.0.2).

Node-RED and Grafana wait for their dependencies' healthchecks before starting.

## Choosing services

Every service sits in a [Compose profile](https://docs.docker.com/compose/how-tos/profiles/)
of its own name, and `COMPOSE_PROFILES` in `.env` lists the ones that start. The default is
the MING stack (`mqtt,influxdb,node-red,grafana`); add `telegraf` for host metrics (CPU,
memory, disk, network) and Mosquitto broker stats, `loki,alloy` for container logs, and
`mqtt-ui` for a web MQTT client. Run `docker compose up -d --remove-orphans` after changing
it. Node-RED needs `mqtt` and `influxdb`, Grafana needs `influxdb`, and Alloy needs `loki`.
Telegraf has no hard dependencies: it retries the broker and buffers writes until InfluxDB is
up. Without a `COMPOSE_PROFILES` line, plain `docker compose up` starts nothing; the `make`
targets fall back to the MING stack.

## Network

The IoT stack **creates** `p4n4-net`:

```yaml
networks:
  p4n4-net:
    name: p4n4-net
    driver: bridge
    ipam:
      config:
        - subnet: ${P4N4_SUBNET:-172.20.0.0/16}
```

All other stacks declare it as `external: true`. If `172.20.0.0/16` overlaps a VPN or another
network, set `P4N4_SUBNET` in the IoT `.env`, stop every p4n4 stack, run
`docker network rm p4n4-net` and start again. p4n4 (CLI) 0.4.0 creates the network itself
on the default range whatever `P4N4_SUBNET` says; 0.4.1 follows it
([Troubleshooting](../guides/troubleshooting.md)).

### Ports

Each published service has a host port and a bind address in `.env`: `MQTT_PORT`,
`MQTT_WS_PORT`, `MQTT_TLS_PORT` and `MQTT_BIND`; `INFLUXDB_PORT` / `INFLUXDB_BIND`;
`NODE_RED_PORT` / `NODE_RED_BIND`; `GRAFANA_PORT` / `GRAFANA_BIND`; `MQTT_UI_PORT` /
`MQTT_UI_BIND`. The binds default to `0.0.0.0` (every interface); `127.0.0.1` keeps a
service on this machine, for example InfluxDB behind a proxy. Telegraf, Loki and Alloy
publish nothing.

## Topics and storage

Devices publish JSON to `sensors/<device-id>/<measurement>`, for example
`sensors/greenhouse-01/temperature` with `{"value": 23.5, "unit": "C"}`.

| Topic | Bucket | Measurement | Tags |
|-------|--------|-------------|------|
| `sensors/<device-id>/<measurement>` | `INFLUXDB_BUCKET` (`raw_telemetry`) | `sensor_data` | `device`, `sensor` |
| `inference/<device-id>/result` | `INFLUXDB_BUCKET` | `inference` | `device`, `model` (if in payload) |
| `sandbox/sensors/…`, `sandbox/inference/…` | `INFLUXDB_SANDBOX_BUCKET` (`sandbox`) | same as above | same as above |
| `system1/decisions/<set>` | `INFLUXDB_BUCKET_AI_EVENTS` (`ai_events`) | `system1_decision`, one point per answer | `set`, `question`, `choice`, `caller`, `checkpoint` |
| `system1/feedback/<set>` | `INFLUXDB_BUCKET_AI_EVENTS` | `system1_feedback`, one point per confirmation | `set`, `question`, `expected`, `choice`, `caller` |

Numbers, strings and booleans in the payload become fields. The device comes from the
topic, so a `device` key in the payload is ignored. Messages on `sensors/` or `inference/`
topics with any other shape (for example `sensors/temperature`) are dropped with a warning
in the Node-RED debug sidebar, and failed InfluxDB writes reach a Catch node. The full
rules are in [§8.6 of the specs](../decisions/specs.md#86-mqtt-topic-conventions).

System 1 decisions ([AI stack](ai-stack.md#system-1-typed-decisions)) go through their own
flow tab, **System 1 Decision Log**. Its tags must match `^[a-z0-9_-]{1,64}$`, so series stay
bounded by the question sets' menus; a decision with any other tag is dropped with a
warning. The fields are `confidence`, `abstain`, `used`, `set_version`, `latency_ms`, `id`,
`revision`, `subject` and `state_sha256`: the state itself is never stored. Feedback
points have `agreed`, `abstain`, `set_version` and `id`.

## InfluxDB buckets

| Bucket | Retention | Purpose |
|--------|-----------|---------|
| `raw_telemetry` | 30 days | Inbound sensor readings and inference results (primary bucket) |
| `processed_metrics` | 365 days | Hourly rollups of `raw_telemetry` |
| `ai_events` | Infinite | Edge runner results, AI annotations, anomaly flags |
| `system_health` | 7 days | Stack component health metrics |
| `sandbox` | 30 days | Development and testing |

`raw_telemetry` is created by InfluxDB's setup; `scripts/init-buckets.sh` creates the rest.
Grafana gets a datasource per bucket. Names and retentions come from `.env`
(`INFLUXDB_BUCKET*`, `INFLUXDB_*_RETENTION`) and apply when InfluxDB first creates the
buckets; change an existing bucket's retention with `influx bucket update`.

**Hourly rollups.** The `p4n4-rollup-hourly` InfluxDB task runs five minutes past every
hour and writes the mean, min, max and count of each numeric field in `raw_telemetry` over
the hour before to `processed_metrics` (`<field>_mean`, `_min`, `_max`, `_count`, same
measurement and tags, timestamped at the start of the hour). Readings that arrive after
their hour was rolled up aren't included. Older installs get the task and any missing
bucket with `make influx-init`.

## Logs

Every service logs to stdout, and Docker rotates each container's log at `LOG_MAX_SIZE`
(`10m`), keeping `LOG_MAX_FILE` (`3`) files. Mosquitto no longer writes a log file of its
own.

With the `loki` and `alloy` profiles, Alloy reads the logs of every container whose name
starts with `p4n4-` (this stack, the ai stack, extras and the dashboard), labels them with
`container`, `service` and `project`, and ships them to Loki, which keeps them for
`LOKI_RETENTION` (`168h`). Grafana's **Stack Logs** dashboard shows them with a container
filter and a search box; **Explore** with the Loki data source runs any LogQL query. Alloy
reads the Docker socket read-only, but anything that can read it sees every container's
configuration, environment variables included.

## Node-RED

The editor and Admin API require a login, checked against `NODE_RED_USER` and
`NODE_RED_PASSWORD`. If `NODE_RED_PASSWORD` is empty, every login is refused.
`p4n4 init` generates it; projects created before that must add both keys to the IoT
`.env`.

Flows live in `config/node-red/flows/flows.json`. Compose mounts the `flows/` directory
rather than the file, because Node-RED saves by renaming a temporary file over
`flows.json`, so deploying from the editor writes straight to the project. Commit the file
to keep your changes.

`NODE_RED_CREDENTIAL_SECRET` encrypts the credentials saved in flows
(`flows/flows_cred.json`). Left empty, Node-RED generates a key inside its data volume, and
the credentials are lost with the volume; set it once, since changing it later makes them
unreadable.

`NODE_RED_PACKAGES` lists npm packages (comma-separated) installed when the container
starts, where the editor's **Manage palette** installs them; they're reinstalled only when
the list changes. For [Node-RED Dashboard 2](https://dashboard.flowfuse.com/) operator UIs
at `http://<host>:1880/dashboard`: `NODE_RED_PACKAGES=@flowfuse/node-red-dashboard@1`.

## System 1

When the AI stack runs System 1 ([AI stack](ai-stack.md#system-1-typed-decisions)), this
stack logs its decisions and feedback (above) and adds three things.

- **The `p4n4-system1` subflow**, in the palette under **p4n4 AI**, calls System 1 from
  any flow:
  - **In:** the state in `msg.payload`, and the set in `msg.set` or the subflow's `SET`
    property (default `incident-triage`).
  - **Out:** the response in `msg.payload`, with each answer's `choice`, `confidence` and
    `abstain`, and `msg.system1 = {id, set, version, used}`.
  - **On any failure** (no key, an unknown set, an error status, a refused connection, a
    timeout after `TIMEOUT_MS`, default 5000) it sets `msg.error` and sends a null
    payload; it never throws. Treat an abstained answer like an error: no suggestion.
  - It publishes the decision log itself, with `CALLER` (default `node-red`) as the
    caller.
  - It needs `SYSTEM1_API_KEY` in this stack's `.env`, the same value as the AI stack's;
    `p4n4 init` and `p4n4 ai system1 enable` write it.
  - The **System 1 Decision Log** tab has a one-click example.
- **Grafana's System 1 dashboard** (folder MING) shows, per set and question:
  - answers per option, and the abstention and used rates;
  - the confidence distribution and latency;
  - agreement with the people who confirmed or corrected answers;
  - a drift table, with each option's share in the last 7 days against the 28 days
    before.
- **Drift alerts** (`provisioning/alerting/system1.yml`, folder "System 1"): every 15
  minutes, per set and question, the last 7 days against the 28 days before. They fire when
  the abstention rate doubles, an option's share moves more than 20 points, or latency p95
  passes 2.5 s, given at least 30 decisions in both periods. They go to the contact point
  **p4n4 System 1 drift**, n8n's System 1 Drift webhook, authenticated with
  `SYSTEM1_ALERT_TOKEN`, which publishes them on `system1/drift/<set>`. They don't touch
  your notification policies.
- **Labels.** `p4n4 ai system1 label <set> --write` writes labels of past events as
  `system1_label` points, shown as the dashboard's **Labels** annotations.
- **Kept examples.** When n8n's Operator Intake sets `FEEDBACK_STATES`, feedback carries
  the message text. Node-RED appends it to `/data/system1/feedback/<set>.jsonl` in its
  volume, never to InfluxDB, for `p4n4 ai system1 feedback`.

## Environment variables

| Variable | Default | Description |
|----------|---------|-------------|
| `TZ` | `UTC` | Container timezone |
| `COMPOSE_PROFILES` | `mqtt,influxdb,node-red,grafana` | Services that start (see above) |
| `LOG_MAX_SIZE` / `LOG_MAX_FILE` | `10m` / `3` | Docker's log rotation per container |
| `P4N4_SUBNET` | `172.20.0.0/16` | Subnet of `p4n4-net` (see [Network](#network)) |
| `*_PORT` / `*_BIND` | see [Ports](#ports) | Host port and address of each published service |
| `MQTT_USER` / `MQTT_PASSWORD` | *(empty)* | Device login; with a password set, the device listeners require a login (see [Mosquitto authentication](#mosquitto-authentication)) |
| `MQTT_PORT` / `MQTT_WS_PORT` | `1883` / `9001` | Host ports of the device listeners (TCP and WebSocket) |
| `MQTT_TLS_CERT_FILE` / `MQTT_TLS_KEY_FILE` | *(empty)* | Server certificate and key in `config/mosquitto/certs/`; both set: a TLS listener on `MQTT_TLS_PORT` (default `8883`) |
| `INFLUXDB_USERNAME` / `INFLUXDB_PASSWORD` | `admin` / `adminpassword` | InfluxDB admin login |
| `INFLUXDB_ORG` | `ming` | InfluxDB organisation (shared with other stacks) |
| `INFLUXDB_TOKEN` | `p4n4-stack-token` | InfluxDB API token (shared with other stacks) |
| `INFLUXDB_BUCKET` / `INFLUXDB_RAW_RETENTION` | `raw_telemetry` / `30d` | Primary bucket and its retention (first run only) |
| `INFLUXDB_BUCKET_PROCESSED` / `INFLUXDB_PROCESSED_RETENTION` | `processed_metrics` / `365d` | Hourly rollups |
| `INFLUXDB_BUCKET_AI_EVENTS` / `INFLUXDB_AI_EVENTS_RETENTION` | `ai_events` / `0` (forever) | AI events |
| `INFLUXDB_BUCKET_HEALTH` / `INFLUXDB_HEALTH_RETENTION` | `system_health` / `7d` | Stack health (Telegraf) |
| `INFLUXDB_SANDBOX_BUCKET` / `INFLUXDB_SANDBOX_RETENTION` | `sandbox` / `30d` | Sandbox bucket |
| `GRAFANA_USER` / `GRAFANA_PASSWORD` | `admin` / `adminpassword` | Grafana admin login |
| `GRAFANA_ALLOW_EMBEDDING` | `false` | Lets p4n4-dashboard show Grafana in a frame; `p4n4 init` sets it with the dashboard layer |
| `GRAFANA_SUB_PATH` | `/` | `/grafana/` when p4n4-dashboard proxies Grafana on its own origin |
| `NODE_RED_USER` / `NODE_RED_PASSWORD` | `admin` / *(empty in Compose, `adminpassword` in `.env.example`)* | Node-RED editor login |
| `NODE_RED_CREDENTIAL_SECRET` | *(empty: generated in the volume)* | Key for credentials saved in flows (see [Node-RED](#node-red)) |
| `NODE_RED_PACKAGES` | *(empty)* | npm packages installed at start |
| `LOKI_RETENTION` | `168h` | How long Loki keeps container logs |
| `TELEGRAF_HOSTNAME` / `TELEGRAF_INTERVAL` | `p4n4` / `10s` | Telegraf's `host` tag and collection interval |
| `SYSTEM1_API_KEY` | *(empty)* | The `p4n4-system1` subflow's bearer token, the AI stack's value; empty: the subflow answers with `msg.error` |
| `SYSTEM1_ALERT_TOKEN` | *(empty)* | What Grafana's System 1 drift alerts send to n8n, the AI stack's value; empty: n8n refuses them |

The defaults are placeholders. `p4n4 init` generates real values. `p4n4 secret rotate`
replaces `NODE_RED_PASSWORD`, `SYSTEM1_API_KEY` and `SYSTEM1_ALERT_TOKEN` (with the same
new values as the AI stack's) only: InfluxDB and Grafana keep the password and token they
first start with, so change those in the service (see the
[Security guide](../guides/security.md#secret-rotation)).

## External MQTT broker

The local broker can pull topics from another broker, like a long-running
`mosquitto_sub -h <host> -u <user> -P <password> -t <topic>`, and republish them locally,
where Node-RED and the edge runner consume them unchanged. It is inbound only: nothing is
published back. Configure it with `p4n4 init` (the wizard, or `--mqtt-remote`), or set the
variables in the IoT `.env` and run `docker compose up -d mqtt`:

| Variable | Default | Description |
|----------|---------|-------------|
| `MQTT_REMOTE_HOST` | *(empty: disabled)* | External broker host |
| `MQTT_REMOTE_PORT` | `8883` with TLS, else `1883` | External broker port |
| `MQTT_REMOTE_USER` / `MQTT_REMOTE_PASSWORD` | *(empty)* | Login on the external broker |
| `MQTT_REMOTE_TOPICS` | `sensors/#` | Comma-separated topic filters to pull in |
| `MQTT_REMOTE_PREFIX` | *(empty)* | Local topic prefix, e.g. `remote/` |
| `MQTT_REMOTE_QOS` | `0` | Subscription QoS |
| `MQTT_REMOTE_CLIENT_ID` | `p4n4-bridge-<container id>` | Client ID on the external broker |
| `MQTT_REMOTE_TLS` | `false` | Connect over TLS |
| `MQTT_REMOTE_CA_FILE` | system CAs | CA certificate in `config/mosquitto/certs/` |
| `MQTT_REMOTE_CERT_FILE` / `MQTT_REMOTE_KEY_FILE` | *(empty)* | Client certificate and key for mutual TLS |

Quote a password containing `$` or `#` in single quotes, or Compose will interpolate or
truncate it (`p4n4 init` does this for you). `make bridge-status` reports whether the bridge
is connected; it reads the state Mosquitto publishes on
`$SYS/broker/connection/p4n4-remote/state`.

## Mosquitto authentication

The stack's own services (Node-RED, Telegraf, the AI stack's runner) connect to `mqtt:1883`
on `p4n4-net`, an internal listener that isn't published and never asks for a login.
Devices connect to separate listeners, published on `MQTT_PORT` (TCP, default 1883),
`MQTT_WS_PORT` (WebSocket, default 9001) and, with a certificate, `MQTT_TLS_PORT` (TLS,
default 8883).

`config/mosquitto/entrypoint.sh` writes the listeners, logins and bridge into
`/mosquitto/conf.d` from `.env` at every start; `mosquitto.conf` keeps only persistence and
logging. Set `MQTT_USER` and `MQTT_PASSWORD` and the device listeners require a login; leave
`MQTT_PASSWORD` empty and they accept anonymous clients. Add accounts in
`config/mosquitto/passwd` (`make mqtt-user NAME=<name>`) and topic rules in
`config/mosquitto/acl` (from `acl.example`). `docker logs p4n4-mqtt` shows what it set up.
See the [Security guide](../guides/security.md#mosquitto-authentication) for the details.

## Make commands

```bash
make up / down / restart / status / logs
make start SERVICE=grafana    # start one service and its dependencies
make buckets                  # list InfluxDB buckets
make test-mqtt                # publish test readings
make test-sandbox             # publish test readings to the sandbox bucket
make bridge-status            # external broker bridge: connected or not
make mqtt-user NAME=sensor-001          # add a device login (prompts for the password)
make mqtt-user-delete NAME=sensor-001   # remove it
make influx-init              # create missing buckets and the hourly rollup task
make backup                   # back up InfluxDB, Grafana, Node-RED and Mosquitto data
make restore BACKUP=backups/<timestamp>
make clean                    # stop and remove data volumes
```

## Backup and restore

`make backup` writes `backups/<timestamp>/`: InfluxDB's own backup (every bucket, with its
users and tokens) and the Grafana, Node-RED and Mosquitto data volumes as `.tar.gz` files,
taken while the services run (stopped ones are skipped). `.env`, `config/` and Loki's logs
aren't included: keep `.env` with the backups and `config/` in git.

`make restore BACKUP=…` asks first, then replaces the data of each service in the backup;
the services must exist (`make up` first on a new machine). Use the `.env` the backup was
taken with: InfluxDB's tokens come back from the backup, and Node-RED, Telegraf and Grafana
authenticate with `INFLUXDB_TOKEN`.
