# Extras

The extras stack (`p4n4-extras`, `stacks/extras`) holds **optional services** that aren't
part of the p4n4 core. The core is the [IoT stack](iot-stack.md) and the
[AI stack](ai-stack.md): device → MQTT → Node-RED → InfluxDB → Grafana, local GenAI, and
[edge inference](edge-inference.md). Everything in extras is opt-in: nothing starts by
default, and no core service, flow or workflow depends on it, so a project without the
extras layer, or with it and nothing enabled, works the same.

It's the repository that was `p4n4-edge`, whose inference runner moved into the AI stack
([ADR-005](../decisions/adr/ADR-005.md)).

## Core or extra?

| | Core (iot, ai) | Extras |
|---|---|---|
| Belongs there when | p4n4's value depends on it | It's useful, but the pipeline works without it |
| Starts | The stack's default `COMPOSE_PROFILES` | Only when `COMPOSE_PROFILES` lists it |
| Depends on | Other core services | Core services, over `p4n4-net` |
| Depended on by | Anything | Nothing in the core |

Typical extras: a reverse proxy with TLS, a container manager, image auto-updates, metrics
(Prometheus, cAdvisor), a vector database, a chat UI, speech (Whisper, Piper), extra MQTT
bridges, backup jobs, video (go2rtc).

## Services

None ship yet: the stack has the structure and a skeleton extra. Projects and templates add
their own, and the community catalog planned in the
[AI harness design](../decisions/ai-harness.md#5-extras-community-stacks) (OBD-II, Modbus,
CAN bus, …) will live here, one directory each. The `mqtt-influx-grafana-ollama-go2rtc`
template runs go2rtc video and a device uplink as its extras layer.

## Using it

```bash
p4n4 init my-project --layer all,extras     # or --layer iot,ai,extras; never part of `all`
cd my-project
# add a service (below), list it in extras/.env COMPOSE_PROFILES, then:
p4n4 up extras
```

`p4n4 up` (and `p4n4-emu up`) skips the layer while nothing is enabled, instead of failing
with Compose's "no service selected".

## Adding an extra

Each extra is a directory of its own, which the layer's `docker-compose.yml` pulls in with
`include:`. Every extra runs in the layer's one Compose project (on `p4n4-net`, with one
`.env`), and paths in an extra's compose file are relative to its directory. `_template/` is
a complete extra to copy, valid on its own (CI checks it):

1. `cp -r _template <name>`, then rename `my-extra` / `MY_EXTRA` in `<name>/`.
2. Add `- <name>/docker-compose.yml` under `include:` in `docker-compose.yml`.
3. Append `<name>/.env.example` to `.env.example`, and add the profile to its `Available:`
   list. `scripts/check_env_example.py` checks every `${VAR}` in every included file.
4. Put config files in `<name>/config/`, mounted read-only.
5. Check it: `make config`, then `COMPOSE_PROFILES=<name> make up`.

Rules: its own profile (opt-in), a pinned image (never `latest`), container name
`p4n4-<name>`, host ports on `127.0.0.1` by default, secrets only in `.env`, no core service
depending on it, and GPU or device settings in `docker-compose.override.yml` so it still
runs on a Raspberry Pi 5 or an x86 host without them.
