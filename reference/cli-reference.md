# CLI Reference

Install: `pip install p4n4`

## Global options

```
p4n4 [--version] [--help]
```

---

## `p4n4 init NAME`

Interactive project wizard. Scaffolds stack files, generates secrets, writes `.p4n4.json`.

```bash
p4n4 init my-project                       # IoT layer (default)
p4n4 init my-project --layer iot,ai        # multiple layers
p4n4 init my-project --layer all           # iot + ai (with the inference runner) + dashboard
p4n4 init my-project --layer all,extras    # plus the optional extras layer
p4n4 init my-project --no-interactive      # skip wizard, use defaults
p4n4 init my-project --source-iot ../p4n4-iot   # scaffold from a local checkout (offline)
```

The wizard prompts for the InfluxDB org, timezone and service passwords (blank means
generate one), including the Node-RED editor password, which is written with
`NODE_RED_USER=admin` to the IoT `.env`. The ai layer gets the inference runner and its
model directories too, sharing the InfluxDB token, org and timezone with the other layers;
the wizard asks whether to run it (`ei-runner` in `COMPOSE_PROFILES`). The extras layer
starts with nothing enabled. Unknown layer names are rejected, and so is `edge`, which
merged into ai ([ADR-005](../decisions/adr/ADR-005.md)). With the IoT layer, the wizard also offers to
pull topics from an [external MQTT broker](../stacks/iot-stack.md#external-mqtt-broker)
(host, login, topics, local prefix, TLS).

| Flag | Description |
|------|-------------|
| `--layer <names>` | Layers to enable: `iot`, `ai`, `extras`, `dashboard`, `all` (every layer but `extras`; `all,extras` adds it), or comma-separated |
| `--ei-runner` / `--no-ei-runner` | Run the ai layer's edge inference runner (adds `ei-runner` to `COMPOSE_PROFILES`). Default: on with `all`, else off |
| `--no-interactive` | Skip the wizard and auto-generate all secrets |
| `--source-iot PATH` / `--source-ai PATH` / `--source-extras PATH` | Use a local stack checkout instead of cloning |
| `--mqtt-remote HOST[:PORT]` | Bridge topics in from an external MQTT broker (IoT layer) |
| `--mqtt-remote-user` / `--mqtt-remote-password` | Its login; the password can come from `P4N4_MQTT_REMOTE_PASSWORD` instead |
| `--mqtt-remote-topics LIST` | Comma-separated topic filters to pull in (default `sensors/#`) |
| `--mqtt-remote-tls` / `--mqtt-remote-ca FILE` | Connect over TLS; optional CA certificate, copied into the project |

**Layout:** single-layer projects place `docker-compose.yml`, `config/`, `scripts/`, and
`.env` at the project root. Multi-layer projects give each layer its own subdirectory
(`<project>/iot/`, `<project>/ai/`, `<project>/extras/`) so the stacks run as separate Compose projects;
shared `.env` keys (e.g. `INFLUXDB_TOKEN`) are written identically to every layer.
Each layer's `.env` sets `COMPOSE_PROJECT_NAME`: `<project>` for single-layer projects,
`<project>-<layer>` for multi-layer ones (lowercased, other characters replaced by `-`). That
keeps each project's volumes separate. Don't change it on an existing project: Compose would
switch to new, empty volumes. A `.gitignore` at the project root keeps every layer's `.env`,
generated credentials (Mosquitto's `passwd` and certificates, Node-RED's credentials), model
files and p4n4-api's data out of git; an existing `.gitignore` is left as it is.
See [Getting Started](../getting-started.md#project-layout).

---

## `p4n4 add STACK`

Add a stack to an existing project.

```bash
p4n4 add ai
p4n4 add extras --path /srv/my-project
```

---

## `p4n4 remove STACK`

Remove a stack from an existing project.

```bash
p4n4 remove extras
p4n4 remove ai --yes       # skip confirmation
```

---

## `p4n4 up [STACK]`

Start one or all enabled stacks in dependency order (`iot` → `ai` → `extras` → `dashboard`).
A stack with no service enabled (an extras layer before you add one) is skipped instead of
failing.

```bash
p4n4 up                    # all stacks
p4n4 up iot                # IoT stack only
p4n4 up --no-detach        # foreground mode
p4n4 up --emu rpi5         # under p4n4-emu, sized like a Raspberry Pi 5
```

`--emu <profile>` runs the stacks under [p4n4-emu](emulator.md) with a hardware profile
(`rpi4`, `rpi5`, `nuc`, `mcu-class`) by running `p4n4-emu up`, so `p4n4-emu` must be on
`PATH`. `p4n4 down` stops stacks started that way through `p4n4-emu down`.

`up` creates the `p4n4-net` network when it's missing, so AI-only projects (edge
inference alone included), and `p4n4 up ai` while the IoT stack is down, start too. A `p4n4-net` made without
Compose's label is recreated with it while no container uses it. Before starting anything it checks
that no other project holds the fixed `p4n4-*` container names (see `down --all` below).

---

## `p4n4 down [STACK]`

Stop one or all stacks, in reverse dependency order (dependents stop before `iot`
removes the shared network).

```bash
p4n4 down
p4n4 down ai
p4n4 down --volumes        # also remove volumes
p4n4 down --all            # every p4n4 project on this host, from any directory
```

`--all` finds every project with a `p4n4-*` container, lists them, and asks before
stopping them. Projects that own a network other projects join (iot's `p4n4-net`) stop
last. With `--volumes` it deletes their data too.

Stacks use fixed container names and host ports, so only one project runs on a host at a
time. `p4n4 up` checks this before starting anything: if another project holds the names,
it lists them and prints the command that stops that project (or `p4n4 down --all`).

---

## `p4n4 status`

Show container status for all enabled stacks, including stopped and crashed services.
Multi-layer projects print one table per stack. It reports an error when Docker or Compose
fails, instead of "No services found".

---

## `p4n4 logs [SERVICE]`

Tail logs from services.

```bash
p4n4 logs                          # single-layer project: all services
p4n4 logs grafana --tail 50        # one service
p4n4 logs --stack ai               # multi-layer project: pick a stack
p4n4 logs --no-follow              # multi-layer: dump all stacks and exit
```

In multi-layer projects, pass `--stack <name>` to follow one stack's logs, or
`--no-follow` to print logs from every stack once.

---

## `p4n4 validate`

Validate `.p4n4.json`, required stack files, and `.env` keys (the IoT layer requires
`NODE_RED_USER` and `NODE_RED_PASSWORD`). Multi-layer projects are checked per layer
directory (`iot/…`, `ai/…`, `extras/…`). The inference runner's files (`runner/`,
`edge-impulse/models`, `onnx/models`) and keys (`MODEL_BACKEND`, `MQTT_HOST`, …) are required
only while `ei-runner` is in the ai layer's `COMPOSE_PROFILES`.

---

## `p4n4 upgrade [STACK]`

Pull latest Docker images.

```bash
p4n4 upgrade               # all stacks
p4n4 upgrade iot
```

---

## `p4n4 secret ACTION`

| Action | Description |
|--------|-------------|
| `show` | Show masked secrets from `.env` (per stack in multi-layer projects) |
| `rotate` | Re-generate passwords and tokens (see below) |
| `generate` | Print new secrets to stdout |

`rotate` replaces whichever of these keys are present, which their services read at every
start:

- **IoT:** `NODE_RED_PASSWORD`, `SYSTEM1_API_KEY`, `SYSTEM1_ALERT_TOKEN` (the same new
  values as the AI layer's)
- **AI:** `LETTA_SERVER_PASSWORD`, `SYSTEM1_API_KEY`, `OPERATOR_INTAKE_TOKEN`,
  `SYSTEM1_ALERT_TOKEN` (n8n's `p4n4 System 1`, `p4n4 Operator Intake` and `p4n4 System 1
  Alerts` credentials are updated with them, when n8n is enabled)

It doesn't rotate `INFLUXDB_PASSWORD`, `INFLUXDB_TOKEN`, `GRAFANA_PASSWORD` or
`N8N_ENCRYPTION_KEY` (nor the unused `N8N_BASIC_AUTH_PASSWORD`), and says so. Their services
keep the value from first setup, so a new value in `.env` alone would lock every client out
of InfluxDB and Grafana, and stop n8n from starting. The
[Security guide](../guides/security.md#secret-rotation) explains how to change them in the
service. `show` lists all of them, masked.

`MQTT_REMOTE_PASSWORD` (an external broker's login) appears in `show`, fully masked, but is
never rotated: the external broker issued it.

In multi-layer projects, `rotate` updates every layer's `.env`. `p4n4 init` writes keys
shared across stacks (e.g. `INFLUXDB_TOKEN` in both `iot/.env` and `ai/.env`) with the
**same** value; change those in every layer when you change them by hand.

---

## `p4n4 ai system1`

System 1: fast typed decisions with laya, off by default
([design](../decisions/system1.md), [AI stack](../stacks/ai-stack.md#system-1-typed-decisions)).
Every command works on the project's ai layer.

| Subcommand | Description |
|------------|-------------|
| `enable [--checkpoint multilingual\|english] [--no-pull] [--source-ai PATH]` | Adds `system1` to `COMPOSE_PROFILES`, generates `SYSTEM1_API_KEY`, `OPERATOR_INTAKE_TOKEN` and `SYSTEM1_ALERT_TOKEN` if they're empty, copies the key and the alert token to the iot layer (Node-RED's subflow, Grafana's drift alerts), copies any default question sets the project lacks, writes n8n's `p4n4 System 1`, `p4n4 Operator Intake` and `p4n4 System 1 Alerts` credentials (when n8n is enabled), pulls the checkpoint, starts the service and waits for it to be healthy |
| `disable [--purge]` | Removes the profile and stops the service. `--purge` also deletes the downloaded model, after asking |
| `pull` | Downloads the pinned checkpoint (about 650 MB) and checks it against the image's hashes |
| `status` | Health, checkpoint, revision, image, and each set's version, thresholds, examples and last evaluation; marks a set whose file changed since the service loaded it (`GET /v1/sets`) |
| `decide SET --state FILE\|- [--json]` | Runs a question set on one JSON state and prints the answers |
| `eval [SET] [--no-gates] [--fit-thresholds] [--fit-calibration [--save-calibration]]` | Runs each set's `<name>.eval.jsonl`, prints coverage, answered accuracy, confident-wrong rate, a confusion matrix, the split by source and by language, and latency, and records the run for `status`. With two or more languages in a file, every gate must pass in each. Exits `5` when a gate fails. The `--fit-*` options print proposed thresholds, or temperatures per laya bucket and per question, and never write them. A question in a set's `eval.rank` is gated on its AUC instead. `--save-calibration` (with `--fit-calibration`, over every set) writes the bucket temperatures to `config/system1/calibration/site.json` and sets `SYSTEM1_CALIBRATION` |
| `label SET --input FILE [--write]` | Runs a set on past events, JSON lines `{"ts", "state", "subject"}`, and prints the labels per question. `--write` writes the answered ones to the iot layer's `ai_events` as `system1_label` points, which the System 1 dashboard shows as annotations |
| `feedback SET [--append]` | Reads the confirmations Node-RED kept with their state (n8n's Operator Intake with `FEEDBACK_STATES = true`) and prints them as evaluation lines (`source: feedback`), or appends the new ones to the set's evaluation file. Needs the iot layer |

`decide` and `eval` talk to the service through `docker compose exec`, so the key never
leaves its container and no host port is needed. Edit a set by hand, bump its `version`,
and run `eval` again: the gates, not intuition, decide whether a set may serve.

---

## `p4n4 migrate edge`

Moves a project's legacy `edge` layer (from before p4n4-edge merged into p4n4-ai) into the
layer that replaced it. Refuses while the edge layer runs (`p4n4 down edge` first).

```bash
p4n4 migrate edge [--source-ai PATH] [--yes]
```

- **The inference runner** moves into the ai layer: its service is spliced into
  `ai/docker-compose.yml` (comments kept), `runner/`, `edge-impulse/` and `onnx/` are copied
  (models already there kept), the edge `.env` keys ai lacks are appended, and `ei-runner`
  joins `COMPOSE_PROFILES`. Without an ai layer, one is scaffolded from p4n4-ai with only the
  runner enabled. A project left with one layer is flattened to the root.
- **Other services** (a template's own edge layer) become the extras layer, keeping the
  Compose project name, so their volumes stay.
- What the move replaces is kept in `edge.pre-migrate/`. Keys set differently in both `.env`
  files keep the ai layer's value and are listed.

Until then, the edge layer keeps running and every command prints a reminder. See
[Migrating from the edge stack](../guides/migrating-from-edge.md).

---

## `p4n4 ei`

> Not implemented in v0.1: `deploy`, `run` and `status` print "not yet implemented" and
> exit `1`. Manage the runner with the AI stack's `make deploy-model` / `make up` and its
> [HTTP API](../stacks/edge-inference.md#http-api) instead.

| Subcommand | Planned behavior |
|------------|------------------|
| `deploy MODEL` | Copy a `.eim` or `.onnx` model to its models directory and recreate the runner |
| `run` | Start the inference runner |
| `status` | Show runner container status |

`list`, `infer`, `update` and `info` are specified for v0.2 ([F-0.2.4](../decisions/specs.md#f-024--p4n4-ei-subcommand-expansion)).

---

## `p4n4 template`

| Subcommand | Description |
|------------|-------------|
| `search [QUERY]` | List templates from the community registry |
| `install NAME` | Install a template into the current project |
| `list` | Show installed templates |
