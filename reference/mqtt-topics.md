# MQTT Topics

Every topic a p4n4 service publishes or subscribes to, in one place. The conventions behind
them are in [§8.6 of the specs](../decisions/specs.md#86-mqtt-topic-conventions); this page
is the lookup table.

Every payload is a JSON object. A device id is one topic level, so it can't contain `/`,
`+` or `#`. Inside `p4n4-net`, services connect to `mqtt:1883` (the internal listener, no
login). Devices and host tools use the device listeners on `MQTT_PORT` (default 1883),
`MQTT_WS_PORT` (9001) or `MQTT_TLS_PORT` (8883), which require a login once `MQTT_PASSWORD`
is set ([Security guide](../guides/security.md#mosquitto-authentication)).

## Telemetry and inference

| Topic | Publisher → consumer | Payload | Stored |
|-------|----------------------|---------|--------|
| `sensors/<device-id>/<measurement>` | devices, [p4n4-api](api.md) `POST /api/v1/telemetry` → Node-RED | `{"value": 23.5, "unit": "C"}`; numbers, strings and booleans become fields | `INFLUXDB_BUCKET` (`raw_telemetry`), measurement `sensor_data`, tags `device`, `sensor` |
| `sensors/<device-id>/raw` | devices → [inference runner](../stacks/edge-inference.md) | `{"values": [1.23, 4.56, …]}`, at most `MAX_FEATURES` numbers | Not by the runner; the result is |
| `inference/<device-id>/result` | inference runner → Node-RED, p4n4-api | `{device, timestamp, label, confidence, anomaly_score, latency_ms, mode}` | The runner: `ai_events`, measurement `inference_result`. Node-RED: `raw_telemetry`, measurement `inference`, tags `device`, `model` |
| `sandbox/sensors/…`, `sandbox/inference/…` | test publishers → Node-RED | As above | `INFLUXDB_SANDBOX_BUCKET` (`sandbox`) |

- Node-RED drops `sensors/` and `inference/` messages whose topic doesn't match these
  shapes (e.g. `sensors/temperature`), with a warning in its debug sidebar. A `device` key in
  the payload is ignored: the topic names the device.
- Messages carrying `"_stored_by": "p4n4-api"` are skipped by Node-RED: p4n4-api has already
  written them (with the device's own `ts`) and publishes them only for live views.
- The runner's topics are `MQTT_TOPIC_INPUT` and `MQTT_TOPIC_RESULTS` in the AI `.env`;
  `{device}` in the results template is replaced by the device id.

## Nodes and devices

| Topic | Publisher → consumer | Payload | Notes |
|-------|----------------------|---------|-------|
| `status/<node>/online` | [hardware daemon](hardware.md#p4n4_hwdpy-hardware-daemon) | `{"online": true}` | Retained. The last will, and a clean stop, set `false` |
| `status/<node>/health` | hardware daemon | `{"state": "ok" \| "degraded" \| "critical", "down": [...], "services": {...}, "ts": ...}` | Retained, published when it changes |
| `sensors/<node>/button` | hardware daemon → Node-RED | `{"value": 1, "press": "single" \| "double" \| "long"}` | Stored like any reading (`sensor=button`) |
| `devices/<device-id>/register` | devices → n8n *Device Onboarding* | the device's details | |
| `devices/<device-id>/status` | n8n *Device Onboarding* → device | confirmation | |
| `commands/<device-id>/<action>` | flows → devices | per action | The specs' convention for actuation; the base stacks don't publish on it |
| `actuators/<zone>/<actuator>/set`, `…/mode` | Node-RED rules → actuators | `mqtt-nodered-influx-grafana` template only | Commands, and manual override |

`<node>` is the daemon's `P4N4_NODE_ID` (default: the hostname).

## Alerts

| Topic | Publisher → consumer | Notes |
|-------|----------------------|-------|
| `alerts/<device-id>/critical` | Node-RED flows → n8n *Incident Escalation* | Other `alerts/<device-id>/<alert-type>` follow the same convention |
| `alerts/escalated` | n8n → consumers | Every critical alert, with the team (or suggested team) and on-call |
| `alerts/escalated/summary` | n8n → consumers | Ollama's summary of an escalation, after it |

## System 1 and operators

| Topic | Publisher → consumer | Stored |
|-------|----------------------|--------|
| `system1/decisions/<set>` | System 1 callers (n8n, the Node-RED subflow) → Node-RED | `ai_events`, measurement `system1_decision`, one point per answer (QoS 1, not retained) |
| `system1/feedback/<set>` | n8n *Operator Intake*, callers → Node-RED | `ai_events`, measurement `system1_feedback` (QoS 1, not retained) |
| `system1/drift/<set>` | n8n *System 1 Drift* → consumers | A drift alert from Grafana, firing or resolved |
| `operator/messages/<channel>` | channel adapter → n8n *Operator Intake* | `{conversation, user, text}`, or `{conversation, button}` for a press |
| `operator/replies/<channel>` | n8n → channel adapter | `{conversation, text, buttons: [{label, data}]}` |
| `operator/requests/<intent>` | n8n → consumers | A request the operator confirmed |

See [AI stack: System 1](../stacks/ai-stack.md#system-1-typed-decisions) and
[Operator Intake](../stacks/ai-stack.md#operator-intake).

## Broker and tooling

| Topic | Publisher | Notes |
|-------|-----------|-------|
| `$SYS/broker/connection/p4n4-remote/state` | Mosquitto | `1` while the [external-broker bridge](../stacks/iot-stack.md#external-mqtt-broker) is connected, `0` otherwise; local only. `make bridge-status` reads it |
| `$SYS/#` | Mosquitto | Broker stats; Telegraf collects them into `system_health` |
| `emu/gpio/<pin>/set` | you → [p4n4-emu](emulator.md) | Drives an emulated input pin (`0` / `1`), e.g. the button on 27 |
| `emu/gpio/<pin>/state` | p4n4-emu | An emulated output pin's level, e.g. the LED on 17 |

## Planned

| Topic | Notes |
|-------|-------|
| `federation/<site-id>/sensors/#` | Spoke → hub telemetry for multi-site federation (v0.3, not implemented) |
