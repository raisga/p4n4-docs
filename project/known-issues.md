# Known Issues

Seven problems found on 2026-09-23 while researching the AI harness design (`docs/decisions/ai-harness.md`), plus problem 8, found on 2026-09-27 while testing the fix for problem 7. All eight problems are fixed. Each section uses the fields of the [bug report template](https://github.com/raisga/p4n4/blob/main/.github/ISSUE_TEMPLATE/bug_report.yml), so it can be filed as an issue in the owning repository. The exception is problem 6, a security issue: report it through a [private security advisory](https://github.com/raisga/p4n4/security/advisories/new), as `SECURITY.md` asks. G-numbers refer to the gap table in §8 of the harness design.

- **Checked against:** `stacks/iot` bef0bb0, `stacks/ai` 564b150, `stacks/edge` cf90d2d, `cli` bc9dea8, `lib` 705c393, `tools/emu` 87a896c and `docs` 62dc808.
- **Environment:** Problem 3 was reproduced by running the CLI from source with Python 3.13 on Windows. The transform output in problem 2 comes from running the Node-RED function code in Node.js. The rest comes from reading the code, plus Node-RED's documentation (problem 6) and source code (problem 7); no containers were started.

| # | Problem | Repository | Severity | Status |
|---|---------|------------|----------|--------|
| 1 | Node-RED writes to a hardcoded InfluxDB org | `raisga/p4n4-iot` | High | Fixed |
| 2 | All `sensors/*` readings share one measurement and field | `raisga/p4n4-iot` | High | Fixed |
| 3 | `p4n4 init --layer all` silently skips edge | `raisga/p4n4-cli`, `raisga/p4n4-lib` | Medium | Fixed |
| 4 | The edge runner has no inference API | `raisga/p4n4-edge` | Low | Fixed |
| 5 | `make deploy-model` says to run `make restart`, which doesn't load the new model | `raisga/p4n4-edge` | Medium | Fixed |
| 6 | The Node-RED editor has no login | `raisga/p4n4-iot` | Critical | Fixed |
| 7 | Deploying from the Node-RED editor fails | `raisga/p4n4-iot` | High | Fixed |
| 8 | Node-RED's MQTT inputs never connect | `raisga/p4n4-iot` | High | Fixed |

**Critical:** anyone who can reach the host can run code in the stack. **High:** telemetry is lost or unusable, or a core workflow fails. **Medium:** a command reports success but leaves the project broken. **Low:** a specified feature is missing.

---

## 1. Node-RED writes to a hardcoded InfluxDB org

- **Affected stack:** p4n4-iot
- **Where:** `stacks/iot/config/node-red/flows.json:123` and `:223`

### Description

Both InfluxDB write nodes hardcode the org and bucket in their URL:

```text
http://influxdb:8086/api/v2/write?org=ming&bucket=raw_telemetry&precision=ms
http://influxdb:8086/api/v2/write?org=ming&bucket=sandbox&precision=ms
```

The rest of the stack follows `INFLUXDB_ORG`. `p4n4 init` prompts for it (`cli/p4n4/commands/init.py:99`), InfluxDB is initialised with it (`stacks/iot/docker-compose.yml:43`), and the Grafana data sources and bucket scripts read it (`stacks/iot/config/grafana/provisioning/datasources/datasources.yml:19`, `stacks/iot/scripts/init-buckets.sh:21`). Node-RED receives it as well (`docker-compose.yml:77`) but doesn't use it. With any org other than `ming`, InfluxDB rejects every write with 404 because the organisation doesn't exist, so no telemetry or sandbox data is stored.

On the production path the failure is silent. The write nodes don't raise errors (`"senderr": false` at `flows.json:129` and `:229`), and the *Production writes* debug node ships disabled (`"active": false` at `flows.json:142`).

### Steps to reproduce

```bash
p4n4 init demo --layer iot    # enter "acme" as the InfluxDB organisation
cd demo
p4n4 up
docker run --rm --network p4n4-net eclipse-mosquitto:2 \
  mosquitto_pub -h mqtt -t sandbox/sensors/temperature \
  -m '{"value": 22.1, "unit": "C", "device": "s1"}'
```

The Node-RED debug sidebar shows `404` from *Sandbox writes*, and the `sandbox` bucket stays empty. Writes to `raw_telemetry` from `sensors/#` and `inference/#` fail the same way.

### Expected behaviour

Node-RED writes to the org and buckets set in `.env`.

### Suggested fix

- In both function nodes, build `msg.url` from `env.get("INFLUXDB_ORG")` and the bucket variables, each passed through `encodeURIComponent`. Clear the `url` field of both http request nodes so they use `msg.url`.
- Pass `INFLUXDB_SANDBOX_BUCKET` to the Node-RED container. Only `INFLUXDB_BUCKET` is passed today (`docker-compose.yml:74-78`).
- Set `senderr` to `true` and add a Catch node so failed writes show up.

**Related:** G5.

**Status:** Fixed. Both function nodes build `msg.url` from `INFLUXDB_ORG` and `INFLUXDB_BUCKET` / `INFLUXDB_SANDBOX_BUCKET`, the http request nodes have an empty URL and `senderr: true`, and a Catch node feeds an active *Write errors* debug node. Compose now passes `INFLUXDB_SANDBOX_BUCKET` to Node-RED. The tracked scaffold at `cli/test-project/` still has the old flows.

---

## 2. All `sensors/*` readings share one measurement and field

- **Affected stack:** p4n4-iot
- **Where:** `stacks/iot/config/node-red/flows.json:102` (*Format for InfluxDB (iot)*) and `:202` (the sandbox copy); `stacks/iot/config/grafana/provisioning/dashboards/json/iot-overview.json:92`

### Description

The transform maps every `sensors/...` topic to the measurement `sensor_data`. It tags only `device` and `model`, turns the other scalar keys into fields and discards the topic suffix. Given payloads in the emulator's format (`tools/emu/p4n4_emu/sim/sensor_sim.py:43-45`), it produces:

```text
sensors/temperature -> sensor_data,device=emu-sensor-0 value=21.4,unit="C"
sensors/humidity    -> sensor_data,device=emu-sensor-0 value=48.2,unit="%"
sensors/pressure    -> sensor_data,device=emu-sensor-0 value=1012.8,unit="hPa"
```

All three readings land in the same series and field, and only the `unit` string tells them apart. The transform sets no timestamp, so InfluxDB stamps each point when the write arrives, at millisecond precision. If two readings from one device arrive in the same millisecond, the later one overwrites the earlier.

The provisioned *Sensor Data* panel (`iot-overview.json:92`) filters only on `_measurement == "sensor_data"` and runs `aggregateWindow(fn: mean)`. `mean` doesn't accept the string field `unit`, so the query fails with `unsupported input type for mean aggregate: string`. Even without `unit`, the panel would average temperature, humidity and pressure together. The stack's own `make test-mqtt` and `make test-sandbox` publish the same shape (`stacks/iot/Makefile:191-215`).

### Steps to reproduce

```bash
p4n4 init demo --layer iot --no-interactive    # default org, so problem 1 doesn't interfere
cd demo
p4n4 up
docker run --rm --network p4n4-net eclipse-mosquitto:2 \
  mosquitto_pub -h mqtt -t sensors/temperature -m '{"value": 23.5, "unit": "C", "device": "s1"}'
docker run --rm --network p4n4-net eclipse-mosquitto:2 \
  mosquitto_pub -h mqtt -t sensors/humidity -m '{"value": 65.3, "unit": "%", "device": "s1"}'
```

In Grafana, the *Sensor Data* panel on *IoT Overview* shows the Flux error. In Explore, both readings are in `sensor_data`, field `value`.

### Expected behaviour

Readings can be queried by device and sensor type, and the provisioned panel renders.

### Suggested fix

- Keep the topic suffix, as a tag (for example `sensor=temperature`) or as the measurement. Which one depends on the canonical topic scheme. Specs §8.6 defines `sensors/<device-id>/<measurement>` (`docs/decisions/specs.md:1054`), while the emulator and the test targets publish `sensors/<measurement>` with the device in the payload.
- Restrict the panel query to numeric fields (for example `r._field == "value"`) and group by device and sensor.
- While rewriting the transform, escape tag values and string fields as line protocol requires. A device named `kitchen sensor` currently produces the invalid line `sensor_data,device=kitchen sensor value=20,unit="C"`.

**Related:** G5, and open question 3 in the harness design.

**Status:** Fixed, following specs §8.6 (`sensors/<device-id>/<measurement>`).
- **Node-RED:** both transforms read the device and measurement from the topic. Readings stay in `sensor_data`, tagged `device` and `sensor`. Payload `device` and `model` keys are ignored for sensor topics. Topics with a different depth, such as `sensors/temperature`, are dropped with a warning. Measurement names, tag keys and values, field keys and string fields are escaped, newlines are removed and non-finite numbers are dropped. `kitchen sensor` now produces `device=kitchen\ sensor`. `inference/*` keeps its payload tags, now escaped.
- **Grafana:** the *Sensor Data* panel filters on `_field == "value"` and groups by `device` and `sensor`.
- **Publishers:** `make test-mqtt` and `make test-sandbox` publish on spec topics. So does the emulator, which no longer puts `device` in the payload.
- **Edge runner:** subscribes to `sensors/+/raw` and takes the device from the topic. It falls back to the payload for other topics, and it ignores payloads that aren't JSON objects instead of raising.
- **Docs:** the iot README's topic section, which recommended a `{site}/{device_type}/{device_id}/{measurement}` scheme the flows never subscribed to, now documents the spec's. The device account in `acl.example` can write only `sensors/iot-device-001/+`. The emulator README, guide and docs page are updated.
- **Inference topic:** results follow §8.6 too. The runner publishes to `inference/<device-id>/result`, from the template `MQTT_TOPIC_RESULTS=inference/{device}/result`. A device id taken from a payload has `/`, `+` and `#` replaced, so it stays one topic level. Node-RED tags inference points with `device` from the topic and drops other `inference/` shapes with a warning. The n8n *Alert Enrichment* trigger subscribes to `inference/+/result`, and `make test-mqtt`, `make test-sandbox` and edge's `make test-inference` use the new topics.
- **Not changed:** the transform sets no timestamp, but readings from different sensors no longer share a series, so they can't overwrite each other.

---

## 3. `p4n4 init --layer all` silently skips edge

- **Affected stack:** p4n4-cli (the fix also touches p4n4-lib)
- **Where:** `cli/p4n4/commands/init.py:77` and `:166-172`; `lib/p4n4_lib/layers.py:75-78`

### Description

`--layer all` expands to `iot`, `ai` and `edge` (`init.py:77`), but `init` scaffolds only iot and ai (`init.py:166-170`) and then records all three layers in `.p4n4.json` (`init.py:172`). There is no `--source-edge` option. The edge layer defines no copy paths, required files or env keys (`layers.py:75-78`), so `p4n4 validate` has nothing to check and passes. `--layer edge` on its own creates only `.p4n4.json`, yet the command exits 0 and suggests `p4n4 up`.

### Steps to reproduce

```bash
p4n4 init demo --layer all --no-interactive
cd demo
p4n4 validate    # All checks passed.
p4n4 up edge     # Error: Stack 'edge' not found in this project.
```

The project contains only `ai/`, `iot/` and `.p4n4.json`, and the manifest lists `"layers": ["iot", "ai", "edge"]`. Add `--source-iot` and `--source-ai` to use local checkouts instead of cloning.

### Expected behaviour

`init` scaffolds the edge stack. Failing that, it reports an error or a warning and leaves edge out of the manifest.

### Suggested fix

- Give the edge layer copy paths, required files and env keys, for example `docker-compose.yml`, `runner`, `edge-impulse`, `onnx` and `scripts` from `stacks/edge`.
- Scaffold edge in `init` and add a `--source-edge` option.
- Record only the layers that were created.

**Related:** G3.

**Status:** Fixed. The edge layer in `lib/p4n4_lib/layers.py` now copies `docker-compose.yml`, `runner`, `edge-impulse` and `onnx`, requires those files plus `.env`, and requires `MODEL_BACKEND`, `MQTT_HOST`, `MQTT_PORT`, `INFLUXDB_TOKEN`, `INFLUXDB_ORG`, `INFLUXDB_BUCKET_AI_EVENTS` and `TZ`. `p4n4 init` scaffolds edge, sharing the InfluxDB token, org and timezone with the other layers, and has a `--source-edge` option. It also rejects unknown layer names, so the manifest records only layers that were created. Scaffolding now skips `__pycache__` in copied directories. The CLI's CI clones p4n4-edge for the tests. Push p4n4-edge's runner changes (problem 4) before releasing p4n4-lib, because `p4n4 init` clones p4n4-edge.

---

## 4. The edge runner has no inference API

- **Affected stack:** p4n4-edge
- **Where:** `stacks/edge/runner/runner.py:125-143`

### Description

The runner's HTTP handler serves only `GET /health` and `GET /`. Other paths return 404, and because there is no `do_POST`, any POST returns 501. Specs §8.5 lists `POST /api/v1/infer` and `GET /api/v1/info` (`docs/decisions/specs.md:1045-1046`), and the planned `p4n4 ei infer` (F-0.2.4) depends on the first (`specs.md:465`, `:472`, `:497`). Specs §4.2 lists `ei infer` as not yet implemented, which is why this is rated low. The same line implies that `ei deploy`, `ei run` and `ei status` work, but they only print "not yet implemented" (`specs.md:317`, `cli/p4n4/commands/ei.py:14-34`).

Without an endpoint, the only way to test a model is to publish on the production input topic `sensors/raw` (`runner.py:84`). Each test sample then has side effects:

- Node-RED stores the sample in `raw_telemetry`, because it subscribes to `sensors/#` (`stacks/iot/config/node-red/flows.json:50`).
- The runner publishes the result to `inference/results` and writes it to `ai_events` (`runner.py:434-437`).
- Node-RED stores the result in `raw_telemetry` too (`inference/#`, `flows.json:69`), and the n8n *Alert Enrichment* workflow sends results with confidence below 0.7 to Ollama (`stacks/ai/config/n8n/workflows/alert-enrichment.json:6` and `:32-33`).

### Steps to reproduce

With the IoT and edge stacks running:

```bash
curl -i -X POST http://localhost:8080/api/v1/infer -d '{"values": [1.2, 4.5, 7.8]}'    # 501
curl -i http://localhost:8080/api/v1/info                                          # 404
```

### Expected behaviour

`POST /api/v1/infer` returns a result for the posted sample without publishing or storing it, and `GET /api/v1/info` returns the backend and model details.

### Suggested fix

Add both endpoints to `_HealthHandler`, reusing `_run_inference` and `_state`. If MQTT is meant to stay the only inference path, amend specs §8.5 and F-0.2.4 instead.

**Related:** G4, and open question 4 in the harness design.

**Status:** Fixed. `_HealthHandler` serves `GET /api/v1/info` (backend, model file, model details read at load time, ONNX labels and MQTT topics) and `POST /api/v1/infer`. The inference endpoint takes `{"values": [...], "device"?}`, returns the same fields as an `inference/results` message and doesn't publish, store or count the result. Invalid bodies get 400, a missing `Content-Length` 411 and bodies over 1 MiB 413. A lock now serializes model calls, because the HTTP thread and the MQTT loop can run inference at the same time. The edge README documents both endpoints. The `ei deploy` / `ei run` / `ei status` mismatch in specs §4.2 is unchanged.

---

## 5. `make deploy-model` says to run `make restart`, which doesn't load the new model

- **Affected stack:** p4n4-edge
- **Where:** `stacks/edge/Makefile:189-190` and `:194-195`; `stacks/edge/README.md:187` and `:224`

### Description

After copying a model, `make deploy-model` tells users to set `EI_MODEL_FILE` or `ONNX_MODEL_FILE` in `.env` and run `make restart`. The README gives the same steps. `make restart` runs `docker compose restart` (`Makefile:71-73`), which restarts the existing container with its old environment. The model paths from `.env` are fixed in the container's environment when the container is created (`stacks/edge/docker-compose.yml:20` and `:23`), so the runner keeps the old path:

- If the old model file is still there, the runner keeps using it.
- If not, the runner logs that the model wasn't found (`stacks/edge/runner/runner.py:214` and `:256`) and falls back to mock mode (`runner.py:458-469`), publishing simulated results. The only signs are log lines and `"mode": "mock"` on `/health`.

Changes to `MODEL_BACKEND` and `ONNX_LABELS` are ignored the same way (`docker-compose.yml:18` and `:24`).

### Steps to reproduce

In `stacks/edge`, with the IoT stack running:

```bash
make up
make deploy-model MODEL=path/to/new-model.onnx
echo "ONNX_MODEL_FILE=new-model.onnx" >> .env    # as the command instructs
make restart
docker compose exec ei-runner printenv ONNX_MODEL_PATH    # still the old path
curl http://localhost:8080/health                        # "mode": "mock" unless an old model is present
```

### Expected behaviour

Following the printed steps loads the new model.

### Suggested fix

In the Makefile hints and the README, tell users to run `make up` instead. `docker compose up -d` recreates containers whose configuration has changed. Alternatively, make `restart` run `docker compose up -d --force-recreate`.

**Related:** §4 of the harness design, where `edge_deploy_model` recreates the container for this reason.

**Status:** Fixed. The `deploy-model` hints and both README walkthroughs now say `make up`, and the README command list notes that `make restart` doesn't apply `.env` changes.

---

## 6. The Node-RED editor has no login

- **Affected stack:** p4n4-iot (the fix also touches p4n4-cli and p4n4-lib)
- **Where:** `stacks/iot/config/node-red/settings.js` (no `adminAuth`) and `stacks/iot/docker-compose.yml:72-73`
- **Report:** through a [private security advisory](https://github.com/raisga/p4n4/security/advisories/new), not a public issue (`SECURITY.md:11-15`)

### Description

`settings.js` doesn't set `adminAuth`, so Node-RED serves the editor and the Admin API without a login. Node-RED warns that in this state "anyone who can access its IP address can access the editor and deploy changes", which "is only suitable if you are running on a trusted network" ([Securing Node-RED](https://nodered.org/docs/user-guide/runtime/securing-node-red)). Compose publishes port 1880 on every interface of the host (`docker-compose.yml:72-73`), so the editor is open to anyone who can reach the host. On Linux, traffic to published ports also bypasses ufw rules ([Docker and ufw](https://docs.docker.com/engine/network/packet-filtering-firewalls/#docker-and-ufw)).

Without a login, anyone can read and deploy flows and, because the palette manager is enabled (`settings.js:32`), install nodes from npm. Deploying a flow and installing a node both run code in the Node-RED container: function nodes run JavaScript, the core exec node runs shell commands, and installed nodes are loaded into the runtime. That code can read the container's environment, including `INFLUXDB_TOKEN` (`docker-compose.yml:76`). That token is also InfluxDB's admin token (`docker-compose.yml:45`), so it gives full control of InfluxDB.

The stack's other web UIs have logins: InfluxDB (`docker-compose.yml:41-42`) and Grafana (`:107-108`), with passwords that `p4n4 init` generates or prompts for (`cli/p4n4/commands/init.py:87-112`). Nothing tells users that Node-RED has none. The *Default Credentials* table (`stacks/iot/README.md:253-256`) lists only InfluxDB and Grafana, and `SECURITY.md:42` advises changing the default passwords in `.env`, which doesn't cover Node-RED.

### Steps to reproduce

```bash
# On the machine running the stack:
p4n4 init demo --layer iot --no-interactive
cd demo
p4n4 up

# From another machine on the same network, with HOST set to the first machine's address:
curl http://$HOST:1880/auth/login    # {}: no active authentication
curl -i http://$HOST:1880/flows      # 200 and the flow configuration, without credentials
```

Opening `http://$HOST:1880` in a browser shows the editor without a login prompt.

### Expected behaviour

The editor and the Admin API require a login, as InfluxDB and Grafana do, with a password that `p4n4 init` writes to `.env`.

### Suggested fix

- In `settings.js`, set `adminAuth` with `type: "credentials"` and custom `users` and `authenticate` functions ([custom user authentication](https://nodered.org/docs/user-guide/runtime/securing-node-red#custom-user-authentication)) that check the username and password against environment variables, for example `NODE_RED_USER` and `NODE_RED_PASSWORD`. Compare the password in constant time, for example with `crypto.timingSafeEqual` on SHA-256 digests, and reject every login if the password variable is unset or empty. This keeps a plain password in `.env`, as for Grafana, so the CLI doesn't need bcrypt. A bcrypt hash in `.env` with the standard `users` list also works, but then `p4n4 init` has to generate the hash.
- Pass both variables to the container (`docker-compose.yml:74-78`). Add them to `.env.example` and to the *Default Credentials* table in `stacks/iot/README.md`.
- In `p4n4 init`, generate the password in the non-interactive branch (`init.py:87-96`), prompt for it after the Grafana password (`:109-112`) and add both keys to `iot_env_values` (`:134-148`). Add them to the iot `required_env_keys` (`lib/p4n4_lib/layers.py:43-52`) so `p4n4 validate` catches a missing key. Add the password to `ROTATABLE_KEYS` (`lib/p4n4_lib/secrets.py:7-16`) so `p4n4 secret rotate` refreshes it.
- Where remote access isn't needed, for example on a development machine, consider publishing the port as `"127.0.0.1:1880:1880"`.

**Related:** G8. The harness shouldn't write flows until this is fixed, and then needs a token from `POST /auth/token` first ([Admin API authentication](https://nodered.org/docs/api/admin/oauth)).

**Status:** Fixed, as suggested: `settings.js` sets `adminAuth` with custom `users` and `authenticate` functions that check `NODE_RED_USER` and `NODE_RED_PASSWORD`, comparing SHA-256 digests with `crypto.timingSafeEqual`. When the password is unset, every login is refused and Node-RED logs a warning. Compose passes both variables, with no default password. `.env.example` and the *Default Credentials* table list `admin` / `adminpassword`, like Grafana. `p4n4 init` generates or prompts for the password, `p4n4 validate` requires both keys and `p4n4 secret rotate` rotates the password. The port is still published on every interface. Tested with Node-RED 4.1.15 run from npm: `/auth/login` reports credentials, `/flows` returns 401 without a token, a wrong user or password gets 403, the right one gets a token that reads `/flows`, and `GET /` (the compose healthcheck) still returns 200. Pushing the fix publicly discloses the problem, so coordinate it with the advisory.

---

## 7. Deploying from the Node-RED editor fails

- **Affected stack:** p4n4-iot (the fix also touches p4n4-lib)
- **Where:** `stacks/iot/docker-compose.yml:82`

### Description

Compose mounts the project's `config/node-red/flows.json` over `/data/flows.json` as a single file (`docker-compose.yml:82`), while the rest of `/data` is a named volume (`:80`). Node-RED saves flows by writing `flows.json.$$$` and renaming it over `flows.json` (`writeFile` in [`localfilesystem/util.js`](https://github.com/node-red/node-red/blob/master/packages/node_modules/%40node-red/runtime/lib/storage/localfilesystem/util.js)). Linux, which also runs the containers under Docker Desktop, doesn't allow renaming onto a mount point, so the rename fails with `EBUSY` on every host. Node-RED starts deployed flows only after saving them (`setFlows` in [`flows/index.js`](https://github.com/node-red/node-red/blob/master/packages/node_modules/%40node-red/runtime/lib/flows/index.js)), so every deploy fails, the old flows keep running and the project's `flows.json` doesn't change. A Node-RED forum report shows the error for a single-file mount ([Flows.json access locked](https://discourse.nodered.org/t/flows-json-access-locked/83335)):

```text
Deploy failed: {"code":"EBUSY","message":"EBUSY: resource busy or locked, rename '/data/flows.json.$$$' -> '/data/flows.json'"}
```

Node-RED 3 and 4 both save this way, and the stack pulls `nodered/node-red:latest` (`docker-compose.yml:69`). Until this is fixed, the workaround is to edit `flows.json` on the host and restart Node-RED. The read-only `settings.js` mount (`:81`) isn't affected, because Node-RED never writes that file.

### Steps to reproduce

```bash
p4n4 init demo --layer iot --no-interactive
cd demo
p4n4 up
```

Open <http://localhost:1880>, move any node and click **Deploy**. The editor shows the `EBUSY` error above, and `iot/config/node-red/flows.json` is unchanged.

### Expected behaviour

Deploying saves the flows to the project's `flows.json` and starts them.

### Suggested fix

- Mount a directory instead of the file, so the rename stays inside one mount. For example, move the file to `config/node-red/flows/flows.json`, mount `./config/node-red/flows:/data/flows` and set `FLOWS: /data/flows/flows.json` in the container's environment. The image passes `FLOWS` to Node-RED on the command line, which overrides `flowFile` in `settings.js` ([Running under Docker](https://nodered.org/docs/getting-started/docker)).
- Node-RED then also writes `.flows.json.backup` to that directory, and `flows_cred.json` if a node has credentials. Keep both out of version control.
- On Linux, the host directory must be writable by the container's user, uid 1000 by default (same page).
- Update the path in `required_files` (`lib/p4n4_lib/layers.py:39`) and in the docs.

**Related:** G9. The harness's `nodered_flows_apply` deploys through the same Admin API, so it fails the same way.

**Status:** Fixed. The flows moved to `config/node-red/flows/flows.json`. Compose mounts `./config/node-red/flows:/data/flows` and sets `FLOWS: /data/flows/flows.json`, and `settings.js` sets `flowFile: 'flows/flows.json'` to match. `.gitignore` covers `flows/flows_cred.json` and `flows/.*.backup`, the iot README notes the uid 1000 requirement, and the path is updated in `lib/p4n4_lib/layers.py`, the CLI test and README, `docs` and the website mockups. Tested with Node-RED 4.1.15 run from npm, without Docker: a deploy through the Admin API returned 200 and saved to `flows/flows.json`, with `.flows.json.backup` next to it. Push p4n4-iot before releasing p4n4-lib: `p4n4 init` clones p4n4-iot, and a clone without the new path fails the new `required_files` check.

---

## 8. Node-RED's MQTT inputs never connect

- **Affected stack:** p4n4-iot
- **Where:** `stacks/iot/config/node-red/flows.json:10` (now `config/node-red/flows/flows.json`)

### Description

The MQTT broker config node has the id `mqtt-broker`, the same string as its type. When Node-RED 4 starts, it scans each config node's properties for the ids of other config nodes, so it reads the node's own `type` as a dependency on itself. It then drops the broker, and the three `mqtt in` nodes start without one:

```text
Error: Circular config node dependency detected: mqtt-broker
[error] [mqtt in:sensors/#] missing broker configuration
[error] [mqtt in:inference/#] missing broker configuration
[error] [mqtt in:sandbox/#] missing broker configuration
```

Node-RED subscribes to nothing, so no MQTT data reaches InfluxDB, whatever the org (problem 1) or the transform (problem 2). This was found by starting Node-RED 4.1.15 with the committed flows. It wasn't tested with Node-RED 3.

### Expected behaviour

The broker node loads and the `mqtt in` nodes subscribe.

**Status:** Fixed. The id is now `broker-mosquitto`, and the three `mqtt in` nodes point to it. Node-RED starts without the error, and the broker node tries to connect to `mqtt:1883`.

---

## Not covered here

The harness design's gap table also tracks wider work that isn't a single bug: the topic contract drifting between the specs, the flows and the n8n triggers (G5), and the docs disagreeing with the code on manifest fields and env key names (G6).
