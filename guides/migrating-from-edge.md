# Migrating from the edge stack

The edge stack (`p4n4-edge`) merged into the AI stack ([ADR-005](../decisions/adr/ADR-005.md)).
Its inference runner is the AI stack's `ei-runner` service, with the same container
(`p4n4-ei-runner`), port (8080 on `127.0.0.1`), MQTT topics (`sensors/+/raw` in,
`inference/<device-id>/result` out), InfluxDB bucket (`ai_events`) and variables. The
repository became `p4n4-extras`, for optional services.

Nothing breaks until you migrate: a project's `edge` layer keeps running, and p4n4 prints a
reminder.

## p4n4 projects

```bash
cd my-project
p4n4 down edge                 # migrate refuses while the edge layer runs
p4n4 migrate edge              # shows what moves, then asks
p4n4 validate
p4n4 up ai
```

What `p4n4 migrate edge` does:

| The project has | Result |
|---|---|
| An ai layer | The runner's service is added to `ai/docker-compose.yml` (its comments kept), `runner/`, `edge-impulse/` and `onnx/` move into `ai/` (models already there are kept), the edge `.env` keys ai lacks are appended, and `ei-runner` joins `COMPOSE_PROFILES` (unless the edge layer had it turned off) |
| No ai layer | An ai layer is scaffolded from p4n4-ai (`--source-ai PATH` offline) with only `ei-runner` enabled, its `.env` filled from the edge layer's, its secrets generated, and your models copied in |
| Only the ai layer left | It's flattened to the project root, as `p4n4 init` lays out a single-layer project; it keeps its `COMPOSE_PROJECT_NAME`, so its volumes |
| An edge layer without the runner (a template's own, e.g. go2rtc video) | It's renamed `extras/`, keeping its Compose project name, so its volumes stay |

Nothing is deleted: what the move replaces stays in `edge.pre-migrate/`. Keys both `.env`
files set to different values keep the ai layer's value, and the command lists them. Delete
`edge.pre-migrate/` once `p4n4 up ai` works.

The runner's volume becomes `<project>-ai_ei-runner-data`. The runner keeps nothing in it,
so nothing needs copying; remove the old one with
`docker volume rm <project>-edge_ei-runner-data`. The runner image is rebuilt on the next
`p4n4 up` (`p4n4 up ai --build` to force it).

## Without the CLI

```bash
cd p4n4-edge && docker compose down && cd ..
cp -r p4n4-edge/edge-impulse/models/. p4n4-ai/edge-impulse/models/
cp -r p4n4-edge/onnx/models/. p4n4-ai/onnx/models/
```

Then in `p4n4-ai/.env`, copy over the runner's values from `p4n4-edge/.env` (`MODEL_BACKEND`,
`EI_MODEL_FILE`, `ONNX_MODEL_FILE`, `ONNX_LABELS`, `MQTT_*`, `INFLUXDB_BUCKET_AI_EVENTS`,
`EI_RUNNER_BIND`, `EI_API_KEY`; the AI stack's `.env.example` lists them), add `ei-runner` to
`COMPOSE_PROFILES`, and start it:

```bash
cd p4n4-ai && docker compose up -d --build
curl -s localhost:8080/health
```

## Monorepo checkouts

The submodule moves from `stacks/edge` to `stacks/extras`, following the repository's new
name. After pulling:

```bash
git submodule sync && git submodule update --init stacks/extras
```

## Changed names

| Before | After |
|---|---|
| `p4n4 init --layer edge`, `--source-edge` | `--layer ai --ei-runner`; `--source-extras` for the extras layer |
| `--layer all` = iot, ai, edge, dashboard | iot, ai (with `ei-runner`), dashboard; `all,extras` adds extras |
| `p4n4 up edge` | `p4n4 up ai` |
| `stacks/edge` (`raisga/p4n4-edge`) | `stacks/ai` for the runner; `stacks/extras` (`raisga/p4n4-extras`) |
| Volume `<project>-edge_ei-runner-data` | `<project>-ai_ei-runner-data` (unused by the runner) |
| `p4n4-emu up --stack edge` | `--stack ai` (the runner's limits moved with it) |
