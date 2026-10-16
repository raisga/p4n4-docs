# Template Registry

The community template registry lives at [raisga/p4n4-templates](https://github.com/raisga/p4n4-templates).

## Using templates

```bash
# Search available templates
p4n4 template search

# Filter by keyword
p4n4 template search manufacturing

# Install a template
p4n4 template install factory-baseline

# List installed templates
p4n4 template list
```

## Built-in templates

| Name | Stacks | Description |
|------|--------|-------------|
| `mqtt-influx-grafana` | iot | MQTT → Telegraf → InfluxDB + file archive → Grafana. See the [greenhouse use case](https://github.com/raisga/p4n4-templates/blob/main/docs/use-cases/greenhouse-telemetry.md) |
| `mqtt-influx-grafana-ollama` | iot + ai | The same pipeline plus a local LLM (Gemma 4 E2B on Ollama) that queries InfluxDB through tool calls. See the [greenhouse assistant use case](https://github.com/raisga/p4n4-templates/blob/main/docs/use-cases/greenhouse-assistant.md) |
| `mqtt-influx-grafana-ollama-go2rtc` | iot + ai + extras | Road traffic counting with ALPR: ingest service (plates kept for a retention period), traffic dashboard without plates, an assistant with fixed numbers and plate lookups, and go2rtc video. See the [road traffic use case](https://github.com/raisga/p4n4-templates/blob/main/docs/use-cases/road-traffic.md) |
| `mqtt-nodered-influx-grafana` | iot | Greenhouse climate control: the same pipeline plus Node-RED rules (fan and valve with hysteresis, manual override, valve watchdog) that publish actuator commands over MQTT, and a dashboard with the reason behind each decision. A simulated greenhouse obeys the commands |
| `mqtt-influx-grafana-n8n` | iot + ai | Cold-chain compliance: n8n workflows alert on temperature excursions after a grace period, escalate when nobody acknowledges, record the corrective action through a signed link, and email and save a daily record. Mailpit catches the emails in the demo |
| `mqtt-influx-grafana-ollama-letta` | iot + ai | Maintenance assistant with memory: condition monitoring for pumps and fans plus a Letta agent on a local Ollama model, with an equipment register in core memory, past incidents in archival memory, InfluxDB tools, and maintenance logged to memory and Grafana. Demo history included |
| `factory-baseline` *(planned)* | iot + ai | Full stack for discrete manufacturing: vibration anomaly detection on the ai layer's `ei-runner`, alert enrichment, incident escalation |
| `iot-minimal` *(planned)* | iot | Minimal IoT-only starter |

`p4n4 template` isn't implemented yet. Copy a template directory to start a project
(`cp -r projects/mqtt-influx-grafana my-project`). Each one runs on its own.

## Project manifest

A template ships a `.p4n4.json` with two optional blocks besides `schema_version`,
`project` and `layers`:

```json
{
  "template": { "name": "mqtt-influx-grafana", "version": "0.2.0" },
  "dashboard": {
    "grafana_path": "/d/p4n4-telemetry/telemetry",
    "tabs": ["services", "edge", "grafana"],
    "theme": "theme",
    "cameras": [{ "id": "floor", "name": "Sales floor", "port": 1984, "path": "/api/stream.mjpeg?src=floor" }]
  }
}
```

| Key | Used by |
|---|---|
| `template` | `p4n4 validate`: for a project made from a template, it checks `docker-compose.yml` and that `.env` sets every variable in `.env.example`, instead of the base stack's files |
| `dashboard.grafana_path` | p4n4-dashboard, via `GET /api/v1/project`: the Grafana tab opens this page unless the admin set one. The registry validator checks that its uid is a provisioned dashboard |
| `dashboard.tabs` | p4n4-dashboard: hides the tabs not listed (`services`, `edge`, `agent`, `grafana`, `video`) while connected |
| `dashboard.cameras` | p4n4-dashboard: the Video tab's cameras until the deployment saves its own. Each is `{id, name}` plus an absolute http(s) `url`, or a `port` and `path` on the host the dashboard is connected to. The registry validator checks that a `port` is published by one of the template's services, and that a template listing `video` has cameras |
| `dashboard.theme` | p4n4-dashboard's brand tool: `dart run tool/brand.dart install <project>` installs this directory (`brand.json`, `icon.png`, `fonts/`) as a white-label brand. The registry validates it against `schema/theme.schema.json` |

`p4n4 validate` rejects unknown `dashboard` keys, unknown tab names, and a
`grafana_path` that doesn't start with `/`, and a `theme` outside the project or
without a `brand.json`, and malformed `cameras` (a bad or repeated id, no name,
neither or both of `url` and `port`).

## Contributing a template

Templates live in `projects/<template-name>/` of
[p4n4-templates](https://github.com/raisga/p4n4-templates). Each one is a complete,
runnable p4n4 project: `cp -r` it and `docker compose up -d` must work with no CLI.

1. Fork `raisga/p4n4-templates`, read its
   [authoring guide](https://github.com/raisga/p4n4-templates/blob/main/docs/authoring.md),
   and copy `mqtt-influx-grafana` (the reference template) as a starting point.
2. Describe it in `template.yaml` and `.p4n4.json`, and add a `README.md` with the use
   case, the hardware it needs and its data format.
3. Run the checks, then open a pull request:

```bash
uv run scripts/validate.py <template-name>   # static checks against schema/
projects/<template-name>/tests/smoke.sh      # end to end (needs Docker)
```

Never commit a `.env`, only `.env.example`; pin image versions to at least the minor
version; keep flows and dashboards self-contained.

## `template.yaml` quick reference

Validated against `schema/template.schema.json`:

```yaml
schema_version: 1
name: my-template
version: 0.1.0
title: Short title
description: >-
  One or two sentences.
layers: [iot]                # iot, ai, extras
tags: [mqtt, influxdb]
maintainers: [your-github-handle]

services:
  - name: mqtt
    description: Eclipse Mosquitto MQTT broker
    ports: [1883, 9001]
  - name: simulator
    description: Demo publisher
    profile: demo            # demo data sits behind a Compose profile

data:
  topics:
    - pattern: sensors/<device-id>/<measurement>
      payload: '{"value": 23.4, "unit": "C"}'

smoke_test: tests/smoke.sh
```
