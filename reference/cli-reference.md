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
p4n4 init my-project --layer all           # iot + ai + edge
p4n4 init my-project --no-interactive      # skip wizard, use defaults
p4n4 init my-project --source-iot ../p4n4-iot   # scaffold from a local checkout (offline)
```

| Flag | Description |
|------|-------------|
| `--layer <names>` | Layers to enable: `iot`, `ai`, `edge`, `all`, or comma-separated |
| `--no-interactive` | Skip the wizard and auto-generate all secrets |
| `--source-iot PATH` / `--source-ai PATH` | Use a local stack checkout instead of cloning |

**Layout:** single-layer projects place `docker-compose.yml`, `config/`, `scripts/`, and
`.env` at the project root. Multi-layer projects give each layer its own subdirectory
(`<project>/iot/`, `<project>/ai/`) so the stacks run as separate Compose projects;
shared `.env` keys (e.g. `INFLUXDB_TOKEN`) are written identically to every layer.
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

Validate `.p4n4.json`, required stack files, and `.env` keys. Multi-layer projects
are checked per layer directory (`iot/…`, `ai/…`).

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
| `rotate` | Re-generate all password/token values |
| `generate` | Print new secrets to stdout |

In multi-layer projects, `rotate` updates every layer's `.env` and writes the **same**
new value to keys shared across stacks (e.g. `INFLUXDB_TOKEN` in both `iot/.env` and
`ai/.env`), so cross-stack credentials never drift.

---

## `p4n4 ei`

| Subcommand | Description |
|------------|-------------|
| `deploy MODEL` | Copy a `.eim` model and restart the runner |
| `run` | Start the Edge Impulse runner |
| `status` | Show runner container status |

---

## `p4n4 template`

| Subcommand | Description |
|------------|-------------|
| `search [QUERY]` | List templates from the community registry |
| `install NAME` | Install a template into the current project |
| `list` | Show installed templates |
