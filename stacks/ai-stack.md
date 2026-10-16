# AI Stack

The AI stack (`p4n4-ai`) provides local LLM inference (Ollama), agent memory (Letta),
workflow automation (n8n), fast typed decisions (System 1) and edge inference (an Edge
Impulse or ONNX model on the sensor data, the `ei-runner` service, which was the
`p4n4-edge` stack until [ADR-005](../decisions/adr/ADR-005.md)). It attaches to `p4n4-net`
as an external network.

## Services

| Service | Image | Port | Role | Starts by default |
|---------|-------|------|------|-------------------|
| Ollama | `ollama/ollama:0.35.1` | 11434 on `127.0.0.1` | Local LLM runtime | Yes |
| Letta | `letta/letta:0.16.8` | 8283 on `127.0.0.1` | AI agent framework with memory | No |
| n8n | `n8nio/n8n:2.41.6` | 5678 | Workflow automation | No |
| System 1 | `ghcr.io/raisga/p4n4-system1:0.1.0` | None (`p4n4-system1:8000` on `p4n4-net`) | Typed decisions with laya ([below](#system-1-typed-decisions)) | No |
| ei-runner | built from `runner/` (`python:3.11-slim`) | 8080 on `127.0.0.1` | Edge Impulse / ONNX inference on `sensors/+/raw` ([Edge Inference](edge-inference.md)) | No (yes with `p4n4 init --layer all` or `--ei-runner`) |

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
all three; `p4n4 init` generates Letta's, n8n's and System 1's secrets either way. None
depends on another: `COMPOSE_PROFILES=ei-runner` runs edge inference alone, without an LLM.

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
| `incident-escalation.json` | Listens on `alerts/+/critical`; site rules route known alerts, System 1 may suggest a team for the rest, and every alert is published to `alerts/escalated` with the on-call recipient; Ollama's summary follows on `alerts/escalated/summary` ([v2](#incident-escalation-v2)) |
| `operator-intake.json` | Turns operators' chat messages into requests they confirm: rules answer greetings, System 1 suggests the request type and the asset, buttons ask for the rest ([below](#operator-intake)) |
| `system1-drift.json` | Receives the IoT stack's Grafana drift alerts for System 1 and publishes each on `system1/drift/<set>` ([below](#system-1-typed-decisions)) |

They use these credentials, matched by name. Create the first two in n8n before importing:

- **`p4n4 MQTT`** (MQTT): host `p4n4-mqtt`, port `1883`, plus the broker login from the IoT
  `.env` if it requires one.
- **`p4n4 InfluxDB`** (Header Auth, used by the Scheduled Digest): name `Authorization`,
  value `Token <INFLUXDB_TOKEN>`.
- **`p4n4 System 1`** (Header Auth, used by the Incident Escalation and the Operator
  Intake): `p4n4 ai system1 enable` creates it, and `p4n4 secret rotate` updates it.
  Without it the workflows still escalate every alert and take every request, with no
  suggestions.
- **`p4n4 Operator Intake`** (Header Auth, checked by the Operator Intake's webhook):
  `Authorization: Bearer <OPERATOR_INTAKE_TOKEN>`, created and rotated the same way.
- **`p4n4 System 1 Alerts`** (Header Auth, checked by the System 1 Drift webhook):
  `Authorization: Bearer <SYSTEM1_ALERT_TOKEN>`, created and rotated the same way.

The Scheduled Digest's **Settings** node holds the InfluxDB org and bucket (`ming`,
`raw_telemetry`); change them there if the project uses others. The workflows don't read
`.env` through `$env`, because n8n blocks that by default.

Import them in the n8n UI (**Workflows → Import from File**, then publish each one), or
from the command line:

```bash
docker cp config/n8n/workflows p4n4-n8n:/tmp/workflows
docker exec p4n4-n8n n8n import:workflow --separate --input=/tmp/workflows
for id in p4n4AlertEnrichment p4n4DeviceOnboarding p4n4IncidentEscalation p4n4OperatorIntake p4n4ScheduledDigest p4n4System1Drift; do
  docker exec p4n4-n8n n8n publish:workflow --id=$id
done
docker restart p4n4-n8n
```

The workflows have fixed IDs, so importing them again updates them instead of adding
copies.

n8n asks whoever opens it first to create the owner account, so open
<http://localhost:5678> and create it as soon as the stack is up. n8n runs without usage
reports or update checks, and its schedules use `TZ` from `.env`.

## System 1 (typed decisions)

System 1 runs [laya](https://huggingface.co/convaiinnovations/laya), a small encoder that
answers *typed questions*: given a state and a question with a closed set of options, it
returns one of the options with probabilities and a confidence, in 0.1–0.3 s on two x86
cores. It can't write text or answer outside the options. p4n4 uses it to *route* where
the rules have no answer, and it abstains when unsure. Severity, safety, actuation and
thresholds stay in rules, and every consumer works the same without it. The design, the
spike's measurements and the thirteen rules of use are in
[System 1](../decisions/system1.md); the decision is [ADR-004](../decisions/adr/ADR-004.md).

```bash
p4n4 ai system1 enable     # profile, keys, question sets, n8n credentials, model, service
p4n4 ai system1 status
echo '{"alert": "Smoke detected in electrical cabinet 3"}' | p4n4 ai system1 decide incident-triage --state -
p4n4 ai system1 eval       # each set's gates on its evaluation file; exit 5 on a failure
p4n4 ai system1 feedback operator-intake   # kept confirmations as evaluation lines
p4n4 ai system1 label incident-triage --input alerts.jsonl --write   # label past alerts
```

Without the CLI: set `SYSTEM1_API_KEY` (32+ characters) and add `system1` to
`COMPOSE_PROFILES`, then `make system1-pull` (or `docker compose run --rm system1-pull`)
and `docker compose up -d`.

**The service.** `p4n4-system1` publishes no port: callers on `p4n4-net` use
`http://p4n4-system1:8000` with `Authorization: Bearer <SYSTEM1_API_KEY>`. It runs as UID
10001 with a read-only root filesystem, no capabilities, and limits of 2 GiB and 2 CPUs. It
serves one checkpoint, `multilingual` by default, offline from the `system1-models` volume
that `system1-pull` fills. It refuses to start without a key, on a revision or files whose
sha256 the image doesn't pin, or with a question set that breaks the model's budgets.

| Method | Path | What it does |
|--------|------|--------------|
| `POST` | `/v1/sets/<name>` | Runs a question set on `{"state": …}`. Adds `set` (name and version), `state_sha256` and, per answer, `abstain` (confidence under the set's threshold) |
| `GET` | `/v1/sets` | The sets the service loaded, with versions, options, thresholds and temperatures |
| `POST` | `/v1/systemone` | Ad hoc questions, `{"state": …, "questions": …}`; no other fields |
| `GET` | `/health` | Liveness, without a token |

A state over the checkpoint's 764-token budget (about 1,500 characters of prose), an
option over 48 tokens, more than 8 questions or 4,000 characters are refused with a 422
instead of being silently truncated, and a body over 64 KiB with a 413. Requests beyond 16
at once get a 503. Callers treat every error, a timeout and `abstain` as "no suggestion".

**Question sets** live in `config/system1/questions/<name>.json`, each with an evaluation
file `<name>.eval.jsonl` next to it. The stack ships two:

- **`incident-triage`**: the likely cause of a critical alert, from the alert's first line.
  Its threshold is 0.9. On the generic evaluation file it then suggests a cause for about
  a quarter of the alerts and is almost never wrong.
- **`operator-intake`**: the request type and the asset of an operator's message, in
  English and Spanish. Its thresholds are 0.95 and 0.8 (version 4), and its `other` and
  `pump` options mean "ask". On the generic file it fills the request type for about half
  the messages and the asset for most of them, with no confident misses.

Add your site's examples to the evaluation file, then run
`p4n4 ai system1 eval --fit-thresholds` to see what threshold your data supports. Edit the
set, bump its `version`, and run `eval` again.
- **Languages:** examples may carry `"lang": "es"`. With two or more languages, every gate
  must pass in each.
- **Temperatures:** a set's `temperatures` key (`{"<question>": T}`, from
  `eval --fit-calibration`) calibrates single questions, and a site file calibrates every
  set. `eval --fit-calibration --save-calibration` writes it to
  `config/system1/calibration/site.json` and sets `SYSTEM1_CALIBRATION`. Then restart
  `system1`, evaluate every set and bump its version.
- **Ranking sets:** a question named in the set's `eval.rank` is judged by the AUC of its
  positive option's probability (`min_auc`), not by coverage and accuracy.
- **Drafts:** `system1/drafts/` holds four sets S3 measured (log triage, device class, a
  System 2 router, an anomaly screen) that don't pass on the base checkpoints yet. The
  service never loads them ([System 1 §16](../decisions/system1.md#16-s3-implementation)).
- **Fine-tuning:** `make system1-train` fine-tunes `multilingual` on a generic synthetic
  corpus, off the device. A fine-tuned checkpoint is published to a Hugging Face repository
  p4n4 controls, pinned in the image like the base, and served with `SYSTEM1_REPO` and
  `SYSTEM1_REVISION`. Every set is then evaluated on it and its version bumped
  ([System 1 §17](../decisions/system1.md#17-s4-implementation)).

**Decisions** are published on `system1/decisions/<set>` and written by the IoT stack's
Node-RED to the `system1_decision` measurement in `ai_events` (tags `set`, `question`,
`choice`, `caller`, `checkpoint`). Confirmations and corrections are published on
`system1/feedback/<set>` and become `system1_feedback` points. States are never logged,
only their sha256. The IoT stack's Grafana has a **System 1** dashboard and its Node-RED
a **p4n4-system1** subflow ([IoT stack](iot-stack.md#system-1)).

**Drift.** The IoT stack's Grafana alerts when a set drifts:
- the abstention rate doubles;
- an option's share moves by more than 20 points;
- latency p95 goes over 2.5 s.

Each compares the last 7 days with the 28 days before. The **System 1 Drift** workflow
publishes every alert on `system1/drift/<set>`. A drift alert means: run `eval` with recent
feedback. It never changes a set.

**Hosts.** Plan 2 GiB of memory for it: fine on a NUC or a Raspberry Pi 5 next to a small
Ollama model, not recommended on a Raspberry Pi 4.

### Incident Escalation v2

1. **Normalise:** the alert's first line, at most 500 characters, becomes the state. The
   device, its type and the site stay in the workflow.
2. **Site Rules** (a Code node to edit for your site): known alert types (e-stop, door and
   firmware events), devices and device types map to teams, and the same node holds the
   cause → team table and the on-call recipient.
3. **System 1**, only when no rule matched, with a 5 s timeout. Its cause is used when it
   isn't abstained and the table maps it to a team. Every answer is logged on
   `system1/decisions/incident-triage`.
4. **Escalate:** always published on `alerts/escalated`, with `routed_by` (`rule`,
   `system1` or `on_call`), `team` or `likely_cause` / `likely_team` with `suggested: true`,
   and `on_call`.
5. **Explain:** Ollama's two-sentence summary follows on `alerts/escalated/summary`, after
   the escalation.

### Operator Intake

A channel adapter (a Telegram or WhatsApp bot, a web chat) connects operators to the
workflow.

- **Messages in:** it publishes `{conversation, user, text}` on
  `operator/messages/<channel>`, or POSTs the same with a `channel` to
  `http://<n8n>:5678/webhook/operator-intake` with `Authorization: Bearer
  <OPERATOR_INTAKE_TOKEN>`.
- **Replies out:** it reads them on `operator/replies/<channel>`, as `{conversation, text,
  buttons: [{label, data}]}`.
- **Button presses:** it sends a press back as `{conversation, button: <data>}`.

The workflow then:

1. **Answers greetings and thanks by rule**, in English or Spanish. They never reach
   System 1.
2. **Asks System 1** (`operator-intake`, 5 s timeout) for the request type and the asset.
   A suggestion is kept only when it clears its threshold.
3. **Asks for whatever is missing** with buttons; an unnumbered pump gets only the pump
   buttons.
4. **Asks the operator to confirm**, with buttons to change either field or cancel.
   Nothing happens before that.
5. **Publishes the request** on `operator/requests/<intent>`, and one confirmation per
   question on `system1/feedback/operator-intake`.

If System 1 is down, slow or refuses the key, the operator gets the buttons for
everything. Edit the **Intake Rules** node for your site: asset names and labels (keep
them in step with the `operator-intake` set's options), greetings, replies.

With `FEEDBACK_STATES = true` there, confirmations also carry the message text, which
Node-RED keeps in its volume, and `p4n4 ai system1 feedback operator-intake --append`
adds them to the evaluation file. Messages can hold personal data, so it is off by
default.

## GPU support

Uncomment the `deploy.resources` block in `docker-compose.override.yml` for NVIDIA GPU: the
Ollama one, and the `ei-runner` one for GPU-enabled `.eim` models. GPU settings stay in the
override file, so the stack runs unchanged on a Raspberry Pi 5 or an x86 host without one.

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
| `INFLUXDB_BUCKET_AI_EVENTS` | Bucket the inference runner writes to (default `ai_events`) |
| `MODEL_BACKEND`, `MAX_FEATURES`, `EI_RUNNER_BIND`, `EI_API_KEY`, `EI_MODEL_FILE`, `ONNX_MODEL_FILE`, `ONNX_LABELS`, `MQTT_*` | The inference runner's settings: see [Edge Inference](edge-inference.md#environment-variables) |
| `SYSTEM1_API_KEY` | Bearer token for System 1's callers, 32+ characters (generated by `p4n4 init`; empty: the service refuses to start) |
| `SYSTEM1_IMAGE` | System 1 image (default `ghcr.io/raisga/p4n4-system1:0.1.0`) |
| `SYSTEM1_REPO` | Hugging Face repository of the checkpoint (default `convaiinnovations/laya`); a fine-tuned one must be pinned in the image too |
| `SYSTEM1_REVISION` | Model commit in `SYSTEM1_REPO`, a 40-character sha; the image must pin its file hashes |
| `SYSTEM1_CHECKPOINT` | `multilingual` (default) or `english` |
| `SYSTEM1_THREADS` | torch threads for inference (default `2`) |
| `OPERATOR_INTAKE_TOKEN` | Bearer token channel adapters send to the Operator Intake webhook (generated by `p4n4 init`; stored in n8n's **p4n4 Operator Intake** credential) |
| `SYSTEM1_CALIBRATION` | A site temperature file in `config/system1/calibration/` (e.g. `site.json`); empty: none |
| `SYSTEM1_ALERT_TOKEN` | Bearer token Grafana's drift alerts send to the System 1 Drift webhook; the same value as the IoT stack's (stored in n8n's **p4n4 System 1 Alerts** credential) |
