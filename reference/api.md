# REST API

`p4n4-api` (`api/`) is the HTTP gateway for a p4n4 project, written in Python
(FastAPI) on top of [`p4n4-lib`](architecture.md). It serves one project, read from
`P4N4_PROJECT_DIR`, and both flat and multi-layer layouts ([ADR-002](../decisions/adr/ADR-002.md)).
Its main client is the [dashboard](dashboard.md).

v0.1 covers sign-in and a **read-only** view of the project, its stacks and the host.
Device registry, telemetry, inference, agent and MQTT endpoints are planned; the design
target is in the [p4n4-api README](https://github.com/raisga/p4n4-api) and its `TODO.md`.

## Run

v0.1 runs on the host, not in a container: stack status shells out to
`docker compose ps` in each stack directory, so it needs the Docker CLI and daemon access.

```bash
cd api
uv venv
uv pip install -e ../lib -e .

export P4N4_PROJECT_DIR=~/projects/my-project
uv run p4n4-api users add admin --role admin   # prompts for a password (10+ characters)
uv run p4n4-api                                # serves on 127.0.0.1:8000
```

```bash
TOKEN=$(curl -s -X POST http://localhost:8000/api/v1/auth/token \
  -H 'Content-Type: application/json' \
  -d '{"username": "admin", "password": "<password>"}' | jq -r .access_token)

curl -H "Authorization: Bearer $TOKEN" http://localhost:8000/api/v1/stacks
```

Interactive docs are at `/swagger-ui` and the spec at `/openapi.json`.

## Endpoints

🔒 needs a `normie`, `operator` or `admin` access token (`Authorization: Bearer <token>`).

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/health` | Liveness probe |
| `GET` | `/ready` | Readiness: the project resolves and Docker is reachable (`503` otherwise) |
| `GET` | `/api/v1/version` | API, API-version and `p4n4-lib` versions |
| `POST` | `/api/v1/auth/token` | Username + password → access token (1 h) + refresh token (7 d) |
| `POST` | `/api/v1/auth/refresh` | Refresh token → new pair; each refresh token works once |
| `POST` | `/api/v1/auth/logout` | Revoke the refresh token and its whole sign-in |
| `GET` | `/api/v1/auth/me` | 🔒 Signed-in user and role |
| `GET` | `/api/v1/project` | 🔒 Manifest (with its optional `template` and `dashboard` blocks, `null` when absent), layout (`flat`/`multi`) and per-stack directories |
| `GET` | `/api/v1/project/validate` | 🔒 `p4n4 validate` checks as `{ok, passed, errors}` |
| `GET` | `/api/v1/stacks` | 🔒 Compose service status for every enabled stack (`503` if Docker is unreachable) |
| `GET` | `/api/v1/stacks/{stack}` | 🔒 One stack (`iot`, `ai`, `edge`); `404` if not enabled |
| `GET` | `/api/v1/edge/metrics` | 🔒 Host CPU, memory, disk, temperature, uptime and load |

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
isn't one, for example in a VM. `inference_ms` will be added with the edge runner proxy.

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
immediately. A `device` role arrives with the device registry.

## Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `P4N4_PROJECT_DIR` | server's cwd | Project to serve; walks up to `.p4n4.json` |
| `P4N4_API_HOST` | `127.0.0.1` | Bind address |
| `P4N4_API_PORT` | `8000` | Listen port |
| `P4N4_API_CORS_ORIGINS` | *(empty: CORS off)* | Comma-separated browser origins, e.g. `http://localhost:8088`. No credentials are allowed, so `*` is accepted |
| `P4N4_API_DATA_DIR` | `~/.local/share/p4n4-api` | SQLite database (`api.db`: users, refresh tokens) and generated JWT secret. Created owner-only; back it up |
| `P4N4_API_JWT_SECRET` | generated into the data dir | HS256 key, 32+ characters. Changing it signs everyone out |
| `P4N4_API_AUTH` | — | `off` treats every request as an admin. **Development only**; logs a warning |

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
