# REST API

`p4n4-api` (`api/`) is the HTTP gateway for a p4n4 project, written in Python
(FastAPI) on top of [`p4n4-lib`](architecture.md). It serves one project, read from
`P4N4_PROJECT_DIR`, and both flat and multi-layer layouts ([ADR-002](../decisions/adr/ADR-002.md)).
Its main client is the [dashboard](dashboard.md).

v0.1 covers sign-in and users, the device registry, the project and its stacks (status,
control as background jobs, logs, an audit log), telemetry (ingest, query and a live
stream), the edge runner's inference, Ollama and Letta agents, MQTT publishing and host
metrics. Prometheus metrics are still planned; see the
[p4n4-api README](https://github.com/raisga/p4n4-api) and its `TODO.md`.

## Run

On the host, stack status and control shell out to `docker compose` in each stack directory,
so the API needs the Docker CLI and daemon access:

```bash
cd api
uv venv
uv pip install -e ../lib -e .

export P4N4_PROJECT_DIR=~/projects/my-project
uv run p4n4-api users bootstrap                # first admin; the generated password is shown once
uv run p4n4-api                                # serves on 127.0.0.1:8000
```

To work on the API together with the rest of the checkout, use `scripts/dev api` (see the
[Development guide](../guides/development.md)).

In a container, the image `ghcr.io/raisga/p4n4-api` (amd64, arm64) runs as a non-root user
on `p4n4-net` as `p4n4-api`, with its upstream URLs pointing at the stacks' containers:

```bash
cp .env.example .env          # set P4N4_PROJECT_DIR (absolute path)
docker compose up -d
docker compose exec api p4n4-api users bootstrap
```

The project is mounted read-only at the same path as on the host, and users, devices, the
audit log and the JWT secret live in the `p4n4-api-data` volume. Docker access is **off** by
default (`P4N4_API_DOCKER=off`): `/stacks` returns `503` and stack control and logs are
unavailable. `docker-compose.docker.yml` turns it on by mounting the Docker socket, which
gives the container root-equivalent control of the host. The port is published on
`127.0.0.1:8000` only (`P4N4_API_BIND`, `P4N4_API_PUBLISH_PORT`); the dashboard reaches the
container as `P4N4_API_UPSTREAM=http://p4n4-api:8000`.

```bash
TOKEN=$(curl -s -X POST http://localhost:8000/api/v1/auth/token \
  -H 'Content-Type: application/json' \
  -d '{"username": "admin", "password": "<password>"}' | jq -r .access_token)

curl -H "Authorization: Bearer $TOKEN" http://localhost:8000/api/v1/stacks
```

Interactive docs are at `/swagger-ui` and the spec at `/openapi.json`.

## Endpoints

🔒 needs a `normie`, `operator` or `admin` access token (`Authorization: Bearer <token>`);
🔑 needs `admin`.

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/health` | Liveness probe |
| `GET` | `/ready` | Readiness probe: project found and Docker reachable (`503` otherwise) |
| `GET` | `/api/v1/version` | API, API-version and `p4n4-lib` versions |
| `POST` | `/api/v1/auth/token` | Username + password → access token (1 h) + refresh token (7 d); or a device's `api_key` → access token (1 h) |
| `POST` | `/api/v1/auth/refresh` | Refresh token → new pair (each refresh token works once) |
| `POST` | `/api/v1/auth/logout` | Revoke the refresh token and its whole sign-in, access tokens included |
| `POST` | `/api/v1/auth/password` | Change your own password (🔒 any role): signs out every other sign-in and returns a new pair |
| `GET` | `/api/v1/auth/me` | The signed-in user and role (🔒 any role) |
| `GET` `POST` | `/api/v1/users` | 🔑 List users / create one (`{username, password, role}`, role defaults to `operator`) |
| `GET` `PATCH` `DELETE` | `/api/v1/users/{username}` | 🔑 Read, change (`{role?, password?}`) or remove a user. The last admin can't be demoted or removed (`409`) |
| `GET` | `/api/v1/devices` | 🔒 Device registry, paginated (`limit`, `offset`) |
| `POST` | `/api/v1/devices` | 🔑 Register a device; the response holds its API key, shown once |
| `GET` `PATCH` `DELETE` | `/api/v1/devices/{id}` | 🔒 Read; 🔑 change (`name`, `description`, `enabled`) or remove |
| `POST` | `/api/v1/devices/{id}/key` | 🔑 Rotate the API key (old key and its tokens stop at once) |
| `GET` | `/api/v1/project` | 🔒 Manifest (including its optional `template` and `dashboard` blocks), layout (`flat`/`multi`), and per-stack directories |
| `GET` | `/api/v1/project/validate` | 🔒 Run `p4n4_lib.validate` checks; returns `{ok, passed, errors}` |
| `GET` | `/api/v1/stacks` | 🔒 Compose service status per stack (`503` if Docker is unreachable) |
| `GET` | `/api/v1/stacks/{stack}` | 🔒 One stack's service status (404 if not enabled) |
| `POST` | `/api/v1/stacks/{stack\|all}/{up\|down\|restart}` | 🔑 Run a stack action as a background job (`202` + `Location`) |
| `POST` | `/api/v1/stacks/{stack}/services/{service}/restart` | 🔑 Restart one service, as a job |
| `GET` | `/api/v1/stacks/{stack}/logs` | 🔑 Container logs (`tail`, `service`); `follow=true` streams them as server-sent events |
| `GET` | `/api/v1/jobs`, `/api/v1/jobs/{id}` | 🔒 Job status and Compose output |
| `GET` | `/api/v1/audit` | 🔑 Audit log: stack actions, user and device changes |
| `POST` | `/api/v1/telemetry` | Device token only: store a batch of readings in InfluxDB, then publish them to MQTT |
| `GET` | `/api/v1/telemetry` | 🔒 Stored readings (filters, time range, windowed aggregates) |
| `GET` | `/api/v1/telemetry/stream` | 🔒 Live readings from MQTT as server-sent events |
| `GET` | `/api/v1/inference/runner` | 🔒 The edge runner's backend (Edge Impulse, ONNX or mock), model and counters |
| `POST` | `/api/v1/inference` | 🔒 Classify a feature vector with the runner's model |
| `GET` | `/api/v1/inference/results` | 🔒 Results the runner's pipeline stored (`ai_events`) |
| `GET` | `/api/v1/dashboard/views` | 🔒 Tabs each p4n4-dashboard view shows and their order: `{tab_order, power_tabs, normie_tabs}` (null = the dashboard's default) |
| `PUT` | `/api/v1/dashboard/views` | 🔑 Set them for every device (audited) |
| `GET` | `/api/v1/agents/config` | 🔒 The assistant everyone chats with: `{backend, model, agent_id}` (null = the first one listed), and `updated_at`/`updated_by` (null until someone chooses; p4n4-dashboard then offers its brand's default) |
| `PUT` | `/api/v1/agents/config` | 🔒 `operator` or `admin`: choose the assistant (audited) |
| `GET` | `/api/v1/agents/models` | 🔒 Ollama models |
| `POST` | `/api/v1/agents/chat` | 🔒 Ollama chat, streamed as Ollama's NDJSON. Normies: the assistant's model only, no `options` (`403 assistant_restricted`) |
| `POST` | `/api/v1/agents/generate` | 🔒 `operator` or `admin`: Ollama one-shot generation |
| `GET` | `/api/v1/agents` | 🔒 Letta agents |
| `POST` | `/api/v1/agents/{id}/chat` | 🔒 Message a Letta agent (password kept server side). Normies: the assistant's agent only |
| `POST` | `/api/v1/mqtt/publish` | 🔒 Publish an MQTT message (allowed topics only) |
| `GET` | `/api/v1/edge/metrics` | 🔒 CPU, memory, disk, temperature, uptime and load of the host (the edge device) |
| `GET` | `/swagger-ui`, `/openapi.json` | Interactive docs / OpenAPI spec |

### Errors

Every error (4xx/5xx) has the body `{"error": {"code": "...", "message": "..."}}`. `code` is
stable and meant for programs (`not_found`, `forbidden`, `validation_error`,
`rate_limited`, `user_exists`, `last_admin`, `assistant_restricted`, …); `message` is for
people. Every response carries an `X-Request-ID` header (the caller's or a generated one),
which the server's log lines repeat.

### Stack status

Each service has `name`, `state` and `health`, plus `image`, `version` (the image tag),
`status` (Docker's text, e.g. `Up 2 hours (healthy)`), `ports`, `exit_code` (stopped
services), and `started_at` / `uptime_s` (running services). Fields Docker doesn't provide,
for example with standalone `docker-compose` v1, are `null`.

### Edge metrics

`GET /api/v1/edge/metrics` returns the dashboard's [edge metrics contract](dashboard.md#edge-metrics-contract):
`cpu_percent`, `mem_percent`, `mem_used_mb`, `mem_total_mb`, `disk_percent` (of `/`),
`uptime_s`, `load` (1/5/15 min) and `temp_c`. `temp_c` comes from a CPU/SoC sensor
(`cpu_thermal` on a Raspberry Pi, `coretemp`/`k10temp` on x86) and is left out when there
isn't one, for example in a VM. `inference_ms` is the edge runner's `last_latency_ms`, when the runner is reachable.

## Users and roles

```bash
p4n4-api users list
p4n4-api users add alice                  # operator by default; --role normie|admin
p4n4-api users passwd alice               # also signs alice out everywhere
p4n4-api users role alice admin
p4n4-api users remove alice
echo "$PASSWORD" | p4n4-api users add alice --password-stdin
p4n4-api users dev                        # development: admin, power, normie
```

For development, `P4N4_API_DEV_USERS=true` (or `p4n4-api users dev`) creates one account per
dashboard view (`admin`, `power`, `normie`) with the public password `p4n4`, keeping any
that already exist. Never use it on a reachable deployment.

| Role | Can |
|------|-----|
| `normie` | Read project, stack, edge, telemetry, inference results and job progress; chat with Ollama models and Letta agents |
| `operator` | Everything a normie can, plus the device registry, MQTT publish, inference and one-shot generation |
| `admin` | Everything an operator can, plus users, devices, stack control, container logs and the audit log |

The role is read from the database on every request, so role changes and removals apply
immediately. Registered devices sign in with their API key (`POST /api/v1/auth/token`) and
get a token that can only send telemetry (`POST /api/v1/telemetry`).

## Configuration

| Variable | Description |
|----------|-------------|
| `P4N4_PROJECT_DIR` | p4n4 project directory to serve (walks up to `.p4n4.json`; default: the server's cwd) |
| `P4N4_API_HOST` | Bind address (default: `127.0.0.1`) |
| `P4N4_API_PORT` | HTTP listen port (default: `8000`) |
| `P4N4_API_CORS_ORIGINS` | Comma-separated browser origins allowed to call the API, e.g. `http://localhost:8088`. Empty (default) disables CORS. No credentials are allowed, so `*` is accepted |
| `P4N4_API_DATA_DIR` | Where the API keeps its SQLite database (`api.db`: users, refresh tokens) and generated JWT secret (default: `$XDG_DATA_HOME/p4n4-api`, i.e. `~/.local/share/p4n4-api`). Back it up; created owner-only |
| `P4N4_API_JWT_SECRET` | HS256 signing key, at least 32 characters (`openssl rand -hex 32`). Default: generated once into `$P4N4_API_DATA_DIR/jwt_secret`. Changing it signs everyone out |
| `P4N4_API_TRUSTED_PROXIES` | Comma-separated reverse-proxy IPs or networks whose `X-Forwarded-For` is believed, so the sign-in rate limit applies per client instead of to everyone behind the proxy. For the dashboard's nginx container reaching a host-run API: `172.16.0.0/12` (Docker's default bridge networks). Default: none; the header is ignored |
| `P4N4_API_LOG_FORMAT` | `text` (default) or `json` (one object per line, for log collectors). Applies to `p4n4-api serve` |
| `P4N4_API_LOG_LEVEL` | `debug`, `info` (default), `warning` or `error` |
| `P4N4_API_INFLUXDB_URL` | InfluxDB 2 base URL (default `http://localhost:8086`, the IoT stack's published port) |
| `P4N4_API_INFLUXDB_TOKEN`, `_ORG`, `_BUCKET` | Default to `INFLUXDB_TOKEN`, `INFLUXDB_ORG` and `INFLUXDB_BUCKET` from the project's `iot/.env` (the stack's own), then `ming` / `raw_telemetry`. Without a token, telemetry endpoints return `503 influxdb_not_configured` |
| `P4N4_API_MQTT_ENABLED` | `false` turns the MQTT connection off (no publishing, no live stream). Default `true` |
| `P4N4_API_MQTT_HOST`, `_PORT` | Broker (default `localhost:1883`, the IoT stack's) |
| `P4N4_API_MQTT_USERNAME`, `_PASSWORD` | Broker credentials, if it requires them |
| `P4N4_API_EDGE_RUNNER_URL` | The edge stack's inference runner (default `http://localhost:8080`, its published port) |
| `P4N4_API_MQTT_PUBLISH_ALLOW` | Topic filters `POST /mqtt/publish` may use, comma-separated (`+`, `#` wildcards). Default `#` |
| `P4N4_API_MQTT_PUBLISH_DENY` | Topic filters it may not use; wins over allow. Default `sensors/#,inference/#` (device data, which Node-RED stores). `none` for no denials |
| `P4N4_API_OLLAMA_URL` | Ollama (default `http://localhost:11434`, the ai stack's published port) |
| `P4N4_API_LETTA_URL` | Letta (default `http://localhost:8283`) |
| `P4N4_API_LETTA_PASSWORD` | Letta's server password; default `LETTA_SERVER_PASSWORD` from the project's `ai/.env`. Sent only from the API to Letta |
| `P4N4_API_INFLUXDB_AI_EVENTS_BUCKET` | Where the runner stores results; default `INFLUXDB_BUCKET_AI_EVENTS` from the project's `edge/.env`, then `ai_events` |
| `P4N4_API_DOCKER` | `off` when the API has no Docker access (a container without the socket): stack status, control and logs return `503`, and `/ready` doesn't require Docker. Default `on` |
| `P4N4_API_DISK_PATH` | Filesystem `/edge/metrics` reports as `disk_percent` (default `/`) |
| `P4N4_API_AUTH` | `off` disables auth: every request is treated as an admin. **Development only**; logs a warning at startup |
| `P4N4_API_DEV_USERS` | `true` creates `admin`, `power` and `normie` (password `p4n4`) at startup if missing; see [Users and roles](#users-and-roles). **Development only**; logs a warning at startup |

The container image adds `P4N4_API_BIND` and `P4N4_API_PUBLISH_PORT` (where Compose
publishes the port) and defaults `P4N4_API_TRUSTED_PROXIES` to `172.16.0.0/12`.

## Security

- Passwords are hashed with argon2id. Unknown usernames take as long to reject as wrong
  passwords.
- Refresh tokens are single-use. Reusing one revokes every token from that sign-in, and a
  password change signs the user out everywhere. Signing out revokes refresh tokens; an
  access token stays valid until it expires (at most 1 h).
- `/auth/token` and `/auth/refresh` allow 10 attempts per client address, then one every
  6 s (`429` with `Retry-After`).
- CORS is off by default. Auth uses the `Authorization` header, so cookies are never
  allowed cross-origin.
- The API binds `127.0.0.1` by default. To reach it from other machines, set
  `P4N4_API_HOST` and put it behind a TLS reverse proxy (see the
  [Security guide](../guides/security.md)).
