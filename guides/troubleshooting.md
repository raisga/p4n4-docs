# Troubleshooting

Common failures and their fixes, grouped by where they show up. Start with these two:

```bash
p4n4 validate          # manifest, required files, .env keys, per layer
p4n4 status            # every container, including stopped and crashed ones
```

`docker logs <container>` (all containers are named `p4n4-*`) usually says the rest. From a
source checkout, `scripts/doctor` also checks tools, submodules, free ports and `p4n4-net`
([Development](development.md)).

## Starting the stacks

| Symptom | Cause and fix |
|---------|---------------|
| `p4n4 up` lists containers held by another project | Container names and host ports are fixed, so one project runs per host. Stop the other one with the command it prints, or `p4n4 down --all` |
| A port is already allocated | Something on the host uses it. Ollama installed natively also listens on 11434: set `OLLAMA_PORT` in the AI `.env`. For the others, stop the host service or change the port mapping in a `docker-compose.override.yml` |
| `network p4n4-net … incorrect label` | `p4n4-net` was created by hand without Compose's label. `p4n4 up` recreates it while no container uses it; otherwise `docker network rm p4n4-net` after stopping everything. To create it by hand, add `--label com.docker.compose.network=p4n4-net` |
| `Could not create the p4n4-net network: … Pool overlaps with other one on this address space` | Another network (a VPN's, or another Compose project's) holds `172.20.0.0/16`; `docker network inspect $(docker network ls -q) --format '{{.Name}} {{range .IPAM.Config}}{{.Subnet}}{{end}}'` shows which. Set `P4N4_SUBNET` to a free range (e.g. `172.28.0.0/16`) in `iot/.env` (the project's `.env` in a single-layer project) and run `p4n4 up` again. p4n4 0.4.0 and earlier ignore it when they create the network: create it by hand with `docker network create --subnet <range> --label com.docker.compose.network=p4n4-net p4n4-net` |
| `p4n4 up extras` does nothing | No extra is enabled: the layer is skipped until `COMPOSE_PROFILES` in `extras/.env` lists one ([Extras](../stacks/extras.md)) |
| Every command reminds you about an `edge` layer | The project predates the merge of the edge stack into ai. Run `p4n4 down edge`, then `p4n4 migrate edge` ([guide](migrating-from-edge.md)) |
| `p4n4 add`, `remove`, `upgrade`, `ei` or `template` prints "not yet implemented" | They are stubs. The [CLI reference](../reference/cli-reference.md) gives the workaround for each |
| A value with `$` or `#` in `.env` comes out truncated | Compose interpolates `$` and treats ` #` as a comment. Wrap the value in single quotes (`p4n4 init` does this for the values it writes) |
| Containers stay down after a reboot | `RESTART_POLICY` is `no`, or Docker doesn't start on boot (`sudo systemctl enable docker`) |

## Logins and secrets

| Symptom | Cause and fix |
|---------|---------------|
| Node-RED refuses every login | `NODE_RED_PASSWORD` is empty in the IoT `.env`, which refuses all logins by design. Set it and `p4n4 up` |
| InfluxDB rejects the token (401), Grafana shows no data, after editing `.env` | `INFLUXDB_TOKEN`, `INFLUXDB_PASSWORD` and `GRAFANA_PASSWORD` are setup-only: the services kept their first values. Put the old values back, or change them in the service ([Secret rotation](security.md#secret-rotation)) |
| The ai layer's n8n or runner can't write to InfluxDB | `INFLUXDB_TOKEN` or `INFLUXDB_ORG` differs between the iot and ai `.env` files. They must match ([Configuration](../reference/configuration.md#shared-across-layers)) |
| n8n doesn't start: "Mismatching encryption keys" | `N8N_ENCRYPTION_KEY` changed after n8n stored credentials. Restore the old key; changing it needs an export and import ([Secret rotation](security.md#secret-rotation)) |
| The Node-RED System 1 subflow always returns `msg.error` | `SYSTEM1_API_KEY` is empty or differs from the ai layer's. `p4n4 ai system1 enable` copies it |
| `docker: permission denied … docker.sock` | Your user isn't in the `docker` group, or the shell predates joining it. `sudo usermod -aG docker $USER`, then log in again |

## MQTT

| Symptom | Cause and fix |
|---------|---------------|
| A device gets "not authorised" | `MQTT_PASSWORD` is set in the IoT `.env`, so devices need a login: `MQTT_USER` or an account in `config/mosquitto/passwd`. `docker logs p4n4-mqtt` shows the accounts and whether an ACL is on |
| A device connects but its messages go nowhere | An ACL in `config/mosquitto/acl` doesn't grant the account that topic. Mosquitto drops a denied publish without telling an MQTT 3.1.1 client |
| Readings arrive but aren't stored | The topic isn't `sensors/<device-id>/<measurement>` (e.g. `sensors/temperature`) or the payload isn't a JSON object. Node-RED's debug sidebar shows the warning ([MQTT topics](../reference/mqtt-topics.md)) |
| The broker exits at start with `p4n4 mqtt: …` | Its entrypoint refused a setting: a value with a space, `MQTT_PASSWORD` without `MQTT_USER`, a TLS certificate without its key, or a certificate file it can't find in `config/mosquitto/certs/` |
| The external-broker bridge never connects | `make bridge-status` in the iot layer, then `docker logs p4n4-mqtt`. Check `MQTT_REMOTE_TLS` and the port (8883 with TLS, 1883 without), and quote a password containing `$` or `#` |

## Node-RED, Grafana and the dashboard

| Symptom | Cause and fix |
|---------|---------------|
| Deploying from the Node-RED editor fails with `EACCES` | `config/node-red/flows/` must be writable by the container's user, uid 1000. `sudo chown -R 1000:1000 config/node-red/flows` |
| The dashboard's Grafana tab is blank | `GRAFANA_ALLOW_EMBEDDING=false` in the IoT `.env`. `p4n4 init` sets it with the dashboard layer; set it to `true` and recreate Grafana |
| Grafana or cameras are blank behind HTTPS | Mixed content: browsers block `http://` frames on an `https://` page. Proxy Grafana through the dashboard (`GRAFANA_SUB_PATH=/grafana/`), and use HTTPS cameras ([Security](security.md)) |
| A dashboard tab shows an error, the API answers `502` | The service behind that tab isn't running or isn't reachable from the dashboard's proxy ([Dashboard service](../stacks/dashboard.md)) |
| The dashboard offers a role picker instead of sign-in | p4n4-api is unreachable or runs with `P4N4_API_AUTH=off`. Keep auth on outside development |

## AI stack

| Symptom | Cause and fix |
|---------|---------------|
| Ollama or another container is killed under load | Out of memory: `docker inspect <container> --format '{{.State.OOMKilled}}'`. Use a smaller model, keep `OLLAMA_MAX_LOADED_MODELS=1`, or run fewer AI services |
| The inference runner stops right after starting | A model is selected or present but doesn't load. It refuses to publish mock results as real ones: `docker logs p4n4-ei-runner` says why ([Edge Inference](../stacks/edge-inference.md)) |
| `.eim` model: `Exec format error` | The `.eim` was built for another architecture. Download the build for the host (x86_64 or AARCH64) from Edge Impulse Studio |
| The runner reports `"influxdb_ok": false` | Its `INFLUXDB_*` values don't match the iot layer's, or InfluxDB isn't up yet. Results still go to MQTT meanwhile, and its log warns about the failed writes |
| System 1 doesn't start | `SYSTEM1_API_KEY` is empty, or the checkpoint isn't downloaded or doesn't match the image's pins. `p4n4 ai system1 pull`, then `p4n4 ai system1 status` |

## Emulator and hardware

| Symptom | Cause and fix |
|---------|---------------|
| `p4n4-emu up` with an ARM profile refuses to start | QEMU isn't registered on the host: `p4n4-emu setup --arch arm64`, or pass `--native` ([Emulator](../reference/emulator.md)) |
| Everything is very slow under `rpi4` / `rpi5` | QEMU arm64 is 3–10× slower than native; LLM inference under it is impractical. Run the ai layer with `--native` |
| `Cannot determine SOC peripheral base address` on a Pi 5 | The original `RPi.GPIO` can't drive the Pi 5. Install `python3-rpi-lgpio` ([Hardware](../reference/hardware.md)) |
| The hardware daemon's LED blinks slowly although everything works | A service in `P4N4_SERVICES` that this node doesn't run. Narrow the list in `/etc/p4n4/hwd.env` |

Cloud and Jetson hosts have troubleshooting tables of their own: [AWS](aws.md#troubleshooting),
[Google Cloud](gcp.md), [Azure](azure.md), [DigitalOcean](digitalocean.md),
[Jetson](jetson.md#8-troubleshooting), [Raspberry Pi 5](raspberry-pi.md#troubleshooting).

Found something not listed? Open an issue in the repository that owns it; security problems go
through a [private advisory](security.md#reporting-vulnerabilities).
