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
p4n4 init my-project --layer all           # iot + ai + edge + dashboard
p4n4 init my-project --no-interactive      # skip wizard, use defaults
p4n4 init my-project --source-iot ../p4n4-iot   # scaffold from a local checkout (offline)
```

The wizard prompts for the InfluxDB org, timezone and service passwords (blank means
generate one), including the Node-RED editor password, which is written with
`NODE_RED_USER=admin` to the IoT `.env`. The edge layer gets the compose file, runner,
model directories and `.env`, sharing the InfluxDB token, org and timezone with the other
layers. Unknown layer names are rejected. With the IoT layer, the wizard also offers to
pull topics from an [external MQTT broker](../stacks/iot-stack.md#external-mqtt-broker)
(host, login, topics, local prefix, TLS).

| Flag | Description |
|------|-------------|
| `--layer <names>` | Layers to enable: `iot`, `ai`, `edge`, `dashboard`, `all`, or comma-separated |
| `--no-interactive` | Skip the wizard and auto-generate all secrets |
| `--source-iot PATH` / `--source-ai PATH` / `--source-edge PATH` | Use a local stack checkout instead of cloning |
| `--mqtt-remote HOST[:PORT]` | Bridge topics in from an external MQTT broker (IoT layer) |
| `--mqtt-remote-user` / `--mqtt-remote-password` | Its login; the password can come from `P4N4_MQTT_REMOTE_PASSWORD` instead |
| `--mqtt-remote-topics LIST` | Comma-separated topic filters to pull in (default `sensors/#`) |
| `--mqtt-remote-tls` / `--mqtt-remote-ca FILE` | Connect over TLS; optional CA certificate, copied into the project |

**Layout:** single-layer projects place `docker-compose.yml`, `config/`, `scripts/`, and
`.env` at the project root. Multi-layer projects give each layer its own subdirectory
(`<project>/iot/`, `<project>/ai/`, `<project>/edge/`) so the stacks run as separate Compose projects;
shared `.env` keys (e.g. `INFLUXDB_TOKEN`) are written identically to every layer.
Each layer's `.env` sets `COMPOSE_PROJECT_NAME`: `<project>` for single-layer projects,
`<project>-<layer>` for multi-layer ones (lowercased, other characters replaced by `-`). That
keeps each project's volumes separate. Don't change it on an existing project: Compose would
switch to new, empty volumes.
See [Getting Started](../getting-started.md#project-layout).

---

## `p4n4 add STACK`

Add a stack to an existing project.

```bash
p4n4 add ai
p4n4 add edge --path /srv/my-project
```

---

## `p4n4 remove STACK`

Remove a stack from an existing project.

```bash
p4n4 remove edge
p4n4 remove ai --yes       # skip confirmation
```

---

## `p4n4 up [STACK]`

Start one or all enabled stacks in dependency order (`iot` → `ai` → `edge`).

```bash
p4n4 up                    # all stacks
p4n4 up iot                # IoT stack only
p4n4 up --no-detach        # foreground mode
```

---

## `p4n4 down [STACK]`

Stop one or all stacks, in reverse dependency order (dependents stop before `iot`
removes the shared network).

```bash
p4n4 down
p4n4 down ai
p4n4 down --volumes        # also remove volumes
```

---

## `p4n4 status`

Show container status for all enabled stacks. Multi-layer projects print one
table per stack.

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
directory (`iot/…`, `ai/…`, `edge/…`).

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

`rotate` replaces whichever of these keys are present:

- **IoT:** `INFLUXDB_PASSWORD`, `INFLUXDB_TOKEN`, `GRAFANA_PASSWORD`, `NODE_RED_PASSWORD`
- **AI:** `LETTA_SERVER_PASSWORD`, `N8N_BASIC_AUTH_PASSWORD`, `N8N_ENCRYPTION_KEY`

`MQTT_REMOTE_PASSWORD` (an external broker's login) appears in `show`, fully masked, but is
never rotated: the external broker issued it.

In multi-layer projects, `rotate` updates every layer's `.env` and writes the **same**
new value to keys shared across stacks (e.g. `INFLUXDB_TOKEN` in both `iot/.env` and
`ai/.env`), so cross-stack credentials never drift.

---

## `p4n4 ei`

> Not implemented in v0.1: `deploy`, `run` and `status` print "not yet implemented" and
> exit `1`. Manage the runner with the edge stack's `make deploy-model` / `make up` and its
> [HTTP API](../stacks/edge-stack.md#http-api) instead.

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
