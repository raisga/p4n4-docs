# Edge Stack

The Edge stack (`p4n4-edge`) runs on-device model inference inside Docker. A Python
runner loads an Edge Impulse `.eim` model or an ONNX `.onnx` model, reads feature vectors
from MQTT, and publishes each result back to MQTT and InfluxDB. Without a model it runs in
**mock mode**, so the whole pipeline can be tested before a model exists.

It attaches to `p4n4-net` and works alongside the IoT and AI stacks, but doesn't need
either of them to start.

## Services

| Service | Container | Image | Port | Role |
|---------|-----------|-------|------|------|
| ei-runner | `p4n4-ei-runner` | built from `runner/` (`python:3.11-slim`) | 8080 | Inference runner and HTTP API |

## Data flow

```
sensors/<device-id>/raw  ──►  ei-runner  ──►  inference/<device-id>/result  (MQTT)
   {"values": [...]}             │
                                 └──────────►  ai_events bucket, inference_result measurement
```

The runner subscribes to `MQTT_TOPIC_INPUT` (default `sensors/+/raw`). The device id comes
from the topic, and results go to `MQTT_TOPIC_RESULTS` (default
`inference/{device}/result`, where `{device}` is replaced by the device id). Node-RED in
the IoT stack stores those results too; see [§8.6 of the specs](../decisions/specs.md#86-mqtt-topic-conventions).

Input payload:

```json
{"values": [1.23, 4.56, 7.89, 0.12, 3.45, 6.78]}
```

`values` must match the model's input size. Result payload:

```json
{
  "device": "vibration-sensor-01",
  "timestamp": "2026-03-11T12:00:00.000000+00:00",
  "label": "anomaly",
  "confidence": 0.9234,
  "anomaly_score": 0.8712,
  "latency_ms": 18.4,
  "mode": "model"
}
```

`mode` is `model` (Edge Impulse), `onnx` or `mock`. ONNX models don't produce an anomaly
score, so `anomaly_score` is always `0.0` in `onnx` mode.

## Model backends

`MODEL_BACKEND` in `.env` picks the backend:

| Value | Behavior |
|-------|----------|
| `auto` (default) | The `.eim` if present, else the `.onnx`, else mock mode |
| `eim` | Edge Impulse only; mock mode if no `.eim` is found |
| `onnx` | ONNX Runtime only; mock mode if no `.onnx` is found |
| `mock` | Simulated results, no model loaded |

For ONNX, the runner feeds `values` to the model's first input as a `float32` tensor of
shape `(1, N)`. Score arrays are softmaxed when they aren't already probabilities and
labelled from `ONNX_LABELS` (falling back to `class_0`, `class_1`, …); probability maps
such as sklearn-onnx `ZipMap` output use the map keys.

## Model deployment

Model files are **never committed**. They're mounted read-only from
`edge-impulse/models/` (`.eim`) and `onnx/models/` (`.onnx`).

```bash
make deploy-model MODEL=~/Downloads/my-model-linux-aarch64-v5.eim
# then set EI_MODEL_FILE=my-model-linux-aarch64-v5.eim in .env

make deploy-model MODEL=~/Downloads/my-classifier.onnx
# then set ONNX_MODEL_FILE=my-classifier.onnx and ONNX_LABELS=... in .env

make up
```

Use `make up` (or `docker compose up -d`) after changing the model or `.env`. It recreates
the runner with the new environment; `make restart` and `docker compose restart` keep the
old values.

`.eim` files are executables for one architecture (x86_64, aarch64, armv7l). Download the
one that matches the device from Edge Impulse Studio under **Deployment → Linux**.

> `p4n4 ei deploy`, `ei run` and `ei status` are stubs in v0.1 and exit non-zero. Use the
> Makefile or Compose commands above. See the [CLI reference](../reference/cli-reference.md#p4n4-ei).

## HTTP API

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/health` | Status, `mode`, model file, inference count, last inference time, MQTT and InfluxDB connection state |
| `GET` | `/api/v1/info` | Active backend, configured backend, model file, model details (project and labels for `.eim`; input name and shape for `.onnx`), `ONNX_LABELS`, MQTT topics |
| `POST` | `/api/v1/infer` | Run one sample (`{"values": [...], "device": "bench"}`) and return the result |

`POST /api/v1/infer` doesn't publish to MQTT, write to InfluxDB or count toward `/health`,
so it's safe for testing a model. `device` is optional and defaults to `api`. Errors return
`{"error": "..."}` with `400` (bad body), `411` (no `Content-Length`) or `413` (body over
1 MiB). The full contract is in [F-0.2.4 of the specs](../decisions/specs.md).

```bash
curl -X POST http://localhost:8080/api/v1/infer \
  -H "Content-Type: application/json" \
  -d '{"values": [1.2, 4.5, 7.8]}'
```

## Environment variables

| Variable | Default | Description |
|----------|---------|-------------|
| `MODEL_BACKEND` | `auto` | `auto` \| `eim` \| `onnx` \| `mock` |
| `EI_MODEL_FILE` | `model.eim` | `.eim` filename in `edge-impulse/models/` |
| `EI_API_KEY` | — | Edge Impulse API key; only for cloud features, blank for offline use |
| `ONNX_MODEL_FILE` | `model.onnx` | `.onnx` filename in `onnx/models/` |
| `ONNX_LABELS` | — | Comma-separated class labels in output order |
| `MQTT_HOST` / `MQTT_PORT` | `p4n4-mqtt` / `1883` | Broker address |
| `MQTT_USER` / `MQTT_PASSWORD` | — | Must match the IoT stack |
| `MQTT_TOPIC_INPUT` | `sensors/+/raw` | Topic filter for feature vectors |
| `MQTT_TOPIC_RESULTS` | `inference/{device}/result` | Result topic template |
| `INFLUXDB_TOKEN` | `p4n4-stack-token` | Must match the IoT stack |
| `INFLUXDB_ORG` | `ming` | Must match the IoT stack |
| `INFLUXDB_BUCKET_AI_EVENTS` | `ai_events` | Bucket for inference results |
| `TZ` | `UTC` | Container timezone |

`p4n4 init --layer edge` (or `--layer all`) writes these, sharing the InfluxDB token, org
and timezone with the other layers.

## Local overrides

Copy `docker-compose.override.yml.example` to `docker-compose.override.yml` and uncomment
what you need. It has blocks for pinning a model file, NVIDIA GPU access, a standalone
network with your own broker and InfluxDB, and a debug port. To pass a USB camera or other
device into the container, add a `devices:` list to `ei-runner` in the same file.

## Standalone mode

The stack declares `p4n4-net` as external, so the network must exist. Without the IoT
stack, either create it:

```bash
docker network create p4n4-net
make up
```

or use the standalone-network block in the override file to point the runner at another
broker and InfluxDB (`INFLUXDB_URL` is fixed to `http://p4n4-influxdb:8086` in
`docker-compose.yml`, so set it there). The HTTP API starts before the MQTT connection,
which retries every 5 s, so `/health` and `POST /api/v1/infer` work without a broker.
