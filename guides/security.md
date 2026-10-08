# Security

## Default credentials

All default passwords and tokens in the `.env.example` files (`adminpassword`,
`p4n4-stack-token`, `lettapassword`, `change-me-32-char-encryption-key`) are public
placeholders. `p4n4 init` generates real values. Without the CLI, replace every one of them
in `.env` **before the first start**: InfluxDB, Grafana and n8n keep the values they first
start with, and `p4n4 secret rotate` doesn't change those afterwards (see
[Secret rotation](#secret-rotation)).

## Node-RED login

The Node-RED editor and Admin API require a login (`NODE_RED_USER` /
`NODE_RED_PASSWORD` in the IoT `.env`). `p4n4 init` generates the password and
`p4n4 secret rotate` replaces it. If `NODE_RED_PASSWORD` is empty, every login is refused,
so projects created before this change must add both keys before they can use the editor.

## REST API

`p4n4-api` binds `127.0.0.1` by default and requires a JWT for everything except
`/health`, `/ready`, `/api/v1/version` and the docs. Create users with
`p4n4-api users add`. Never set `P4N4_API_AUTH=off` outside development: it treats every
request as an admin. Its data directory (`~/.local/share/p4n4-api`) holds the user database
and JWT signing key, so keep it private and back it up. See the
[REST API reference](../reference/api.md#security).

## Network exposure

p4n4 0.2.x is meant for development and trusted local networks.

The services without authentication are published on `127.0.0.1` only: Ollama (11434,
`OLLAMA_BIND`), Letta (8283, `LETTA_BIND`), the inference runner (8080, `EI_RUNNER_BIND`)
and p4n4-api (8000, `P4N4_API_HOST`). Keep them there unless the network is trusted, and use
a reverse proxy with TLS for remote access rather than changing the bind address.

Mosquitto (1883/9001), InfluxDB (8086), Node-RED (1880), Grafana (3000), n8n (5678) and the
dashboard (8088) are published on every interface. In production:

- Place a reverse proxy (Nginx, Caddy, Traefik) in front with TLS termination.
- Restrict inbound ports with firewall rules. On Linux, traffic to published Docker ports
  bypasses ufw rules ([Docker and ufw](https://docs.docker.com/engine/network/packet-filtering-firewalls/#docker-and-ufw)),
  so filter in the `DOCKER-USER` chain or at the cloud provider.
- Do not expose InfluxDB or Mosquitto directly to the internet.

## Dashboard

The dashboard signs in with p4n4-api accounts: `admin` accounts get the admin view,
`operator` accounts the power view and `normie` accounts the normie view. The API enforces
each role, so a normie account can read status and chat with agents but nothing more, even
with a modified dashboard. It falls back to a role picker only when the API
runs with `P4N4_API_AUTH=off` or can't be reached, so keep auth on. For its web service
(`p4n4-dashboard`, port 8088):

- Optionally add HTTP basic auth as a second layer: `make htpasswd NAME=admin` in `dashboard`, then
  `DASHBOARD_BASIC_AUTH='admin:$2y$…'` (single-quoted) in its `.env`. It covers the UI and
  every proxied route, and the credentials never reach the services behind it.
- Publish it on a LAN interface only: `DASHBOARD_BIND=<lan-ip>` in its `.env`.
- Keep it off the internet. If it must be reachable, serve it over HTTPS
  (`make up-tls`) with basic auth on.
- Keep p4n4-api on the Docker bridge (`P4N4_API_HOST=172.17.0.1`) rather than `0.0.0.0`,
  so only the dashboard's proxy reaches it.
- `GRAFANA_ALLOW_EMBEDDING=true` (needed for its Grafana tab) lets any site frame
  Grafana. Pair it with a firewall that keeps Grafana on the LAN.
- **HTTPS:** behind TLS, browsers block `http://` iframes and images (mixed content).
  Proxy Grafana through the dashboard (`GRAFANA_UPSTREAM`, `DASHBOARD_GRAFANA_BASE=/grafana/`,
  and `GRAFANA_SUB_PATH=/grafana/` in the IoT `.env`), and use HTTPS cameras.

## Mosquitto authentication

The default `mosquitto.conf` ships with `allow_anonymous true` for easy development.
For production, disable anonymous access and use password + ACL files:

```conf
allow_anonymous false
password_file /mosquitto/config/passwd
acl_file /mosquitto/config/acl
```

Generate the password file:

```bash
mosquitto_passwd -c config/mosquitto/passwd <username>
```

Then set matching `MQTT_USER` / `MQTT_PASSWORD` in the edge stack's `.env` and in any
Node-RED or n8n MQTT nodes.

## Secret rotation

`p4n4 secret rotate` replaces the secrets their services read at every start, then the
next `p4n4 up` applies them:

| Layer | Rotated by `p4n4 secret rotate` | Changed in the service (setup-only) |
|-------|---------------------------------|-------------------------------------|
| iot | `NODE_RED_PASSWORD` | `INFLUXDB_PASSWORD`, `INFLUXDB_TOKEN`, `GRAFANA_PASSWORD` |
| ai | `LETTA_SERVER_PASSWORD` | `N8N_ENCRYPTION_KEY` (and the unused `N8N_BASIC_AUTH_PASSWORD`) |

```bash
p4n4 secret rotate
p4n4 down && p4n4 up
```

The setup-only secrets are read once, when the service first creates its data. A new value
in `.env` alone would lock every client out of InfluxDB and Grafana, and n8n refuses to
start with an encryption key that doesn't match the one its credentials were stored with,
so `rotate` leaves them alone and says so. Change them in the service, then write the new
value to `.env` by hand so `p4n4 secret show` and the other layers stay in step:

- **InfluxDB password:** `docker exec -it p4n4-influxdb influx user password -n admin`,
  then set `INFLUXDB_PASSWORD` in the IoT `.env`.
- **InfluxDB token:** create an all-access token
  (`docker exec p4n4-influxdb influx auth create --org <org> --all-access`), set it as
  `INFLUXDB_TOKEN` in **every** layer's `.env`, restart with `p4n4 down && p4n4 up`, then
  delete the old token (`influx auth list`, `influx auth delete --id <id>`).
- **Grafana password:** `docker exec p4n4-grafana grafana cli admin reset-admin-password <new>`,
  then set `GRAFANA_PASSWORD` in the IoT `.env`.
- **n8n encryption key:** keep it. Changing it means exporting the credentials decrypted
  (`n8n export:credentials --decrypted`), starting n8n with an empty data volume and the
  new key, and importing them again.

## `.env` file

- Never commit `.env` to version control — it is in `.gitignore`.
- Only `.env.example` (with placeholder values) is committed.
- Use `p4n4 secret show` to audit current secrets (values are masked).
- Multi-layer projects have one `.env` per stack (`iot/.env`, `ai/.env`, `edge/.env`).
  `p4n4 init` writes the same value to keys shared across stacks (e.g. `INFLUXDB_TOKEN`),
  and `p4n4 secret rotate` updates every file. When you change a shared key by hand, change
  it in every layer's `.env`, or the stacks drift out of sync.

## Reporting vulnerabilities

**Do not open a public issue for security vulnerabilities.** Report them through a
[private security advisory](https://github.com/raisga/p4n4/security/advisories/new) on the
umbrella repo; this covers every repository in the `raisga` organisation. Include a
description, steps to reproduce, the potential impact and a suggested fix if you have one.
