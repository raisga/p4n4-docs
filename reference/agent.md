# Incident Investigator (`agent` profile)

A durable, read-only agent in the AI stack. It investigates each anomaly alert with the
local Ollama model, through tools that only read, and publishes its finding. Every step is
checkpointed with [Pi Durable](https://github.com/earendil-works/pi/tree/main/packages/durable),
so an investigation interrupted by a crash or a restart resumes where it was. It is the
`agent` service of [p4n4-ai](../stacks/ai-stack.md), off by default, and the
[`mqtt-influx-grafana-ollama-pidurable`](template-registry.md) template shows it end to end.

Phase 1, described here, changes nothing outside the agent. Phase 2 (actions) is a
[design note](../decisions/agent-phase2.md).

## How it works

```
inference/<device>/result ──► Node-RED flow (anomaly_score > threshold)
        ──► alerts/<device>/anomaly (QoS 1)
        ──► p4n4-agent: one investigation per device and alert window
              tools (read only) ──► InfluxDB, System 1, Letta
              model             ──► Ollama /v1 (AGENT_MODEL)
        ──► agent/investigations/<device> (QoS 1) + agent_investigation in ai_events
```

1. An alert arrives on `AGENT_ALERT_TOPIC`. Its **request ID** is
   `<device>:<window start>`, the start of the `AGENT_ALERT_WINDOW_S` window (aligned to
   the clock) that holds the alert's own `timestamp`. A redelivered or repeated alert for
   the same device and window has the same ID.
2. A new ID opens an investigation: one commit creates its own conversation, the
   incident's documents and its entry in the incident index. A known ID only counts the
   alert. The MQTT message is acknowledged after that commit, so an alert lost to a crash
   before it is stored is delivered again.
3. The alert goes to the conversation as its input, with the request ID; Pi Durable
   admits a request ID once per conversation.
4. The model calls tools until it calls `report_finding`, or until the tool budget is
   spent. Each tool call is its own durable task.
5. The finding is published to MQTT and InfluxDB, then the incident is marked
   `reported`.

Each incident has its own conversation, so a model never sees another incident, and a
conversation stays a few thousand tokens long.

## Tools

All are `replay: "safe"`: after a crash an interrupted call runs again. None changes the
outside world.

| Tool | Parameters | Reads |
|------|------------|-------|
| `query_influx` | `device`, `measurement` (`sensor_data` \| `inference`), `sensor`?, `field`?, `minutes` 1–1440, `every` (`10s` \| `1m` \| `5m` \| `15m`), `fn` (`mean` \| `max` \| `min` \| `last`) | An aggregated series of one device from `INFLUXDB_BUCKET`, at most 500 rows. The Flux query is a fixed template; device, sensor and field must be plain identifiers, so the model can't write Flux |
| `recent_inference` | `device`, `limit` 1–50 | The newest `inference` points (anomaly score, confidence, label) |
| `system1_cause` | none | System 1's `incident-triage` answer for the stored alert. Offered with the `system1` profile; an unreachable System 1 reads as "unavailable" |
| `letta_history` | `limit` 1–10 | The device's past faults from the archival memory of the Letta agent `AGENT_LETTA_AGENT` (GET requests only). Offered with the `letta` profile |
| `report_finding` | `cause`, `confidence` 0–1, `summary` | Records the finding in the incident's document and ends the run |

The tools see the alert through the incident's document, never through model text.

## Topics and data

| Topic | Direction | Payload |
|-------|-----------|---------|
| `alerts/<device>/anomaly` (`AGENT_ALERT_TOPIC`) | in, QoS 1, persistent session | `{"device", "anomaly_score", "label", "threshold", "timestamp"}`. The topic names the device; `timestamp` is ISO 8601 or epoch s/ms/µs/ns |
| `agent/investigations/<device>` (`AGENT_RESULT_TOPIC`) | out, QoS 1, not retained | `{"incident", "device", "status": "answered" \| "failed", "cause", "confidence", "summary", "error", "alert_at", "started_at", "finished_at"}` |

InfluxDB: one point per finding in `INFLUXDB_BUCKET_AI_EVENTS` (`ai_events`), measurement
`agent_investigation`, tags `device` and `status`, fields `incident`, `cause`,
`confidence`, `summary` and `duration_ms`, at the time the investigation started.

Nothing in the base stacks publishes `alerts/<device>/anomaly` yet. A Node-RED function
after an `inference/+/result` input does it; the template's flow
(`iot/config/node-red/flows/flows.json`, *Anomaly threshold*) is one:

```js
const r = msg.payload, device = msg.topic.split("/")[1];
if (typeof r.anomaly_score !== "number" || r.anomaly_score <= 0.7) return null;
return { topic: `alerts/${device}/anomaly`, qos: 1,
         payload: { device, anomaly_score: r.anomaly_score, label: r.label, threshold: 0.7, timestamp: r.timestamp } };
```

## Status API

Published on `${AGENT_BIND}:${AGENT_PORT}` (`127.0.0.1:8095`), for p4n4-api to proxy later.

| Request | Auth | Answer |
|---------|------|--------|
| `GET /health` | none | `{"status": "ok" \| "degraded", "mqtt": true, "model": "ollama/<model>"}` |
| `GET /v1/investigations?state=open\|recent\|all` | `Authorization: Bearer <AGENT_API_TOKEN>` | Incidents, newest first: `incident`, `device`, `status` (`open`, `concluded`, `reported`), `outcome`, `cause`, `confidence`, `summary`, `error`, `alerts`, `submissionId`, times |
| `GET /v1/investigations/<incident>` | the same | One incident, with `toolCalls` and the tools running now |

## Environment variables

In the AI stack's `.env`:

| Variable | Default | |
|----------|---------|---|
| `AGENT_IMAGE` | `ghcr.io/raisga/p4n4-agent:0.1.0` | `p4n4-agent:dev` after `make agent-build` |
| `AGENT_API_TOKEN` | empty | 32+ characters; empty or shorter: the agent refuses to start. `p4n4 init` generates it, `p4n4 secret rotate` rotates it |
| `AGENT_BIND` / `AGENT_PORT` | `127.0.0.1` / `8095` | Status API |
| `AGENT_MODEL` | `llama3.2` | An Ollama model that calls tools, pulled already. Ollama keeps one model in memory here (`OLLAMA_MAX_LOADED_MODELS=1`), so use the one the stack runs |
| `AGENT_CONTEXT_TOKENS` | `4096` | The context to plan for; keep it within Ollama's (`OLLAMA_CONTEXT_LENGTH`) |
| `AGENT_SQLITE_SYNC` | `FULL` | `NORMAL`, `FULL` or `EXTRA`: see [Durability](#durability) |
| `AGENT_ALERT_TOPIC` / `AGENT_RESULT_TOPIC` | `alerts/+/anomaly` / `agent/investigations` | The device is the alert topic's second level |
| `AGENT_ALERT_WINDOW_S` | `900` | Alert window, in seconds |
| `AGENT_MAX_TOOL_CALLS` | `8` | Tool calls per investigation; after them only `report_finding` runs |
| `AGENT_INVESTIGATION_TIMEOUT_S` | `900` | Counted from the alert's admission, across restarts; then the investigation ends `failed` |
| `AGENT_HISTORY_KEEP` | `200` | Reported investigations kept in the index |
| `AGENT_LETTA_AGENT` | `p4n4-maintenance` | The Letta agent `letta_history` reads |

The agent also reads `COMPOSE_PROFILES` (to offer the System 1 and Letta tools),
`SYSTEM1_API_KEY`, `LETTA_SERVER_PASSWORD`, `MQTT_*` and `INFLUXDB_*` from the same `.env`.
`AGENT_MODEL_TIMEOUT_S` (`300`), `AGENT_SYSTEM1_URL` and `AGENT_LETTA_URL` (a URL, or `off`)
can be set in `docker-compose.override.yml`.

## Durability

Pi Durable commits each conversation entry, each tool call's intent and result, and the
incident's documents to one SQLite file, `/data/agent.sqlite` on the `agent-data` volume,
before anything acts on them.

- **Process crash, OOM kill, `docker kill`, `docker restart`.** Nothing committed is lost.
  On start the agent reopens the store, resumes every open investigation under its
  original submission, reruns the read-only tool that was interrupted, and finishes. The
  template's `tests/crash.sh` checks this under Raspberry Pi 5 limits.
- **Power loss.** It depends on `AGENT_SQLITE_SYNC`. With `FULL` (the default), every
  commit is on the storage before it is acknowledged, so a power cut loses nothing
  committed; the cost is an fsync per commit, on an SD card a few milliseconds each and
  some wear (an investigation makes tens of commits, not thousands). `NORMAL` writes less
  and survives any process crash, but a power cut may lose the last commits, and with them
  the record that an alert was received: the broker still has it only if it was not
  acknowledged yet. `EXTRA` also syncs the directory. Cheap SD cards may acknowledge
  writes they haven't made, which no setting fixes; use an SSD or a high-endurance card.
- **One owner.** Pi Durable has no cross-process locking. The agent holds an exclusive
  SQLite lock on the store, so a second process that opens it fails instead of
  corrupting it. Run one agent per store: no `scale`, no second container on the same
  volume.
- **At-least-once results.** Publishing and recording "reported" are separate steps. A
  crash between them publishes the finding again: the MQTT message carries `incident` to
  deduplicate on, and the InfluxDB point has the same time and tags, so it is overwritten.
- **Crash loops.** Pi Durable 1.3.0 has no limit on how often a task cut short by a crash
  runs again (later releases add one). The agent's tools are HTTP requests, which can't
  crash the process; a store error ends the agent, and Docker restarts it.

## Context and compaction

Pi Durable can compact a long conversation, summarizing older entries with the model so
they fit its context. The agent turns automatic compaction off. Local models have small
contexts (Ollama's `/v1` endpoint uses the model's `num_ctx`, and cuts longer requests
silently), and a summary written by the same small model is lossy: it may drop the very
numbers an investigation rests on. Instead, each incident gets its own short
conversation, the tool budget bounds its length, and everything the agent relies on
(alert, finding, status) lives in typed documents, not in the transcript. If a model
still runs out of context, raise `OLLAMA_CONTEXT_LENGTH` and `AGENT_CONTEXT_TOKENS`
together or lower `AGENT_MAX_TOOL_CALLS`.

## Versions

The image pins `@earendil-works/pi-durable`, `pi-ai` and `chord` at 1.3.0 (Pi Durable is
experimental). The next release migrates SQLite stores to a new schema on open, and a
migrated store can't be opened by 1.3.0: back up `agent-data` before upgrading.

## Phase 2

Writing to Letta, n8n alerts and actuator commands are out of scope for phase 1. How
they would work: [Incident investigator: phase 2](../decisions/agent-phase2.md).
