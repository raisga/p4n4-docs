# Security

## Default credentials

All default passwords (`changeme`, `adminpassword`) are **placeholders only**.
Change them before first run:

```bash
# Auto-generate strong secrets
p4n4 secret rotate
```

Or set them manually in `.env`.

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

By default, all services bind to `0.0.0.0`. In production:

- Place a reverse proxy (Nginx, Caddy, Traefik) in front with TLS termination.
- Restrict inbound ports with firewall rules.
- Do not expose InfluxDB (8086), Mosquitto (1883/9001) or the inference runner (8080)
  directly to the internet. The runner's HTTP API has no authentication; in production,
  remove its host-port binding and reach it only from `p4n4-net`.

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

Rotate all secrets periodically:

```bash
p4n4 secret rotate
p4n4 down && p4n4 up
```

## `.env` file

- Never commit `.env` to version control — it is in `.gitignore`.
- Only `.env.example` (with placeholder values) is committed.
- Use `p4n4 secret show` to audit current secrets (values are masked).
- Multi-layer projects have one `.env` per stack (`iot/.env`, `ai/.env`, `edge/.env`).
  `p4n4 secret rotate` updates all of them and writes the same new value to keys
  shared across stacks (e.g. `INFLUXDB_TOKEN`), so never rotate one file by hand —
  the stacks would drift out of sync.

## Reporting vulnerabilities

See [SECURITY.md](https://github.com/raisga/p4n4/blob/main/SECURITY.md) in the umbrella repo.
