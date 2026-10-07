# Dashboard Service

The dashboard layer (`p4n4-dashboard`) is the platform's web UI: one container that serves
the Flutter web build of [p4n4-dashboard](../reference/dashboard.md) and proxies the
services it talks to, so a browser on any LAN device opens `http://<host>:8088` and needs
nothing else. The same code also builds the desktop and mobile apps.

It attaches to `p4n4-net` and starts after the other stacks (`iot → ai → edge →
dashboard`). It only reads from them, so any subset of stacks works.

## Services

| Service | Container | Image | Port | Role |
|---------|-----------|-------|------|------|
| dashboard | `p4n4-dashboard` | `ghcr.io/raisga/p4n4-dashboard:<version>` (nginx, non-root) | 8088 | Web UI and same-origin proxy |

## Routes

| Path | Goes to | Notes |
|------|---------|-------|
| `/` | The web app | Unknown paths fall back to `index.html` |
| `/api/`, `/health` | p4n4-api (`P4N4_API_UPSTREAM`) | Service status, project info, edge metrics |
| `/ollama/` | Ollama (`OLLAMA_UPSTREAM`) | Agent chat; buffering off for streaming |
| `/letta/` | Letta (`LETTA_UPSTREAM`) | Agent chat |
| `/grafana/` | Grafana (`GRAFANA_UPSTREAM`) | Optional; `404` until set. Grafana must serve from `/grafana/` (`GRAFANA_SUB_PATH`) |
| `/config.json` | Rendered at start | The app's default addresses (`DASHBOARD_*` variables) |
| `/healthz` | nginx | Health check |

By default Grafana and cameras aren't proxied: the browser loads them directly (`<iframe>`, `<img>`)
from `DASHBOARD_HOST`, or from the host the page was served from. Grafana must allow
framing: `GRAFANA_ALLOW_EMBEDDING=true` in the IoT `.env` (`p4n4 init` sets it when the
dashboard layer is enabled).

Upstreams are resolved per request, so the UI keeps working while a stack is down. Its
calls to that service get `502`, and the matching tab shows an error.

## Setup

```bash
p4n4 init my-project --layer iot,dashboard     # or --layer all
cd my-project && p4n4 up                       # prints the dashboard URL
```

Or on its own, from the dashboard repository: `cp .env.example .env && docker compose up -d`.

p4n4-api runs on the host in v0.1, and the container reaches it at
`host.docker.internal`. Start the API where the container can reach it:
`P4N4_API_HOST=172.17.0.1` (the Docker bridge) or `0.0.0.0`. Create dashboard accounts on
the API (`p4n4-api users add <name> --role admin|operator|normie`); admins get the admin
view, operators the power view and normies the normie view.

## Environment variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `DASHBOARD_VERSION` | `1.1.0` | Image tag (a p4n4-dashboard release) |
| `DASHBOARD_PORT` | `8088` | Host port |
| `DASHBOARD_BIND` | `0.0.0.0` | Interface to publish on; a LAN address keeps it off other networks |
| `DASHBOARD_HOST` | *(empty)* | Host the browser uses for Grafana and direct links; empty means the page's host |
| `P4N4_API_UPSTREAM` | `http://host.docker.internal:8000` | p4n4-api |
| `OLLAMA_UPSTREAM` | `http://p4n4-ollama:11434` | Ollama |
| `LETTA_UPSTREAM` | `http://p4n4-letta:8283` | Letta |
| `GRAFANA_UPSTREAM` | *(empty)* | Grafana, for the optional `/grafana/` route |
| `DASHBOARD_GRAFANA_BASE` | *(empty)* | `/grafana/` to make the app use that route |
| `DASHBOARD_BASIC_AUTH` | *(empty)* | htpasswd entries (`make htpasswd NAME=admin`); set, the whole UI needs a login |
| `DASHBOARD_TLS_SITE`, `DASHBOARD_TLS` | `localhost`, `internal` | HTTPS front end (`make up-tls`): the site address, and `internal` or an email for Let's Encrypt |

## White-label images

The public image carries the `p4n4` brand. For a client, build an image with their theme
(see the [template registry](../reference/template-registry.md#project-manifest)) and push
it to a private registry:

```bash
cd dashboard
make image THEME=~/projects/greenhouse     # → p4n4-dashboard:<theme id>, only that theme inside
```

Then set the project's `dashboard/docker-compose.yml` image to that tag.

## Security

- **Sign-in** uses p4n4-api accounts. Keep the API's auth on: with `P4N4_API_AUTH=off` (or
  the API unreachable) the sign-in screen falls back to a role picker. Optionally add
  **basic auth** (`DASHBOARD_BASIC_AUTH`) and bind it to a LAN address (`DASHBOARD_BIND`).
  See the [security guide](../guides/security.md#dashboard).
- **HTTPS.** `make up-tls` adds a Caddy front end (`docker-compose.tls.yml`), using its
  own CA on a LAN or Let's Encrypt for a public domain. Serve Grafana through `/grafana/`
  as well, or browsers block its `http://` frame on the `https://` page.
- **Hardening.** The container runs nginx as a non-root user with a read-only filesystem,
  no capabilities and `no-new-privileges`. It enforces a Content Security Policy that
  stops other sites from framing it. Base images are pinned by digest, Dependabot keeps
  them current, and CI scans every image with Trivy.
- **Fixed upstreams.** The proxy only forwards to the three upstreams above, never to
  arbitrary URLs, so it can't be used as an open relay.

## Resource limits (emulator)

`p4n4-emu` treats `dashboard` as a stack with a small share (5 % CPU, 1 % memory of the
profile). On a Raspberry Pi 5 that's about 80 MB, well above what nginx serving static
files needs.
