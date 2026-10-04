# IoT Stack

The IoT stack (`p4n4-iot`) is the foundation of every p4n4 deployment. It owns the
`p4n4-net` Docker bridge network that all other stacks attach to.

## Services

| Service | Container | Image | Port | Role |
|---------|-----------|-------|------|------|
| Mosquitto | `p4n4-mqtt` | `eclipse-mosquitto:2` | 1883 (TCP) / 9001 (WebSocket) | MQTT broker |
| InfluxDB | `p4n4-influxdb` | `influxdb:2` | 8086 | Time-series database |
| Node-RED | `p4n4-node-red` | `nodered/node-red:latest` | 1880 | Flow-based data routing |
| Grafana | `p4n4-grafana` | `grafana/grafana-oss:latest` | 3000 | Dashboards |

Node-RED and Grafana wait for their dependencies' healthchecks before starting.

## Network

The IoT stack **creates** `p4n4-net`:

```yaml
networks:
  p4n4-net:
    name: p4n4-net
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

All other stacks declare it as `external: true`.

## Topics and storage

Devices publish JSON to `sensors/<device-id>/<measurement>`, for example
`sensors/greenhouse-01/temperature` with `{"value": 23.5, "unit": "C"}`.

| Topic | Bucket | Measurement | Tags |
|-------|--------|-------------|------|
| `sensors/<device-id>/<measurement>` | `INFLUXDB_BUCKET` (`raw_telemetry`) | `sensor_data` | `device`, `sensor` |
| `inference/<device-id>/result` | `INFLUXDB_BUCKET` | `inference` | `device`, `model` (if in payload) |
| `sandbox/sensors/…`, `sandbox/inference/…` | `INFLUXDB_SANDBOX_BUCKET` (`sandbox`) | same as above | same as above |

Numbers, strings and booleans in the payload become fields. The device comes from the
topic, so a `device` key in the payload is ignored. Messages on `sensors/` or `inference/`
topics with any other shape (for example `sensors/temperature`) are dropped with a warning
in the Node-RED debug sidebar, and failed InfluxDB writes reach a Catch node. The full
rules are in [§8.6 of the specs](../decisions/specs.md#86-mqtt-topic-conventions).

## InfluxDB buckets

| Bucket | Retention | Purpose |
|--------|-----------|---------|
| `raw_telemetry` | 30 days | Inbound sensor readings and inference results (primary bucket) |
| `processed_metrics` | 365 days | Downsampled / aggregated data |
| `ai_events` | Infinite | Edge runner results, AI annotations, anomaly flags |
| `system_health` | 7 days | Stack component health metrics |
| `sandbox` | 30 days | Development and testing |

`raw_telemetry` is created by InfluxDB's setup; `scripts/init-buckets.sh` creates the rest.
Grafana gets a datasource per bucket.

## Node-RED

The editor and Admin API require a login, checked against `NODE_RED_USER` and
`NODE_RED_PASSWORD`. If `NODE_RED_PASSWORD` is empty, every login is refused.
`p4n4 init` generates it; projects created before that must add both keys to the IoT
`.env`.

Flows live in `config/node-red/flows/flows.json`. Compose mounts the `flows/` directory
rather than the file, because Node-RED saves by renaming a temporary file over
`flows.json`, so deploying from the editor writes straight to the project. Commit the file
to keep your changes.

## Environment variables

| Variable | Default | Description |
|----------|---------|-------------|
| `TZ` | `UTC` | Container timezone |
| `INFLUXDB_USERNAME` / `INFLUXDB_PASSWORD` | `admin` / `adminpassword` | InfluxDB admin login |
| `INFLUXDB_ORG` | `ming` | InfluxDB organisation (shared with other stacks) |
| `INFLUXDB_TOKEN` | `p4n4-stack-token` | InfluxDB API token (shared with other stacks) |
| `INFLUXDB_BUCKET` | `raw_telemetry` | Primary bucket |
| `INFLUXDB_BUCKET_PROCESSED` | `processed_metrics` | Downsampled data |
| `INFLUXDB_BUCKET_AI_EVENTS` | `ai_events` | AI events |
| `INFLUXDB_BUCKET_HEALTH` | `system_health` | Stack health |
| `INFLUXDB_SANDBOX_BUCKET` / `INFLUXDB_SANDBOX_RETENTION` | `sandbox` / `30d` | Sandbox bucket |
| `GRAFANA_USER` / `GRAFANA_PASSWORD` | `admin` / `adminpassword` | Grafana admin login |
| `NODE_RED_USER` / `NODE_RED_PASSWORD` | `admin` / *(empty)* | Node-RED editor login |

The defaults are placeholders. `p4n4 init` generates real values and
`p4n4 secret rotate` replaces them.

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

`config/mosquitto/mosquitto.conf` ships with `allow_anonymous true` for development. For
production, turn it off and use a password file and ACL (templates are in
`config/mosquitto/passwd.example` and `acl.example`):

```conf
allow_anonymous false
password_file /mosquitto/config/passwd
acl_file /mosquitto/config/acl
```

```bash
mosquitto_passwd -c config/mosquitto/passwd <username>
```

Clients such as the edge runner then need matching `MQTT_USER` / `MQTT_PASSWORD` in their
`.env`. See the [Security guide](../guides/security.md#mosquitto-authentication).

## Make commands

```bash
make up / down / restart / status / logs
make start SERVICE=grafana    # start one service and its dependencies
make buckets                  # list InfluxDB buckets
make test-mqtt                # publish test readings
make test-sandbox             # publish test readings to the sandbox bucket
make bridge-status            # external broker bridge: connected or not
make clean                    # stop and remove data volumes
```
