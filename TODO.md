# TODO — p4n4 code review findings

Findings from a code review of raisga/p4n4 `main` at `512e385` and its public submodules, written 2026-11-02. Section 2 has one checkbox per finding: tick it when the fix lands. Section 3 gives the evidence and a fix for each, and section 5 the order to start in.

Paths are relative to the p4n4 superproject root, not to this repo, and line numbers are at the submodule commits that `512e385` pins (section 6).

## 1. Summary

p4n4 is in better shape than most projects at this stage: secrets come from a CSPRNG, no Python code in any of the repos uses `shell=True`, and the AI stack binds Ollama, Letta, the Edge Impulse runner and the agent to 127.0.0.1 by default. The API and dashboard pair argon2id, rotating refresh tokens and a strict CSP with non-root, read-only containers. The serious risks are at the edges: two shipped templates are exploitable from the LAN (code execution through go2rtc's writable API, and forgeable acknowledgement links in the n8n cold-chain flow), and one anonymous MQTT message can crash-loop the opt-in incident agent for good. In the core, the trusted-LAN stance erodes, because n8n listens on 0.0.0.0 and makes its first visitor the owner, MQTT stays anonymous after `p4n4 init`, and Compose fallbacks bring back the public example secrets when `.env` is missing. Supply chain is the main structural weakness: `init` copies four repos from unpinned `main` (one of which has no CI), `p4n4-lib` has no upper bound and has already broken CLI 0.3.0, and release jobs hand write tokens to third-party actions pinned only by tag. Correctness problems cluster in the glue between stacks, where the API's copy of start/stop lacks the CLI's fixes, shared InfluxDB settings are never compared across layers, and same-name projects share volumes. The docs are thorough but drifting, starting with a security stance that still names 0.2.x. In total there are 3 High, 21 Medium, 38 Low and 9 Info findings and no Critical; one suspected High was refuted by running `docker compose config` and survives only as L21, for older Compose versions.

## 2. Findings checklist

Grouped by severity. Each entry gives the ID and a one-line description, then the confidence and the location.

### High (3)

- [ ] **H1.** go2rtc template publishes an unauthenticated, writable API on :1984 that allows code execution
  - Confirmed · `tools/templates/projects/mqtt-influx-grafana-ollama-go2rtc/extras/docker-compose.yml:20`; `extras/config/go2rtc/video.sh:17-18,22,59-60,70`
- [ ] **H2.** Cold-chain ack links are signed with a public or empty key, so anyone who can reach n8n can forge an ack and stop escalation
  - Confirmed · `tools/templates/projects/mqtt-influx-grafana-n8n/ai/.env.example:51`; `ai/docker-compose.yml:18,54`; `ai/config/n8n/workflows/acknowledge.json:129`
- [ ] **H3.** One anonymous MQTT alert with a huge timestamp crash-loops the incident agent permanently
  - Confirmed · `stacks/ai/agent/src/request-id.ts:69`; `investigator.ts:50,270,304`; `main.ts:93,135-136`

### Medium (21)

- [ ] **M1.** n8n binds 0.0.0.0, the first visitor becomes owner, and the init password does nothing
  - Confirmed · `stacks/ai/docker-compose.yml:83-87`; `cli/p4n4/commands/init.py:357,417`
- [ ] **M2.** Compose fallbacks bring back the public example secrets when `.env` is missing or incomplete
  - Confirmed · `stacks/iot/docker-compose.yml:89,93,186`; `stacks/ai/docker-compose.yml:56,88,105`; `stacks/iot/Makefile:80-82`
- [ ] **M3.** MQTT accepts anonymous clients by default, even after `p4n4 init`
  - Confirmed · `stacks/iot/config/mosquitto/bridge.sh:118-128,147,179`; `cli/p4n4/commands/init.py:399`
- [ ] **M4.** Sign-in rate limit trusts X-Forwarded-For from all of 172.16.0.0/12
  - Confirmed · `api/docker-compose.yml:34`; `api/p4n4_api/main.py:101-107`; `dashboard/docker/nginx/proxy_common.conf:4`
- [ ] **M5.** Open SSE streams outlive sign-out, password changes and user removal
  - Confirmed · `api/p4n4_api/routes/telemetry.py:182-193`; `api/p4n4_api/routes/control.py:179-193`
- [ ] **M6.** API image resolves dependencies at build time and ignores `uv.lock`
  - Confirmed · `api/Dockerfile:20-25`
- [ ] **M7.** `p4n4 init` copies four repositories from unpinned `main`
  - Confirmed · `lib/p4n4_lib/scaffold.py:103`; `lib/p4n4_lib/sources.yaml:1-4`
- [ ] **M8.** `p4n4-lib` has no upper bound, and lib 0.4 already broke the published CLI 0.3.0
  - Confirmed · `cli/pyproject.toml:26`; `api/pyproject.toml:25`; `tools/emu/pyproject.toml:31`
- [ ] **M9.** Release jobs hand write tokens to third-party actions pinned only by tag
  - Confirmed · `api/.github/workflows/image.yml:27-31,110` (and all 21 workflows)
- [ ] **M10.** The core IoT stack has no CI
  - Confirmed · `stacks/iot/` (no `.github/workflows`)
- [ ] **M11.** `.env` files are written world-readable and non-atomically
  - Confirmed · `lib/p4n4_lib/env.py:72,90,92`; `cli/p4n4/commands/secret.py:105`
- [ ] **M12.** Projects with the same directory name share data volumes and stale credentials
  - Confirmed · `lib/p4n4_lib/layout.py:66-67`
- [ ] **M13.** Shared InfluxDB settings are never compared across layers
  - Confirmed · `lib/p4n4_lib/validate.py:84-98`
- [ ] **M14.** The API's start/stop duplicates the CLI's logic without its fixes
  - Confirmed · `api/p4n4_api/jobs.py:116-125,163-168`
- [ ] **M15.** Embedded services and Swagger UI run on the dashboard's origin, next to its stored session
  - Confirmed (config) · `dashboard/docker/nginx/embed_proxy.conf:1-3,23`; `default.conf.template:155-164`
- [ ] **M16.** `scripts/dev` exposes a dev API seeded with admin/p4n4 to the network
  - Confirmed · `scripts/dev:89,162-165,235-236`
- [ ] **M17.** The dashboard's default API upstream doesn't reach the API container on Linux
  - Confirmed (runtime Suspected) · `dashboard/docker-compose.yml:16,26-27`
- [ ] **M18.** Ollama templates publish an unauthenticated Ollama proxy with InfluxDB tools on all interfaces
  - Confirmed · `tools/templates/projects/mqtt-influx-grafana-ollama/ai/docker-compose.yml:19`; `ai/agent/agent.py:569,612`
- [ ] **M19.** ONNX runner reads only `outputs[0]`, so scikit-learn classifiers always report the first class at 100%
  - Confirmed · `stacks/ai/runner/runner.py:500-503`
- [ ] **M20.** Only the IoT stack rotates container logs
  - Confirmed · `stacks/iot/docker-compose.yml:12-16`
- [ ] **M21.** The AI and extras docs create `p4n4-net` without the Compose label, so the IoT stack later refuses to start
  - Confirmed · `stacks/ai/README.md:104,638`; `stacks/extras/README.md:70`

### Low (38)

- [ ] **L1.** The incident agent has no admission control
  - Confirmed · `stacks/ai/agent/src/request-id.ts:6,69`
- [ ] **L2.** Refresh-token rotation is check-then-set
  - Confirmed · `api/p4n4_api/auth.py:167-177`
- [ ] **L3.** The MQTT publish deny list misses topics that are stored as data
  - Confirmed · `api/p4n4_api/config.py:79`
- [ ] **L4.** `secret rotate` prints every new secret
  - Confirmed · `cli/p4n4/commands/secret.py:51,90-95`
- [ ] **L5.** With the API unreachable, anyone at the device can pick Admin and redirect the next sign-in
  - Confirmed · `dashboard/lib/pages/login_page.dart:217,266`
- [ ] **L6.** Alloy's `:ro` Docker socket mount still gives full Docker API access
  - Confirmed · `stacks/iot/docker-compose.yml:281,293`
- [ ] **L7.** Anonymous clients can grow the feedback files without limit
  - Confirmed · `stacks/iot/config/node-red/flows/flows.json:664,724-728`
- [ ] **L8.** Default binds publish more than the trusted-network stance needs
  - Confirmed · `stacks/iot/docker-compose.yml:38,85,126,183,315`
- [ ] **L9.** The ei-runner image is unhardened and built from floating inputs
  - Confirmed · `stacks/ai/runner/Dockerfile:3,14`
- [ ] **L10.** The OBD2 guard is broader than needed, and the VIN hash can be reversed
  - Confirmed · `stacks/extras/obd2/service/obd2_bridge/guard.py:29`; `bridge.py:225-226`
- [ ] **L11.** Cloud bootstrap runs remote scripts as root and builds from `main`
  - Confirmed · `tools/emu/terraform/aws/emulator/cloud-init.yaml.tftpl:14,18-19`
- [ ] **L12.** Every AWS emulator can read and overwrite every run's project and report
  - Confirmed (reachability Suspected) · `tools/emu/terraform/aws/shared/main.tf:109-114`
- [ ] **L13.** Two helper commands run mutable `:latest` images
  - Confirmed · `tools/emu/scripts/install-binfmt.sh:7`; `stacks/ai/Makefile:240`
- [ ] **L14.** Most repos have no Dependabot; lib and cli publish without tests or a tag check
  - Confirmed · `lib/.github/workflows/publish.yml:19-20`
- [ ] **L15.** The template validator and CI have gaps
  - Confirmed · `tools/templates/scripts/validate.py:128`
- [ ] **L16.** The AWS emulator's TTL is lost on reboot
  - Confirmed · `tools/emu/terraform/aws/emulator/cloud-init.yaml.tftpl:11-12`
- [ ] **L17.** Disk limits may target the client's disk instead of the engine's
  - Suspected · `tools/emu/p4n4_emu/utils/docker_info.py:16,39`; `commands/up.py:228`
- [ ] **L18.** One idle connection can wedge the runner's HTTP server
  - Confirmed · `stacks/ai/runner/runner.py:56,189,264`
- [ ] **L19.** The OBD2 simulator's memory grows without limit
  - Confirmed · `stacks/extras/obd2/simulator/elm327_sim.py:138,169-174`
- [ ] **L20.** The extras validator misses long-syntax ports
  - Confirmed · `stacks/extras/scripts/validate.py:123-126`
- [ ] **L21.** Braces in a default may break the result topic on older Compose
  - Suspected · `stacks/ai/docker-compose.yml:226`
- [ ] **L22.** The "must match p4n4-iot" comment is wrong for `MQTT_PORT`
  - Confirmed · `stacks/ai/docker-compose.yml:220-222`; `stacks/ai/.env.example:130-133`
- [ ] **L23.** `p4n4 up --no-detach` never starts the second stack
  - Confirmed · `cli/p4n4/commands/lifecycle.py:156-168`
- [ ] **L24.** The InfluxDB password prompt has no 8-character minimum
  - Suspected · `cli/p4n4/commands/init.py:328-331`
- [ ] **L25.** An interrupted `init` leaves a partial project and temp clones behind
  - Confirmed · `cli/p4n4/commands/init.py:462`; `lib/p4n4_lib/scaffold.py:99-108`
- [ ] **L26.** CORS blocks the API's PUT routes
  - Confirmed · `api/p4n4_api/main.py:116`
- [ ] **L27.** The API compose file defaults to image 0.3.0, while the package is 0.4.0
  - Confirmed · `api/docker-compose.yml:13`; `api/.env.example:9`
- [ ] **L28.** `p4n4-emu` preflight crashes without the Docker CLI
  - Confirmed · `tools/emu/p4n4_emu/utils/preflight.py:43,71`
- [ ] **L29.** Emulated CPU limits can exceed the engine's CPU count
  - Suspected · `tools/emu/p4n4_emu/overlays/generator.py:264,269`
- [ ] **L30.** The hardware button's restart skips several containers
  - Confirmed · `tools/hw/scripts/rpi5/p4n4_common.py:63-66`
- [ ] **L31.** Golden fixtures can be captured from uncommitted code
  - Confirmed · `xtask/src/golden.rs:24-25,274,280-304`
- [ ] **L32.** An empty `SITE_TOKEN` crash-loops the go2rtc template's ingest
  - Confirmed · `tools/templates/projects/mqtt-influx-grafana-ollama-go2rtc/iot/docker-compose.yml:29`
- [ ] **L33.** The templates have drifted from the core stacks
  - Confirmed · `tools/templates/projects/mqtt-influx-grafana/docker-compose.yml:66,110,177`
- [ ] **L34.** A stale committed test project uses public credentials
  - Confirmed · `cli/test-project/docker-compose.yml:42,45,69,73,101,108`
- [ ] **L35.** The security stance and guide are out of date
  - Confirmed · `README.md:11`; `docs/guides/security.md:34,37,71`
- [ ] **L36.** The CLI docs don't match the CLI
  - Confirmed · `cli/README.md:485,582,629`; `docs/reference/cli-reference.md:45`
- [ ] **L37.** The API docs misstate roles and endpoints
  - Confirmed · `api/README.md:16,31,50,68,147`
- [ ] **L38.** The template docs point to missing files and commands
  - Confirmed · `CONTRIBUTING.md:48-49`; `tools/templates/README.md:5`

### Info (9)

- [ ] **I1.** Telegraf mounts the whole host filesystem
  - Confirmed · `stacks/iot/docker-compose.yml:250`
- [ ] **I2.** The AWS emulator's root volume isn't explicitly encrypted
  - Confirmed · `tools/emu/terraform/aws/emulator/main.tf:69-73`
- [ ] **I3.** The GCP and Azure `report` outputs hard-code `~/greenhouse`
  - Confirmed · `tools/emu/terraform/gcp/emulator/outputs.tf:39`; azure `outputs.tf:50`
- [ ] **I4.** Emulator overlays outlive their containers
  - Confirmed · `tools/emu/p4n4_emu/overlays/paths.py:7-8`; `cli/p4n4/commands/lifecycle.py:202-206`
- [ ] **I5.** No test checks that the proxy strips credentials
  - Confirmed · `dashboard/.github/workflows/image.yml:76-100`
- [ ] **I6.** Android release builds are signed with the debug key
  - Confirmed · `dashboard/android/app/build.gradle.kts:35-36`
- [ ] **I7.** `jinja2` is an unused dependency
  - Confirmed · `cli/pyproject.toml:30`
- [ ] **I8.** The quick-start prerequisites and the URL table are incomplete
  - Confirmed · `README.md:102-107,186-198`
- [ ] **I9.** The dashboard docs have drifted
  - Confirmed · `docs/stacks/dashboard.md:19-25,57-70,98-99`

## 3. Detailed findings

Paths are relative to the superproject root. Each finding's confidence is given at the start of the finding, before the location. "Confirmed" means the code was read and the behavior traced. "Suspected" means the issue is plausible but wasn't verified.

### High

#### H1. go2rtc template: an unauthenticated, writable API allows code execution
Confirmed (from the code and go2rtc v1.9.14 source; not run) · `tools/templates/projects/mqtt-influx-grafana-ollama-go2rtc/extras/`

- **What's wrong:** `video.sh` writes go2rtc's config to a writable tmpfs. It then starts go2rtc with its API on `:1984` and no credentials. In go2rtc v1.9.14:
  - `POST /api/config` overwrites the config file with `os.WriteFile(app.ConfigPath, …)`.
  - `/api/restart` reloads the file.
  - Streams defined in the file are exempt from the API's `exec:` refusal.

  Anyone who reaches port 1984 can POST a config with an `exec:` stream, restart, then request `/api/frame.jpeg?src=…`. Their command then runs as root in a container on `p4n4-net`. `GET /api/config` also returns `CAMERA_URL`, including any camera password embedded in it.
- **Why it matters:** this goes beyond the trusted-LAN stance. The template's own comments and README say this path is closed, so users have no reason to firewall it. The container sits on the network with MQTT, InfluxDB and Node-RED. The service has no profile, so it starts whenever the extras layer does.
- **Evidence:**
  - `extras/docker-compose.yml:20` publishes `- "1984:1984"   # MJPEG streams, snapshots and go2rtc's web UI: site network only`.
  - `extras/docker-compose.yml:23-25` sets `read_only: true` plus `tmpfs: - /tmp`. There is no `user:` and no `cap_drop`.
  - `extras/config/go2rtc/video.sh:17-18` says "The config lives in /tmp, so the API can't rewrite it; go2rtc refuses exec: sources from its API."
  - `video.sh:22` sets `CONFIG=/tmp/go2rtc.yaml`.
  - `video.sh:59-60` writes `echo "api:"` / `echo "  listen: ':1984'"`.
  - `video.sh:70` runs `exec go2rtc -config "$CONFIG"`.
  - The template README (:180) repeats the claim: "go2rtc's API refuses `exec:` sources, and its config isn't writable…"
- **Fix:** restrict the API to the paths the dashboard needs. Bind it to localhost unless the user opts in, or set `api.username` and `api.password` and pass them to the dashboard's Video tab. Drop privileges, and correct the comment and the README.
  ```sh
  echo "api:"
  echo "  listen: ':1984'"
  echo "  allow_paths: [/api/stream.mjpeg, /api/frame.jpeg]"   # plus any path the Video tab calls
  ```
  ```yaml
  ports: ["${VIDEO_BIND:-127.0.0.1}:1984:1984"]
  cap_drop: [ALL]
  security_opt: ["no-new-privileges:true"]
  ```

#### H2. n8n cold-chain template: anyone can forge acknowledgement links and silence escalation
Confirmed · `tools/templates/projects/mqtt-influx-grafana-n8n/ai/`

- **What's wrong:** ack links are signed with HMAC-SHA256 keyed by `ACK_SECRET`.
  - `.env.example` ships a public value.
  - The compose fallback is an empty string, and the Code nodes accept an empty key.
  - The signed message is the device name plus the excursion id, which is the excursion's start time as an ISO timestamp, so it is guessable.
  - Tokens are compared with `!==`.

  The `coldchain/ack` (GET and POST), `coldchain/check` and `coldchain/report` webhooks have no authentication, and n8n is published on all interfaces. So anyone who can reach port 5678 can acknowledge an open excursion, which stops its escalation.
- **Why it matters:** the README says "Unsigned or forged links are refused" (:126) and expects phones to open the links from outside the LAN (:223). A forged ack suppresses alerts about spoiling stock, which is the template's core safety function. The default `COMPOSE_PROFILES=mailpit` (`ai/.env.example:13`) also exposes the real links in Mailpit on :8025.
- **Evidence:**
  - `ai/.env.example:51` has `ACK_SECRET=change-me-ack-secret`.
  - `ai/docker-compose.yml:54` has `ACK_SECRET: ${ACK_SECRET:-}`, and `:18` publishes `- "5678:5678"`.
  - `ai/config/n8n/workflows/acknowledge.json:129` signs with `crypto.createHmac('sha256', setting('ACK_SECRET', ''))` and checks `b.token !== sign(b.device, b.excursion)`.
  - The same signer appears at `acknowledge.json:74`, `monitor.json:172` and `report.json:64,185`.
  - `monitor.json:172` creates the id as `ex = { id: new Date(start).toISOString(), …}`.
  - `acknowledge.json:255` shows that an ack ends escalation.
  - The template's `entrypoint.sh:17` already refuses an empty owner password, so the same pattern is available.
- **Fix:** have `entrypoint.sh` exit when `ACK_SECRET` is empty, equal to the example value, or shorter than 32 characters. Ship it blank, with an `openssl rand -hex 32` hint. In the Code nodes:
  - throw on an empty key;
  - compare with `crypto.timingSafeEqual`;
  - put an expiry in the signed message;
  - accept acks only for excursions that are still open.

  Publish n8n on `${N8N_BIND:-127.0.0.1}`, or put header auth on the webhooks.

#### H3. Incident agent: one anonymous MQTT message crash-loops it permanently
Confirmed (reproduced `parseAlert` under node; the MQTT.js stall side effect is Suspected) · `stacks/ai/agent/src/`

- **What's wrong:**
  - Alert timestamps are only checked against a lower bound.
  - A huge nanosecond value passes the check. It becomes a valid window key, `pump1:+275760-09-13T00:00:00.000Z`, but later `new Date(ts).toISOString()` throws `RangeError`.
  - The `alert()` call sits outside the handler's try/catch, and the message is never acknowledged, so the broker redelivers it.
  - `recover()` re-runs open items at every start and throws again, so `process.exit(1)` plus `restart: unless-stopped` loops forever. Recovery requires deleting the agent's data store.
- **Why it matters:** MQTT is anonymous by default (M3), so any LAN device can do this with `{"timestamp":8640000000000500000000}` on `alerts/<device>/anomaly`. This contradicts `docs/reference/agent.md:30, :120, :138-140`. The agent profile is opt-in (`agent.md:7`).
- **Evidence:**
  - `request-id.ts:39` returns `Math.round(v / 1e6)`.
  - `request-id.ts:69` only checks `if (!Number.isFinite(timestamp) || timestamp < MIN_TIME) timestamp = receivedAt;`.
  - `investigator.ts:50` calls `const at = new Date(alert.timestamp).toISOString();`.
  - `investigator.ts:114` restarts open items with no submission id: `if (known.status === "open" && known.submissionId === null) await this.start(key);`.
  - `investigator.ts:270` builds `alert_at: new Date(item.alertAt).toISOString(),`.
  - `investigator.ts:304` runs `await this.serial(() => this.start(key));` inside `recover()`.
  - `main.ts:93` (`const { created } = await investigator.alert(parsed);`) sits outside the try/catch.
  - `mqtt.ts:24-25` says "Not acknowledged: the broker delivers it again on the next connection".
  - `main.ts:135-136` then calls `process.exit(1)`, and the compose service is `restart: unless-stopped` (`stacks/ai/docker-compose.yml:261`).
  - The only timestamp test checks the lower bound (`test/request-id.test.ts:41-45`).
- **Fix:**
  ```ts
  // request-id.ts:69
  if (!Number.isFinite(timestamp) || timestamp < MIN_TIME || timestamp > receivedAt + 86_400_000) timestamp = receivedAt;
  ```
  Also:
  - Add an `iso(ms)` helper that returns `"unknown time"` when `Math.abs(ms) > 8.64e15`, and use it at `investigator.ts:50` and `:270`.
  - Wrap each item in `recover()` in try/catch and mark failures as failed.
  - Acknowledge (`done()`) messages whose errors redelivery can't fix.
  - Add tests for the upper bound and for a poisoned store.

### Medium

#### M1. n8n is published on 0.0.0.0, the first visitor becomes owner, and the password `p4n4 init` asks for does nothing
Confirmed (the n8n profile is opt-in) · `stacks/ai/docker-compose.yml:83-87`, `cli/p4n4/commands/init.py:357,417`

- **What's wrong:** every other unauthenticated AI service binds 127.0.0.1, but n8n publishes `- "5678:5678"` (:83). The `N8N_BASIC_AUTH_*` variables at :85-87 (`${N8N_BASIC_AUTH_PASSWORD:-adminpassword}`) have been ignored since n8n 1.0. The project says so itself in `lib/p4n4_lib/secrets.py:39` ("Unused: n8n dropped basic auth in 1.0") and in `docs/reference/configuration.md:168`. `stacks/ai/README.md:619` adds that "n8n asks whoever opens it first to create the owner account". Yet `p4n4 init` still prompts for an "n8n admin password, used if you enable n8n…" (:357) and writes it as `"N8N_BASIC_AUTH_PASSWORD": n8n_password` (:417).
- **Why it matters:** `docs/guides/security.md:32` promises "The services without authentication are published on `127.0.0.1` only". In practice, whoever reaches :5678 first after the profile is enabled owns n8n. Its Code nodes can then reach the internal anonymous MQTT listener, Ollama, System 1 and InfluxDB, plus the credentials that `cli/p4n4/commands/ai.py:169-170` imports. Users who answered the prompt believe n8n is protected.
- **Fix:** publish `"${N8N_BIND:-127.0.0.1}:5678:5678"` and delete the `N8N_BASIC_AUTH_*` variables. Either create the owner from env as the n8n template already does (its `ai/docker-compose.yml:31-32` sets `N8N_INSTANCE_OWNER_MANAGED_BY_ENV: "true"`, with a bcrypt hash in `entrypoint.sh:18-21`), or remove the prompt. The dashboard reaches `p4n4-n8n` over `p4n4-net`, so a loopback bind doesn't affect it.

#### M2. Compose fallbacks bring back the public example secrets whenever `.env` is missing or incomplete
Confirmed · `stacks/iot/docker-compose.yml`, `stacks/ai/docker-compose.yml`, `stacks/iot/Makefile:80-82`

- **What's wrong:**
  - In `stacks/iot/docker-compose.yml`:
    - :89 `DOCKER_INFLUXDB_INIT_PASSWORD: ${INFLUXDB_PASSWORD:-adminpassword}`
    - :93 `DOCKER_INFLUXDB_INIT_ADMIN_TOKEN: ${INFLUXDB_TOKEN:-p4n4-stack-token}`, repeated at :131, :194, :239 and in `scripts/init-buckets.sh:33`
    - :186 `GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD:-adminpassword}`
  - In `stacks/ai/docker-compose.yml`:
    - :56 and :292 fall back to `lettapassword`
    - :88 falls back to `${N8N_ENCRYPTION_KEY:-change-me-32-char-encryption-key}`
    - :105, :229 and :302 fall back to the public token
  - `make up` (`stacks/iot/Makefile:80-82`) never checks that `.env` exists.
  - The `.env.example` files ship the same values (`stacks/iot/.env.example:98,100,121,136`; `stacks/ai/.env.example:42,53,184`).
  - The templates show the same pattern, plus a read check that fails open:
    - `tools/templates/projects/mqtt-influx-grafana-ollama-go2rtc/iot/.env.example:58,65` ship `SITE_TOKEN=demo-site-token` and `INGEST_READ_TOKEN=demo-read-token`.
    - That template's `iot/ingest/ingest.py:163-164` skips the token check when the read token is empty (`if not self.read_token:` / `return True, None`). It does this for plate lookups, which are personal data, on the port the template's README:179 says to put behind an internet-facing HTTPS proxy.
    - Pidurable's `ai/.env.example:31` placeholder `AGENT_API_TOKEN=change-me-to-32-or-more-random-characters` passes the agent's 32-character check (`stacks/ai/agent/src/config.ts:62`), while the core ships the token empty.
- **Why it matters:** the InfluxDB token is an all-access operator token, and InfluxDB is published on `0.0.0.0:8086` (iot compose:85). These values are applied only on the first start (setup mode, iot compose:87), so a single start without `.env` locks the public credentials into the volume. `README.md:151` warns (`# replace every password and token: the examples are public`), and `p4n4 init` generates everything, but the manual and `make` paths fail open.
- **Fix:**
  - Fall back to empty (`${INFLUXDB_TOKEN:-}`) so the services refuse to start.
  - Blank the secrets in `.env.example`.
  - Add a Makefile guard that requires `.env` and rejects the known public values.
  - Avoid `${VAR:?}`: Compose interpolates services in disabled profiles too, so it would fail when an unrelated layer's variable is unset.
  - In the templates, make the ingest refuse reads when the read token is empty, and refuse `demo-*` and short tokens unless a demo flag is set.

#### M3. MQTT accepts anonymous clients by default, including after `p4n4 init`
Confirmed · `stacks/iot/config/mosquitto/bridge.sh:118-128,147,179`, `stacks/iot/docker-compose.yml:36,44-45`, `cli/p4n4/commands/init.py:399`

- **What's wrong:**
  - The core broker supports device auth, but it is off unless `MQTT_PASSWORD` is set. `bridge.sh:126` falls through to `echo "allow_anonymous true"`, and the internal listener (1883) is always anonymous (:147).
  - The device listener is published on `"${MQTT_BIND:-0.0.0.0}:${MQTT_PORT:-1883}:1884"` (iot compose:36).
  - `p4n4 init` writes `NODE_RED_PASSWORD` (init.py:399) but never `MQTT_USER` or `MQTT_PASSWORD`, so the startup log line at `bridge.sh:179` ("devices ($listeners) connect anonymously (MQTT_PASSWORD is empty)") is what every initialized project gets.
  - Seven template `mosquitto.conf` files hard-code `allow_anonymous true` with no option to change it (for example `tools/templates/projects/mqtt-influx-grafana/config/mosquitto/mosquitto.conf:21-22`). The go2rtc README (:182) tells users to set up broker auth, but its ingest is never passed MQTT credentials.
- **Why it matters:** several consumers act on whatever any publisher sends:
  - the agent (H3);
  - n8n's `incident-escalation.json:7` (`alerts/+/critical`);
  - Node-RED's feedback file writes (`stacks/iot/config/node-red/flows.json:664`);
  - the greenhouse template's `actuators/+/+/mode` valve control (`mqtt-nodered-influx-grafana/…/flows.json:174`). A forced-on valve "is left alone" (:240), so the template's max-run safety doesn't apply.

  That's acceptable on a trusted LAN, but `p4n4 init` is the "secure" path, and it leaves this open with no way to opt in.
- **Fix:**
  - Have `init` generate `MQTT_USER` and `MQTT_PASSWORD`, plus a read-only ACL for dashboards, and pass them to Telegraf, Node-RED, n8n and the agent.
  - Port the core's opt-in auth to the templates.
  - Apply `VALVE_MAX_RUN` to forced-on valves too.

#### M4. The sign-in rate limit trusts X-Forwarded-For from all of 172.16.0.0/12
Confirmed · `api/docker-compose.yml:34`, `api/p4n4_api/main.py:101-107`, `dashboard/docker/nginx/proxy_common.conf:4`

- **What's wrong:**
  - The API container's default is `P4N4_API_TRUSTED_PROXIES: ${P4N4_API_TRUSTED_PROXIES:-172.16.0.0/12}`. That range covers every default Docker bridge network, `p4n4-net` (172.20.0.0/16) included, and many corporate LANs.
  - `ProxyHeadersMiddleware` takes the rightmost untrusted hop (main.py:103-107), and the limiter keys on that address (`auth.py:300` `login_limiter.hit(request.client.host …)`).
  - Any container on `p4n4-net` (n8n's Code node, Node-RED, a template service) can call `p4n4-api:8000` directly with a fresh `X-Forwarded-For` per attempt. A browser client whose address nginx sees inside 172.16/12 can do the same, because nginx appends the client's header instead of replacing it (`proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`).
  - `tests/test_auth.py:187-193` shows that a new forwarded address gets a fresh budget.
- **Why it matters:** this allows unlimited password guessing against the admin account. `users.py:30` uses argon2's defaults (`PasswordHasher()`, 64 MiB per hash), so many concurrent attempts also exhaust memory on small edge hosts: about 2.5 GiB at anyio's 40 worker threads. That figure is computed, not measured.
- **Fix:**
  - Trust only the dashboard: give it a fixed `ipv4_address` on `p4n4-net` and default `P4N4_API_TRUSTED_PROXIES` to that /32.
  - In nginx (the edge proxy), use `proxy_set_header X-Forwarded-For $remote_addr;`.
  - Add a per-username failure limit, and cap concurrent argon2 verifications with a small semaphore.

#### M5. Open SSE streams outlive sign-out, password changes and user removal
Confirmed · `api/p4n4_api/routes/telemetry.py:182-193`, `api/p4n4_api/routes/control.py:179-193`

- **What's wrong:** the session check (`auth.py:_session_live`, used by `current_user`) runs once, when the request starts. After that:
  - `stream_events` loops `while True` on readings and keep-alives (telemetry.py:182-187).
  - `follow_events` loops on `docker compose logs -f` (control.py:179-184).

  The keep-alive comments also keep nginx's read timeout from closing either stream.
- **Why it matters:** `docs/reference/api.md:217-219` promises that signing out, changing a password or removing the user ends access tokens "at once rather than when they expire". A removed admin, or a stolen access token, keeps a live view of container logs, which the API itself says "can contain secrets" (control.py). The token also keeps live telemetry flowing indefinitely.
- **Fix:** re-check `_session_live` (and the user's role) on every heartbeat in both generators, and end the stream with `event: end` when it fails. Also cap the stream's lifetime at the access token's `exp`.

#### M6. The API image resolves dependencies at build time and ignores `uv.lock`
Confirmed · `api/Dockerfile:20-25`, `api/.github/workflows/ci.yml:36`

- **What's wrong:** the image runs `ARG P4N4_LIB="p4n4-lib>=0.4.0,<0.5"`, then `pip install "$P4N4_LIB"`, then `pip install .`. Every transitive version is whatever PyPI serves at build time, with no hashes, even though the repo has a `uv.lock`. CI tests a third combination: `pip install "p4n4-lib @ git+https://github.com/raisga/p4n4-lib.git"` (lib `main`).
- **Why it matters:** the image published to GHCR is not the dependency set that was tested or locked. A compromised or broken release of any transitive dependency ships straight into the image. The Trivy gate only fails on `severity: CRITICAL` (`image.yml:113`).
- **Fix:** export the lock and install with hashes, for example `uv export --frozen --no-dev --format requirements-txt > requirements.txt`, then `pip install --require-hashes -r requirements.txt` and `pip install --no-deps .`. Run CI against the same lock, and test lib `main` in a separate, non-blocking job.

#### M7. `p4n4 init` copies four repositories from unpinned `main`
Confirmed · `lib/p4n4_lib/scaffold.py:103`, `lib/p4n4_lib/sources.yaml:1-4`, `cli/p4n4/commands/lifecycle.py:24`

- **What's wrong:**
  - `subprocess.run(["git", "clone", "--depth", "1", "--config", "core.autocrlf=false", layer.repo_url, tmp])` takes no `--branch`. The URLs in `sources.yaml` (`p4n4-iot`, `p4n4-ai`, `p4n4-extras`, `p4n4-dashboard`) carry no ref, so a released CLI scaffolds whatever is on `main` that day.
  - The cloned scripts are then made executable (scaffold.py:141-144).
  - `env.write` copies every `.env.example` line it has no value for verbatim (`lib/p4n4_lib/env.py:89` `out.append(line)`). A secret added upstream after the CLI release therefore ships with its public placeholder.
  - The `--emu` hint installs `git+https://github.com/raisga/p4n4-emu` (lifecycle.py:24), also unpinned.
- **Why it matters:** this is the main supply-chain gap. `p4n4-iot` has no CI at all (M10), so an untested or compromised commit lands on every new install. Two installs of the same CLI version also aren't reproducible.
- **Fix:** add a `ref` per source in `sources.yaml` (a tag, or ideally a commit SHA that the release workflow bumps) and pass `--branch <tag>` (or fetch the SHA). Warn when an `.env.example` key looks secret but has no generated value. Pin the emu install to a tag.

#### M8. `p4n4-lib` has no upper bound, and that has already broken a published CLI
Confirmed · `cli/pyproject.toml:26`, `api/pyproject.toml:25`, `tools/emu/pyproject.toml:31`

- **What's wrong:** the three consumers declare `"p4n4-lib>=0.4.2",` and `"p4n4-lib>=0.4.0",`, while lib treats 0.x minor releases as breaking. The cli tag v0.3.0 declares `"p4n4-lib>=0.3.0"` and its `init` uses the `edge` layer. lib 0.4.0 then shipped "Removed: The `edge` layer from `LAYERS`" (`lib/CHANGELOG.md:76-78`), so installing `p4n4==0.3.0` today pulls lib 0.4.x and breaks `init`. The API image does pin `<0.5` (`api/Dockerfile:20`), but its pyproject doesn't.
- **Fix:** declare `p4n4-lib>=0.4.2,<0.5` (or `~=0.4.2`) in all three. Publish a 0.3.x post-release of cli with `p4n4-lib<0.4`, or yank 0.3.0.

#### M9. Release jobs hand write tokens to third-party actions that are pinned by tag
Confirmed · all 21 workflows; for example `api/.github/workflows/image.yml:27-31,110`

- **What's wrong:**
  - No workflow pins any action to a commit SHA.
  - Eight jobs hold write scopes:
    - `image.yml` in api, dashboard and tools/emu (`packages: write`);
    - `publish.yml` in lib, cli and tools/emu (`id-token: write`, with `pypa/gh-action-pypi-publish@release/v1`, a branch ref);
    - `agent.yml` and `system1.yml` in stacks/ai.
  - In api's `image.yml`, `permissions: contents: read / packages: write` is followed by `- uses: actions/checkout@v7`. Checkout's default `persist-credentials` leaves the token in `.git/config` for later steps, such as `uses: aquasecurity/trivy-action@v0.36.0` (:110) and the `docker/*@v4…v7` actions.
  - Eleven workflows have no `permissions:` block and inherit the repository default: root `submodules.yml`; `ci.yml` in api, cli, dashboard, lib, stacks/ai, stacks/extras, tools/emu, tools/hw and tools/templates; and `tools/emu/.github/workflows/terraform.yml`.
- **Why it matters:** if any of those tags is moved, an attacker can push a malicious `ghcr.io/raisga/p4n4-*` image or PyPI release, and users' compose files and `pip install p4n4` pick it up by tag.
- **Fix:**
  - Pin every `uses:` to a full SHA, with a version comment, and let Dependabot's `github-actions` ecosystem bump them.
  - Set `persist-credentials: false`.
  - Add top-level `permissions: contents: read` to every workflow.
  - Give `packages: write` or `id-token: write` only to a small push/publish job that runs no third-party scanners.

#### M10. The core IoT stack has no CI at all
Confirmed · `stacks/iot/` (no `.github/workflows`), `CONTRIBUTING.md:36-38`

- **What's wrong:** CONTRIBUTING requires that compose stacks "Run `docker compose config --quiet` before pushing" and "Keep healthchecks on every service". stacks/ai and stacks/extras have `ci.yml`, but stacks/iot has no workflow. That leaves the broker auth script (`bridge.sh`), `init-buckets.sh`, the Node-RED flows and the Grafana provisioning untested, while `p4n4 init` clones this repo's `main` (M7).
- **Fix:** add a `ci.yml` modeled on stacks/extras':
  - `docker compose --profile "*" config --quiet` with `.env.example`;
  - `shellcheck config/mosquitto/*.sh scripts/*.sh`;
  - a JSON parse of the flows and dashboards;
  - a `docker compose up --wait` smoke test of the MING profile.

#### M11. `.env` files holding every secret are written world-readable and non-atomically
Confirmed · `lib/p4n4_lib/env.py:72,90,92`, `cli/p4n4/commands/secret.py:105`

- **What's wrong:** every write is `path.write_text(...)`, so the mode follows the umask (normally 0644). Nothing in lib or cli calls `chmod`, sets a umask or uses `os.replace` for `.env`. `secret rotate` calls `envutil.set_value(path, key, value)` once per key, and each call rewrites the whole file.
- **Why it matters:** `.env` holds the all-access InfluxDB token, the Grafana, Node-RED and Letta passwords, and the n8n encryption key. Any local user or process on a shared host can read them. A crash between the per-key rewrites leaves a half-rotated file.
- **Fix:**
  - Write to a temp file in the same directory, opened with `os.open(tmp, os.O_WRONLY | os.O_CREAT | os.O_EXCL, 0o600)`; `fsync` it, then `os.replace` it over `.env`.
  - Apply all rotated keys in one write.
  - Have `p4n4 validate` warn when `.env` is group- or world-readable.

#### M12. Projects with the same directory name silently share data volumes
Confirmed · `lib/p4n4_lib/layout.py:66-67`, `cli/p4n4/commands/init.py:127`

- **What's wrong:** the Compose project name comes only from the directory's basename: `raw = project if len(layers) <= 1 else f"{project}-{name}"`, called with `project_dir.name`. Named volumes (`influxdb-data`, `grafana-data`, … at iot compose:335-342) are scoped by that name. `init` never checks for existing volumes; the only `docker volume` mention in lib or cli is in `migrate.py`.
- **Why it matters:** `p4n4 init demo` in another directory, or after deleting an old `demo` without `down -v`, attaches the old volumes. InfluxDB and Grafana keep their first credentials (setup mode, iot compose:87; see the `SETUP_KEYS` comment in `lib/p4n4_lib/secrets.py:28-35`), so the freshly generated `.env` doesn't match:
  - Telegraf, Node-RED and Grafana get auth failures;
  - the old data and the old admin password stay live.
- **Fix:** in `init`, run `docker volume ls --filter label=com.docker.compose.project=<name>`. If volumes exist, refuse, or derive a unique name (for example, append a short hash of the absolute path). Alternatively, offer to reuse them, with a warning that the old credentials still apply.

#### M13. Shared InfluxDB settings are never compared across layers
Confirmed · `lib/p4n4_lib/validate.py:84-98`, `README.md:180`, `docs/reference/cli-reference.md:66-68`

- **What's wrong:** `README.md:180` says "`INFLUXDB_TOKEN`, `INFLUXDB_ORG`, and `INFLUXDB_BUCKET` must be identical across all stack `.env` files". `validate` only checks that keys are present in each layer (`if (key not in env if from_template else not env.get(key))`). The documented way to add a layer, "copy the layer's directory from a fresh `p4n4 init` and add it to `layers` in `.p4n4.json`" (cli-reference.md:66-68), brings in a different, freshly generated token. In a flat single-layer project, `layout.compose_dirs` returns only the root when a root compose file exists (`layout.py:76-78`), so the added layer is never started. The n8n digest also hard-codes the org and bucket (`stacks/ai/config/n8n/workflows/scheduled-digest.json:29` `"value": "ming"`, `:33` `"value": "raw_telemetry"`).
- **Why it matters:** a mismatch fails silently. AI workflows and the agent get 401s or empty queries, and nothing in `p4n4 validate` or `p4n4 up` says why.
- **Fix:**
  - `validate` should compare the shared keys (`INFLUXDB_TOKEN`, `INFLUXDB_ORG`, `INFLUXDB_BUCKET`, `SYSTEM1_ALERT_TOKEN`) across every layer's `.env` and fail on a mismatch, and `up` should run it.
  - Implement `p4n4 add`, or have it copy the shared keys.
  - The digest should read `$env.INFLUXDB_ORG` and `$env.INFLUXDB_BUCKET`.

#### M14. The API's start and stop logic duplicates the CLI's without its fixes
Confirmed · `api/p4n4_api/jobs.py:116-125,163-168`, `lib/p4n4_lib/compose.py:82-86`, `cli/p4n4/commands/lifecycle.py:163-167`

- **What's wrong:**
  - The API builds its own commands. `if action == "down": return [["down"]]`, while lib's `compose.down` uses `["--profile", "*", "down"]` because plain `down` "only stops the ones in active profiles" (compose.py:82-86).
  - The API's `up` never calls `compose.ensure_network`, which the CLI calls (lifecycle.py:163-167) to create `p4n4-net` with Compose's label.
  - `break  # later stacks may depend on this one` (jobs.py:167-168) also applies to `down`, so one failing stack leaves the others running. The CLI's `_down_all` collects failures and continues.
- **Why it matters:** the dashboard's Stop button leaves profiled services running, which also keeps `p4n4-net` in use. Its Start button fails on a fresh host for ai or extras where `p4n4 up` succeeds. The two front ends drift further apart with every fix.
- **Fix:** move command construction into lib (for example, `compose.commands(action, …)`, plus `ensure_network` inside `compose.up`) and call it from both. For `down`, keep going after a failure and report every stack that failed.

#### M15. Embedded services and Swagger UI run on the dashboard's origin, next to its stored session
Confirmed (config); the token exposure relies on flutter_secure_storage's documented web behavior · `dashboard/docker/nginx/embed_proxy.conf:1-3,23`, `default.conf.template:155-164`

- **What's wrong:**
  - `embed_proxy.conf` serves `/grafana/`, `/svc/<id>/` (Node-RED, n8n) and Swagger UI with "their own pages, headers and scripts, not the dashboard's CSP". Its only policy is `add_header Content-Security-Policy "frame-ancestors 'self'" always;`.
  - `/swagger-ui` and `/openapi.json` are proxied to the API (`docs_url="/swagger-ui"` at `api/p4n4_api/main.py:93`). FastAPI's default page loads `swagger-ui-dist@5` from cdn.jsdelivr.net with no SRI.
  - The web app keeps its session in `PlatformSecretStore()` (`dashboard/lib/core/settings.dart:97`). On web, flutter_secure_storage stores both its AES key and the ciphertext in that origin's localStorage.
- **Why it matters:** the dashboard's strict CSP (`security_headers.conf:13`, `script-src 'self' 'wasm-unsafe-eval'`) protects only its own pages. An XSS in Grafana, Node-RED, n8n or Swagger UI, or a bad jsdelivr asset, runs on the same origin. It can read the refresh token and act as the signed-in admin against the API.
- **Fix:**
  - Serve the embedded tools from a separate origin, such as a second port or subdomain.
  - Self-host the Swagger UI assets (a custom docs route with local static files), or don't proxy them through the dashboard.
  - On web, keep the refresh token in an HttpOnly, `SameSite=Strict` cookie scoped to `/api/v1/auth`.

#### M16. `scripts/dev` exposes a dev API seeded with admin/p4n4 to the whole network
Confirmed · `scripts/dev:89,162-165,235-236`, `api/p4n4_api/users.py:24-27`

- **What's wrong:**
  - `export P4N4_API_DEV_USERS=${P4N4_API_DEV_USERS:-true}` seeds `DEV_PASSWORD = "p4n4"` for `("admin", "admin")`. users.py:24 itself says these accounts must "never" be used "in production".
  - `flutter run -d web-server --web-hostname 0.0.0.0 … --dart-define=P4N4_DEV_PROXY=true --dart-define=P4N4_DEV_USERS=true` publishes the app on every interface. :162 notes "the dev server proxies /api/ and /health to localhost:8000".
  - :235-236 copies `.env.example` to `.env`, so the stacks start with the public secrets.
- **Why it matters:** anyone on the developer's network (an office or a café) can open the web port and sign in as admin/p4n4. Through the proxy they can then start and stop stacks or read logs on the developer's machine.
- **Fix:** use `--web-hostname 127.0.0.1` by default, with an explicit `P4N4_DEV_HOST=0.0.0.0` opt-in. When the host isn't loopback, generate a random dev password per run and print it.

#### M17. The dashboard's default API upstream doesn't reach the API container on Linux
Confirmed (config); Suspected (runtime 502 on Linux, not executed) · `dashboard/docker-compose.yml:16,26-27`, `dashboard/.env.example:17-20`

- **What's wrong:** the default is `P4N4_API_UPSTREAM: ${P4N4_API_UPSTREAM:-http://host.docker.internal:8000}` with `"host.docker.internal:host-gateway"   # p4n4-api v0.1 runs on the host`. But the API now ships as a container (`api/docker-compose.yml:14` `container_name: p4n4-api`, on `p4n4-net` at :44). It publishes only on `"${P4N4_API_BIND:-127.0.0.1}:…:8000"` (:17). On Linux, `host-gateway` is the bridge's gateway address, and a port published on 127.0.0.1 isn't reachable through it. `p4n4 init` copies the dashboard's `.env.example` unchanged (`init.py:454-456`).
- **Why it matters:** on Linux, the main target for edge devices, the documented `p4n4 init --layer all` followed by `p4n4 up` gives a dashboard whose `/api/` calls fail. Docker Desktop hides the problem.
- **Fix:** default to `P4N4_API_UPSTREAM=http://p4n4-api:8000`, since both containers are on `p4n4-net`. Drop the `host-gateway` entry and update the comments.

#### M18. The Ollama templates publish an unauthenticated Ollama proxy with InfluxDB tools on all interfaces
Confirmed · `tools/templates/projects/mqtt-influx-grafana-ollama/ai/docker-compose.yml:19`, `ai/agent/agent.py:534-538,569,612` (the `-ollama-go2rtc` template has the same code)

- **What's wrong:**
  - The service publishes `- "11434:11434"`, and `ThreadingHTTPServer(("0.0.0.0", PORT), Handler)` accepts anyone.
  - `do_GET = do_DELETE = do_HEAD = _forward`, and every POST other than `/api/chat` goes to Ollama unchanged.
  - `/api/chat` runs tool calls against InfluxDB with the server's token.
- **Why it matters:** the core stack deliberately keeps Ollama on 127.0.0.1. Here, anyone on the LAN can:
  - delete models;
  - `POST /api/pull` models until the disk fills;
  - tie up the device's CPU or GPU;
  - read telemetry through the agent's tools.
- **Fix:** publish `"${AGENT_BIND:-127.0.0.1}:11434:11434"` and have the dashboard reach the agent over `p4n4-net`. Forward only an allowlist (`/api/chat`, `/api/tags`, `/api/version`, `/api/show`), with an optional bearer token.

#### M19. The ONNX runner reads only the first output, so scikit-learn classifiers always report the first class at 100%
Confirmed (traced in code; a sub-review also ran the post-processing in memory) · `stacks/ai/runner/runner.py:500-503,456-482`

- **What's wrong:** the runner does `outputs = _onnx_session.run(None, …)` and then calls `_postprocess_onnx_output(outputs[0])`. skl2onnx classifiers return `[label, probabilities]`, so `outputs[0]` is the int64 label tensor. That gives a one-element score list, which either already sums to 1 or is softmaxed to 1.0. The result is always `ONNX_LABELS[0]` (or `class_0`) with confidence 1.0, whatever the input. The README documents this exact export (`stacks/ai/README.md:293-294`, `from skl2onnx import to_onnx`) and claims ZipMap support (:304). The tests use a mocked session (`tests/test_runner.py:145-151`), so this path is never exercised.
- **Why it matters:** the documented path silently returns wrong inference results, and those results feed alerts and dashboards.
- **Fix:** choose the output by shape. Prefer a ZipMap (a list of dicts) or a float tensor with a class axis, and fall back to `outputs[0]`. Add a test with a real tiny skl2onnx model.

#### M20. Only the IoT stack rotates container logs
Confirmed · `stacks/iot/docker-compose.yml:12-16`

- **What's wrong:** iot defines `x-logging: &logging` with `max-size: ${LOG_MAX_SIZE:-10m}` and uses it on every service. The stacks/ai and stacks/extras compose files have no `logging:` key, and none of the 13 template compose files sets `max-size`.
- **Why it matters:** Docker's default json-file driver grows without limit. Ollama, Letta, n8n, the runner and the agents run for months on SD cards and small disks.
- **Fix:** copy the `x-logging` anchor into the ai, extras and template compose files.

#### M21. The AI and extras docs create `p4n4-net` without Compose's label, so the IoT stack later refuses to start
Confirmed · `stacks/ai/README.md:104,638`, `stacks/ai/Makefile:86`, `stacks/extras/README.md:70`, `stacks/extras/Makefile:52`, `stacks/extras/docker-compose.yml:22`

- **What's wrong:** the root README gets this right (`README.md:169-175`, `--label com.docker.compose.network=p4n4-net`). It explains that without the label, "Compose 2.19.1 to 5.3.1 refuse to start the IoT stack on it later". All six places listed above say plain `docker network create p4n4-net`.
- **Why it matters:** if you start the AI or extras stack first, as their READMEs describe, the IoT stack later fails with "incorrect label". The CLI repairs this (`lifecycle.py:163-166`), but the manual path doesn't.
- **Fix:** replace those six commands with the root README's command, or add a `make network` target to each stack.

### Low

Each item gives the evidence and the fix.

#### L1. The incident agent has no admission control
Confirmed · `stacks/ai/agent/src/request-id.ts:6,69`, `investigator.ts:118-154`, `docs.ts:74-77`

- **Problem:**
  - Every new `<device>:<window>` key from the anonymous broker opens an LLM investigation.
  - Nothing caps how many are open, and docs.ts:74 says "Unfinished ones always stay."
  - Ollama serves one request at a time (`OLLAMA_NUM_PARALLEL:-1`, ai compose:28), so a burst of fake alerts starves real ones until their 900 s deadlines run out.
- **Fix:**
  - Bound timestamps, as in H3.
  - Add an `AGENT_MAX_OPEN` cap.
  - Rate-limit new incidents with a token bucket.

#### L2. Refresh-token rotation is check-then-set
Confirmed (from the code; a sub-review also reproduced the race against SQLite in memory) · `api/p4n4_api/auth.py:167-177`, `api/p4n4_api/routes/auth.py:81-86`

- **Problem:**
  - `rotate()` reads `row["used"]`, then runs `UPDATE refresh_tokens SET used = 1 WHERE jti = ?` with no condition.
  - The route is a plain `def`, so FastAPI runs concurrent calls on a threadpool.
  - Two refreshes with the same stolen token can therefore both succeed, and reuse detection never fires.
- **Fix:** run `UPDATE … SET used = 1 WHERE jti = ? AND used = 0`, and treat `rowcount == 0` as reuse.

#### L3. The API's MQTT publish deny list misses topics that are stored as data
Confirmed · `api/p4n4_api/config.py:79`, `stacks/iot/config/node-red/flows/flows.json:325,342,654,664`

- **Problem:**
  - The default deny list is `("sensors/#", "inference/#")`.
  - Node-RED also stores `system1/decisions/#` and `system1/feedback/#` in InfluxDB, and appends the feedback to a file.
  - So even with MQTT auth on, an operator can forge System 1 decisions and training feedback through `POST /api/v1/mqtt/publish`.
- **Fix:** add `system1/#` to the default deny list.

#### L4. `secret rotate` prints every new secret
Confirmed · `cli/p4n4/commands/secret.py:51,90-95`

- **Problem:**
  - Before the confirmation prompt, a table with a "New value" column goes to the terminal. That happens even if the user declines.
  - `secret show` masks with `val[:4] + "*" * …`, which reveals each value's first four characters and its length.
- **Fix:** list key names only, with an opt-in `--show-values`, and use a fixed-width mask.

#### L5. When the API is unreachable, anyone at the device can pick Admin and redirect the next sign-in
Confirmed (from the code) · `dashboard/lib/pages/login_page.dart:217,266`, `dashboard/lib/pages/settings_page.dart:1557`, `dashboard/lib/api/auth.dart:68-70`

- **Problem:**
  - Any probe result other than 401 counts as "unreachable", including nginx's 502 while the API restarts.
  - The unreachable screen offers an offline role picker that includes `session.signInLocal(Role.admin)`.
  - That local session can change the saved API address. The next user's password is then sent to that address; the CSP allows `connect-src 'self' http: https:`.
- **Fix:**
  - When the API is unreachable, offer only the read-only view.
  - Block Connection changes in local sessions.
  - Warn on the login form when the API address differs from the default.

#### L6. Alloy's `:ro` Docker socket mount still gives full Docker API access
Confirmed · `stacks/iot/docker-compose.yml:281,293`, `stacks/iot/README.md:330`

- **Problem:**
  - The `:ro` on `/var/run/docker.sock:/var/run/docker.sock:ro` makes only the socket file read-only; the Docker API stays fully writable.
  - Anything that compromises Alloy can create a privileged container, which means root on the host.
  - The README describes only information disclosure.
- **Fix:** put a socket proxy in front that allows only GET on containers and logs, and correct the wording.

#### L7. Anonymous clients can grow the feedback files without limit
Confirmed · `stacks/iot/config/node-red/flows/flows.json:664,724-728`

- **Problem:**
  - The `system1/feedback/#` flow appends every message to a `.jsonl` file (`appendNewline`, `createDir`) with no size cap.
  - No Mosquitto config sets `max_packet_size`.
  - With the anonymous broker (M3), any LAN client can fill the disk.
- **Fix:** set `max_packet_size` in the config bridge.sh generates, and validate and cap or rotate the files in the flow.

#### L8. Default binds publish more than the trusted-network stance needs
Confirmed · `stacks/iot/docker-compose.yml:38,85,126,183,315`, `dashboard/docker-compose.yml:14`, templates (e.g. `tools/templates/projects/mqtt-influx-grafana/docker-compose.yml:59`)

- **Problem:**
  - MQTT TLS (8883), InfluxDB (8086), Node-RED, Grafana, mqtt-ui (8081) and the dashboard all default to `0.0.0.0`.
  - The core documents this. But InfluxDB holds the all-access token, and mqtt-ui doesn't need to be reachable from the LAN.
  - Templates have no `*_BIND` settings at all, for example `- "8086:8086"`.
- **Fix:** default InfluxDB and mqtt-ui to 127.0.0.1, and add `*_BIND` settings to the templates.

#### L9. The ei-runner image is unhardened and built from floating inputs
Confirmed · `stacks/ai/runner/Dockerfile:3,14`, `stacks/ai/runner/requirements.txt:8-11`, `stacks/ai/docker-compose.yml:200-245`

- **Problem:**
  - `FROM python:3.11-slim` has no digest, and the Dockerfile has no `USER`.
  - Dependencies use open ranges (`onnxruntime>=1.17`, `paho-mqtt>=2.0`) and are built on users' machines.
  - The service has no `cap_drop`, `read_only` or `no-new-privileges`.
  - It falls back to `INFLUXDB_TOKEN: ${INFLUXDB_TOKEN:-p4n4-stack-token}` (:229).
- **Fix:** match System 1: digest-pinned base, hash-locked requirements per architecture, a non-root user, dropped capabilities.

#### L10. OBD2: the guard is broader than needed, and the VIN hash can be reversed
Confirmed · `stacks/extras/obd2/service/obd2_bridge/guard.py:29`, `stacks/extras/obd2/service/obd2_bridge/bridge.py:225-226`

- **Problem:**
  - The guard allows `ATSH` to any CAN ID and `ATSP A`–`C`. No untrusted input reaches the guard today, so this is defense in depth.
  - The VIN "hash" is `hashlib.sha256(vin.encode()).hexdigest()[:16]`. If you know the car's make, model and year, it can be brute-forced.
  - The unauthenticated `/api/v1/probe` serves it.
- **Fix:**
  - Allow only `SH7DF|SH7E[0-7]` and `SP[0-9]`.
  - Use an HMAC with a per-install key.

#### L11. Cloud bootstrap runs remote scripts as root and builds from `main`
Confirmed · `tools/emu/terraform/aws/emulator/cloud-init.yaml.tftpl:14,18-19` (gcp :18-21, azure :39-42), `aws/emulator/variables.tf:39`

- **Problem:**
  - `curl -fsSL https://get.docker.com | sh` and the uv installer run as root.
  - The clones default to `main`.
  - `${git_refs.*}` goes unvalidated into `su - ubuntu -c '…'`.
- **Fix:**
  - Default to release tags and add a `validation` regex.
  - Install Docker from its signed apt repository.
  - Install a pinned uv release and verify its checksum.

#### L12. Every AWS emulator can read and overwrite every run's project and report
Confirmed (the IAM role); Suspected (whether containers can reach it) · `tools/emu/terraform/aws/shared/main.tf:109-114`, `aws/emulator/main.tf:65-66`

- **Problem:**
  - The role grants `s3:GetObject` and `s3:PutObject` on all `projects/*` and `reports/*`, plus `ListBucket`.
  - Shipped projects can include `.env`.
  - `metadata_options` sets only `http_tokens`, so the metadata hop limit depends on the AMI.
- **Fix:** use per-run prefixes or presigned URLs, and set `http_put_response_hop_limit = 1`.

#### L13. Two helper commands run mutable `:latest` images
Confirmed · `tools/emu/scripts/install-binfmt.sh:7`, `stacks/ai/Makefile:240`

- **Problem:**
  - `docker run --privileged --rm tonistiigi/binfmt --install arm64` pulls an unpinned image and runs it with full host privileges. `p4n4-emu setup` pins the same image by digest (setup.py:89-93).
  - The ai Makefile's test uses `curlimages/curl:latest`.
- **Fix:** have the script call `p4n4-emu setup`, and pin curl by tag and digest.

#### L14. Most repos have no dependency updates, and lib and cli publish without checks
Confirmed · only `api/` and `dashboard/` have `.github/dependabot.yml`; `lib/.github/workflows/publish.yml:19-20` (cli's is identical)

- **Problem:**
  - Digest-pinned base images elsewhere (e.g. `stacks/ai/system1/Dockerfile:3`) never get security bumps.
  - lib and cli publish any release after only `python -m build`. Unlike tools/emu (`publish.yml:22-28`), they run no tests and don't check that the tag matches the version.
- **Fix:** add a dependabot.yml to each repo. Add `needs: test` and the version check to both publish workflows.

#### L15. The template validator and CI have gaps
Confirmed · `tools/templates/scripts/validate.py:128`, `tools/templates/.github/workflows/ci.yml:15-16,37-39`

- **Problem:**
  - The dashboard-key allowlist lacks `vision` and `chats`, which lib accepts (`lib/p4n4_lib/manifest.py:96`).
  - Image tags are never checked, although `tools/templates/docs/authoring.md:12` says "Never use `latest`".
  - CI runs only the smoke tests, so the go2rtc token tests never run.
  - A `.pyc` file is tracked in git.
- **Fix:**
  - Share one key set with lib.
  - Reject untagged images.
  - Run `tests/test_*.py` in CI.
  - `git rm --cached` the `.pyc`.

#### L16. The AWS emulator's TTL is lost on reboot
Confirmed · `tools/emu/terraform/aws/emulator/cloud-init.yaml.tftpl:11-12`

- **Problem:**
  - The TTL is `shutdown -P +${ttl_minutes}` in `runcmd`, which runs only once per instance.
  - A reboot or a stop/start drops it, and the instance keeps billing, although `docs/guides/aws.md:28` promises it "terminates itself".
  - Azure already uses a persistent systemd timer (azure cloud-init:36).
- **Fix:** use the same `Persistent=true` timer on AWS, and update the docs and the tftest.

#### L17. Disk limits may target the client's disk instead of the engine's
Suspected · `tools/emu/p4n4_emu/utils/docker_info.py:16,39`, `tools/emu/p4n4_emu/commands/up.py:228`

- **Problem:**
  - The engine's `DockerRootDir` is looked up in the client machine's `/proc/mounts`.
  - On Docker Desktop, WSL2 or a remote `DOCKER_HOST`, `blkio_config` gets a device from the wrong machine. Containers may then fail to start, or an unrelated disk gets throttled.
- **Fix:** return None when the engine isn't local; `docker_host().desktop` already exists.

#### L18. One idle connection can wedge the runner's HTTP server
Confirmed · `stacks/ai/runner/runner.py:56,189,264`

- **Problem:**
  - `HTTPServer` is single-threaded and has no socket timeout.
  - A client that stalls in `self.rfile.read(length)` blocks `/health`, `/api/v1/info` and `/api/v1/infer`, so the healthcheck fails.
  - The server is reachable on 127.0.0.1 and on p4n4-net.
- **Fix:** use `ThreadingHTTPServer` and set `timeout = 10` on the handler; `_infer_lock` already serializes inference.

#### L19. The OBD2 simulator's memory grows without limit
Confirmed · `stacks/extras/obd2/simulator/elm327_sim.py:138,169-174`

- **Problem:**
  - Every command is appended to `self.requests` and logged.
  - The `obd2-sim` service runs under `restart: unless-stopped` without `--quiet`, at about 22 requests per second.
- **Fix:** use `collections.deque(maxlen=10_000)`, and pass `--quiet` in compose.

#### L20. The extras validator misses long-syntax ports
Confirmed (from the code; the Compose behavior is from the spec) · `stacks/extras/scripts/validate.py:123-126`, `stacks/extras/tests/test_validate.py:267`

- **Problem:**
  - A dict port without `published` is skipped, and the test locks that in: `([{"target": 80}], [])`.
  - The short form `"80"` is flagged, yet Compose publishes both forms on an ephemeral port on every interface.
- **Fix:** always append `host_ip` for dict ports.

#### L21. Braces in a default may break the result topic on older Compose
Suspected · `stacks/ai/docker-compose.yml:226`, `README.md:104`

- **Problem:**
  - Current Compose renders `${MQTT_TOPIC_RESULTS:-inference/{device}/result}` correctly, as `docker compose config` shows.
  - compose-go releases before its fix of February 2024 misparse the inner `}`.
  - The README only asks for "Docker 24+ (with Compose v2)".
- **Fix:** set the topic's default in runner.py, and state a minimum Compose version.

#### L22. The "must match p4n4-iot" comment is wrong for `MQTT_PORT`
Confirmed · `stacks/ai/docker-compose.yml:220-222`, `stacks/ai/.env.example:130-133`, `stacks/iot/docker-compose.yml:36`

- **Problem:**
  - In iot, `MQTT_PORT` is the host port mapped to listener 1884.
  - ei-runner uses the same variable as the in-network port on `p4n4-mqtt`.
  - So copying a changed host port, as the comment says, breaks the runner.
- **Fix:** hard-code the in-network port (or use a separate variable), and fix the comment.

#### L23. `p4n4 up --no-detach` never starts the second stack
Confirmed · `cli/p4n4/commands/lifecycle.py:156-168`

- **Problem:**
  - Each stack runs `compose.up(..., detach=not no_detach)` in turn.
  - An attached `up` returns only when that stack stops, so later stacks never start.
  - `logs` already guards the multi-stack case (:336).
- **Fix:** reject `--no-detach` when there are several stacks, or detach all but the last.

#### L24. The InfluxDB password has no 8-character minimum
Suspected (InfluxDB's behavior wasn't run) · `cli/p4n4/commands/init.py:328-331`

- **Problem:**
  - The prompt passes no `min_length`; only the n8n key prompt does (:363).
  - InfluxDB 2.x setup rejects passwords shorter than 8 characters.
- **Fix:** pass `min_length=8`.

#### L25. An interrupted `init` leaves a partial project and temp clones behind
Confirmed (the Windows clean-up part is Suspected) · `cli/p4n4/commands/init.py:239,462`, `lib/p4n4_lib/scaffold.py:99-108,157-159`

- **Problem:**
  - The project is removed under `except Exception` (init.py:462), which doesn't catch Ctrl+C. The half-written project, possibly with a layer's generated `.env`, stays on disk, and a re-run stops at "already exists" (:239).
  - `fetch_source` calls `mkdtemp` (scaffold.py:99) before a `git clone` that has no timeout. If the clone raises (Ctrl+C, or Git not installed), the temp clone leaks, because the caller's `finally` (:157-159) only covers what happens after `fetch_source` returns.
  - Both clean-ups use `rmtree(..., ignore_errors=True)` (:108, :159), which on Windows silently leaves Git's read-only pack files behind.
- **Fix:**
  - Clean up on `BaseException`, then re-raise.
  - In `fetch_source`, wrap the clone in try/except and give it a timeout.
  - Pass rmtree an `onexc` handler (`onerror` before Python 3.12) that clears the read-only bit and retries.

#### L26. CORS blocks the API's PUT routes
Confirmed · `api/p4n4_api/main.py:116`

- **Problem:**
  - `allow_methods=["GET", "POST", "PATCH", "DELETE"]` leaves out PUT.
  - Saving views, view settings and the banner (`routes/dashboard.py:130,162,194`), and the assistant config (`routes/agents.py:240`), all use `PUT`.
  - A cross-origin client allowed through the CORS origins therefore fails at preflight.
- **Fix:** add `"PUT"`.

#### L27. The API's compose file defaults to an older image
Confirmed · `api/docker-compose.yml:13`, `api/.env.example:9`, `api/pyproject.toml:7`

- **Problem:** `ghcr.io/raisga/p4n4-api:${P4N4_API_VERSION:-0.3.0}` and `P4N4_API_VERSION=0.3.0`, while the package is 0.4.0. The README's documented run (`docker compose up -d`, `api/README.md:270`) therefore starts the previous release.
- **Fix:** bump both in the release process, and have CI check them against `pyproject.toml`.

#### L28. `p4n4-emu` preflight crashes without the Docker CLI
Confirmed · `tools/emu/p4n4_emu/utils/preflight.py:43,71`

- **Problem:**
  - `_run` calls `subprocess.run` (:43) without catching `FileNotFoundError`.
  - `run_preflight` runs `docker compose version` (:71) even after `docker_host()` has returned None.
  - So without the Docker CLI, `setup` (setup.py:38) and the commands that call `check_or_exit` (`up`, `calibrate`) crash with a traceback instead of reporting that Docker is missing.
- **Fix:** catch the error and return 127, and skip the Compose check when there's no host.

#### L29. Emulated CPU limits can exceed the engine's CPU count
Suspected · `tools/emu/p4n4_emu/overlays/generator.py:264,269`

- **Problem:**
  - `cpus = profile.cpus * cpu_factor * share.cpu * own.cpu` is never capped. For example, ollama gets 2.32 CPUs on jetson-orin-nano.
  - On a 2-CPU engine, Docker rejects the value and `up` rolls back.
- **Fix:** use `min(cpus, host_cpus)`, as the cpuset path already does (:180).

#### L30. The hardware button's restart skips several containers
Confirmed · `tools/hw/scripts/rpi5/p4n4_common.py:63-66`

- **Problem:**
  - The hard-coded list omits p4n4-loki, -alloy, -mqtt-ui, -agent, -obd2 and -obd2-sim.
  - `tools/hw/README.md:109` says it restarts "each p4n4 container".
- **Fix:** add the missing names in start order, or build the list from `docker ps --filter name=^p4n4-`.

#### L31. Golden fixtures can be captured from uncommitted code
Confirmed · `xtask/src/golden.rs:24-25,202,225-228,274,280-304`

- **Problem:**
  - The doc comment (:24-25) says a capture runs only when each submodule "is at the commit the superproject pins".
  - The only check, though, is `git submodule status` (:274), whose first character (:284-300) flags a wrong commit but not uncommitted changes. Nothing runs `git status --porcelain`.
  - So a capture can record fixtures from code that is in no commit, while the manifest names the pinned commits (:202).
  - The final swap isn't atomic either: the old fixtures are deleted (:225-227) before the rename (:228), so a failed rename leaves only `<kind>.partial`.
- **Fix:**
  - Refuse to capture when `git status --porcelain` is non-empty in the superproject or any baseline submodule.
  - Rename the old fixtures aside, rename the new ones in, then delete the old.

#### L32. An empty `SITE_TOKEN` crash-loops the go2rtc template's ingest
Confirmed · `tools/templates/projects/mqtt-influx-grafana-ollama-go2rtc/iot/docker-compose.yml:29`, `iot/ingest/ingest.py:91-92`

- **Problem:** with an empty token, Compose builds `demo-road:` and the ingest exits on it. This contradicts `.env.example:57`, "Empty: every write is refused".
- **Fix:** skip pairs with empty tokens and log a warning.

#### L33. Templates have drifted from the core stacks
Confirmed · e.g. `tools/templates/projects/mqtt-influx-grafana/docker-compose.yml:66,110,177`

- **Problem:**
  - Every template uses `grafana/grafana-oss:12.4.3`, while the core notes that `grafana/grafana-oss` stopped at 13.0.2 and uses `grafana/grafana:13.0` (iot compose:176-177).
  - They hard-code `subnet: 172.20.0.0/16`, where the core lets `P4N4_SUBNET` override it (iot compose:330).
  - They use `INFLUXDB_RETENTION`, where the core uses `INFLUXDB_RAW_RETENTION` (iot compose:92).
  - The two templates that ship Node-RED (`mqtt-nodered-influx-grafana` and `…-pidurable`) don't set `NODE_RED_CREDENTIAL_SECRET`, which the core passes (iot compose:146).
- **Fix:** align them with the core, or document each difference.

#### L34. The CLI repo contains a stale project with public credentials
Confirmed · `cli/test-project/docker-compose.yml:42,45,69,73,101,108`

- **Problem:**
  - It uses `${INFLUXDB_PASSWORD:-adminpassword}` and `${INFLUXDB_TOKEN:-p4n4-stack-token}`.
  - It uses `:latest` images and publishes `"1880:1880"`, and its settings.js has no `adminAuth`.
  - The only reference to it is an exclude in `cli/pyproject.toml:52`.
- **Fix:** delete it, and build test fixtures in a temp directory.

#### L35. The security stance and guide are out of date
Confirmed · `README.md:11`, `SECURITY.md:7`, `cli/README.md:48`, `.github/ISSUE_TEMPLATE/bug_report.yml:62`, `docs/guides/security.md:34,37,71`

- **Problem:**
  - The stance still covers "p4n4 0.2.x", while releases are now at 0.4.x (dashboard 1.5.0).
  - security.md has three errors:
    - it names `P4N4_API_HOST`, but Docker uses `P4N4_API_BIND`;
    - its list of all-interface ports leaves out 8883 and 8081;
    - it cites `config/mosquitto/entrypoint.sh`, which doesn't exist (the script is `bridge.sh`). `docs/stacks/iot-stack.md:259` and `docs/decisions/ai-harness.md:230` cite it too.
- **Fix:** make the stance version-agnostic, and correct the names.

#### L36. The CLI docs don't match the CLI
Confirmed · `cli/README.md:477-480,485,582,629`, `docs/reference/cli-reference.md:45,198-203`

- **Problem:**
  - `p4n4 secret` on its own is a command group and rotates nothing.
  - `p4n4 template apply` doesn't exist.
  - Both lists of rotated keys are incomplete.
  - `--source-dashboard` (`init.py:176`) isn't documented.
- **Fix:** update both documents.

#### L37. The API docs misstate roles and endpoints
Confirmed · `api/README.md:16,31,50,68,147`, `docs/reference/api.md:63`

- **Problem:**
  - The 🔒 legend says "normie, operator or admin".
  - But `GET /devices`, `POST /inference` and `POST /mqtt/publish` are operator-only (`main.py:140,142`, `routes/inference.py:114`).
  - Line 147 advertises a `/metrics` endpoint, which `release-notes/v0.4.0.md:29` says isn't implemented.
- **Fix:** mark the operator-only rows, and list `/metrics` as planned.

#### L38. The template docs point to missing files and commands
Confirmed · `CONTRIBUTING.md:48-49`, `.github/ISSUE_TEMPLATE/template_submission.yml:8`, `tools/templates/README.md:5`, `docs/reference/template-registry.md:15`

- **Problem:**
  - `TEMPLATE_GUIDE.md` and `p4n4-template.toml` don't exist.
  - `p4n4 template install` isn't implemented.
- **Fix:** point to `tools/templates/docs/authoring.md` and `tools/templates/scripts/validate.py`, and mark the commands as planned.

### Info

#### I1. Telegraf mounts the whole host filesystem
`stacks/iot/docker-compose.yml:250`

- **Detail:** `- /:/hostfs:ro` mounts the host's entire filesystem, read-only, into the Telegraf container, so any file Telegraf's user can read is exposed. No doc mentions it.
- **Fix:** mount only the paths the inputs need, or document the mount.

#### I2. The AWS emulator's root volume isn't explicitly encrypted
`tools/emu/terraform/aws/emulator/main.tf:69-73`

- **Detail:** `root_block_device` has no `encrypted = true`, so the volume holding the emulated project is encrypted only if the account enables EBS encryption by default.
- **Fix:** set `encrypted = true`.

#### I3. The GCP and Azure `report` outputs hard-code `~/greenhouse`
`tools/emu/terraform/gcp/emulator/outputs.tf:39`, azure `outputs.tf:50`

- **Detail:** the command fails for any other project.
- **Fix:** add a `project_dir` variable.

#### I4. Emulator overlays outlive their containers
`tools/emu/p4n4_emu/overlays/paths.py:7-8,42-45`, `cli/p4n4/commands/lifecycle.py:202-206`

- **Detail:** p4n4-emu takes an existing overlay to mean "the stack is running under p4n4-emu" (paths.py:7-8). `p4n4 down --all` stops projects with a plain `compose.down_project` and leaves the overlays behind. p4n4-emu's `status`, `logs` and `down` then still treat those stacks as emulated, with their old profile.
- **Fix:** in `_down_all`, hand emulated stacks to `p4n4-emu down`, as the per-stack `down` already does (:244-247). Treat an overlay as live only while containers created with it exist.

#### I5. No test checks that the proxy strips credentials
`dashboard/.github/workflows/image.yml:76-100`

- **Detail:** the smoke tests only check status codes.
- **Fix:** add an echo upstream and assert which headers it receives.

#### I6. Android release builds are signed with the debug key
`dashboard/android/app/build.gradle.kts:35-36`

- **Detail:** the file says `// Signing with the debug keys for now`.
- **Fix:** don't distribute CI-built APKs until a release key is configured.

#### I7. `jinja2` is an unused dependency
`cli/pyproject.toml:30`

- **Detail:** nothing in cli, lib or the tests imports it.
- **Fix:** remove it from `dependencies`.

#### I8. The quick-start prerequisites and the URL table are incomplete
`README.md:102-107`, `README.md:186-198`

- **Detail:** Git isn't listed, although `init` runs it (`lib/p4n4_lib/scaffold.py:103`). The service URL table omits ports 8883, 8081 and 8095.
- **Fix:** add Git to the prerequisites, and add the three ports to the table.

#### I9. The dashboard docs have drifted
`docs/stacks/dashboard.md:19-25,57-70,98-99`, `docs/reference/dashboard.md:119-120`, `dashboard/SERVICE_INTEGRATION.md:282`

- **Routes:**
  - The routes table (`docs/stacks/dashboard.md:19-25`) omits `/svc/node-red/`, `/svc/n8n/` and `/swagger-ui`, all of which nginx serves (`dashboard/docker/nginx/default.conf.template:134,143,155`).
  - The "Fixed upstreams" note (:98-99) still says the proxy forwards only to p4n4-api and Grafana.
- **Env vars:** the variables table (:57-70) omits `NODE_RED_UPSTREAM`, `N8N_UPSTREAM` and `DASHBOARD_FRAME_ORIGINS`.
- **Token storage:** `docs/reference/dashboard.md:119-120` says the refresh token is kept in "the platform's secure storage". It leaves out the web caveat that `dashboard/README.md:154` gives: there, it "isn't real protection against scripts on the same origin".
- **Stale example:** `SERVICE_INTEGRATION.md:282` still shows `ollamaBase` and `lettaBase` in `/config.json`, which the dashboard no longer serves.
- **Fix:** update both tables, the upstreams note and the storage note, and mark SERVICE_INTEGRATION.md as historical or refresh it.

## 4. Positive observations

- **Secrets and subprocesses:**
  - `p4n4 init` generates secrets with `secrets.token_hex` (`lib/p4n4_lib/secrets.py:57`) and overwrites every public default it knows about.
  - It refuses an existing directory (`init.py:239`), writes a `.gitignore` that keeps the `.env` files out of git (:460), and prints no secrets.
  - The CLI never uses `shell=True`; its subprocess calls take argv lists.
  - n8n credentials reach the container on stdin under `umask 077` (`cli/p4n4/commands/ai.py:169`).
- **CLI safety:**
  - Destructive commands ask for confirmation.
  - `migrate` keeps the old layer in a backup directory (`cli/p4n4/commands/migrate.py:102`).
  - `ensure_network` creates `p4n4-net` with Compose's label.
  - `up` checks for container-name conflicts first and prints the command that frees each name (`lifecycle.py:49-65,150`).
- **API authentication:**
  - Passwords use argon2id, with a dummy hash so unknown users take the same time. Tokens use a fixed HS256.
  - Role and session are re-read from the database on every request (`api/p4n4_api/auth.py:205-221`).
  - Refresh tokens are single-use; reusing one revokes its whole family.
  - Flux and line protocol are escaped, and CORS is configured without credentials.
- **Container hardening:**
  - The API runs as UID 10001 with a read-only root filesystem, `cap_drop: ALL` and `no-new-privileges`. It binds to 127.0.0.1 and uses digest-pinned base images.
  - Access to the Docker socket is opt-in and guarded by `DOCKER_GID:?`.
  - System 1, the agent and the OBD2 services also run read-only with `cap_drop` and `no-new-privileges` (`stacks/ai/docker-compose.yml:130-136,267-273`, `stacks/extras/obd2/docker-compose.yml:24-27,64-66`). The dashboard runs on nginx-unprivileged.
- **Dashboard:**
  - Its own pages have a strict CSP, and the access token stays in memory.
  - The proxy forwards only to configured upstreams and strips the API token before embeds (`embed_proxy.conf:10-13`).
  - CI runs Trivy and publishes provenance and an SBOM.
- **Services:**
  - Node-RED's `adminAuth` fails closed.
  - The AI stack binds to 127.0.0.1 by default (n8n excepted), and its compose file uses no `latest` tags.
  - The agent builds its Flux queries from fixed templates, gives its read-only tools budgets, and compares tokens with `timingSafeEqual`.
  - System 1 requires a key of 32 or more characters, compares it with `compare_digest`, and hash-locks its dependencies.
  - The OBD2 guard is a fail-closed allowlist.
  - bridge.sh hashes MQTT passwords with `mosquitto_passwd` under `umask 077`.
- **Cloud tooling:**
  - No emulator accepts public inbound traffic. AWS uses SSM with no inbound rules (`aws/shared/main.tf:130`), GCP allows SSH only through IAP, and Azure by default allows SSH only from the operator's own /32.
  - AWS requires IMDSv2 (`aws/emulator/main.tf:65-66`).
  - Runs expire on their own: AWS terminates on shutdown, GCP deletes the VM at `max_run_duration`, and Azure uses a persistent TTL timer. AWS's timer has the reboot gap in L16.
  - Providers are pinned, and every `.terraform.lock.hcl` is committed.
- **Templates and hardware:**
  - The Node-RED templates compare passwords in constant time and refuse every sign-in while no admin password is set (`mqtt-nodered-influx-grafana/config/node-red/settings.js:23`).
  - The n8n template provisions its owner account from `.env` at every start and refuses an empty password (`ai/config/n8n/entrypoint.sh:17-21`), which avoids M1's first-visitor problem.
  - The tools/hw installer refuses to run as root and installs `hwd.env` as root-owned 0600 (`tools/hw/scripts/rpi5/install-hwd.sh:23,53`).

## 5. Top 5 actions

1. **Fix the exploitable templates (H1, H2, M18):**
   - Limit go2rtc's API to the stream paths and give it credentials. Bind it, like the template agent, to 127.0.0.1 unless the user opts in.
   - Make the cold-chain template refuse to start without a generated signing key, and compare tokens in constant time.
2. **Close the anonymous MQTT attack surface (H3, M3, L7):**
   - In the agent, bound alert timestamps, format dates safely and acknowledge messages that can never be processed.
   - Have `p4n4 init` generate MQTT credentials so the broker requires authentication by default.
3. **Remove the default-credential paths (M1, M2, M11, M16):**
   - Bind n8n to 127.0.0.1, and either provision its owner from `.env` or drop the prompt that does nothing.
   - Fall back to empty values instead of the public example secrets.
   - Write `.env` with mode 0600, atomically.
   - Keep `scripts/dev` on loopback.
4. **Pin the supply chain (M6–M10):**
   - Clone the stacks at release tags.
   - Give `p4n4-lib` an upper bound.
   - Build the API image from `uv.lock`.
   - Pin actions by SHA, with least-privilege `permissions:` and Dependabot.
   - Add CI to stacks/iot.
5. **Bring the API and dashboard in line with the CLI (M4, M5, M14, M17, L2):**
   - Trust X-Forwarded-For only from the dashboard's address, and have nginx overwrite the header.
   - End SSE streams when a session is revoked.
   - Make refresh-token rotation atomic.
   - Point the dashboard at `http://p4n4-api:8000` by default.
   - Move start and stop into lib so both front ends share one code path.

## 6. Coverage note

- **Reviewed:** a fresh recursive clone of raisga/p4n4 `main` at 512e385 (README, CONTRIBUTING, SECURITY, `.gitmodules`, `.github/`, `scripts/dev`, `xtask/`), plus all 11 public submodules at their pinned commits:

  | Submodule | Commit | Version |
  |---|---|---|
  | api | 60531e0 | v0.4.0 |
  | cli | cc46b0e | v0.4.2 |
  | dashboard | a2629a7 | v1.5.0 |
  | docs | e2a5dbf | |
  | lib | 51dc352 | v0.4.2 |
  | stacks/ai | cb23a02 | |
  | stacks/extras | d684d4d | |
  | stacks/iot | d80ea1b | |
  | tools/emu | 9ba42c0 | |
  | tools/hw | 432c74e | |
  | tools/templates | 30ace57 | |

- **Method:**
  - Eight parallel read-only review passes, one per area:
    - root and CI;
    - Compose stacks;
    - AI and extras services;
    - lib and CLI;
    - API;
    - dashboard;
    - templates;
    - emu and hw.
  - Then consolidation and severity recalibration. Every High and Medium was re-read in the clone, and the Low evidence was spot-checked.
- **External sources:** the go2rtc v1.9.14 source (H1), and compose-go's template parser and its history (L21).
- **Executed:**
  - `docker compose config` (L21).
  - In-memory runs of repo code by the sub-reviews:
    - `parseAlert` under node (H3);
    - the runner's ONNX post-processing (M19);
    - the OBD2 guard's `check()` (L10);
    - a Starlette CORS preflight (L26);
    - a SQLite refresh race (L2).
- **Not run:** `docker compose up`, the test suites, `flutter analyze` and `flutter test`, Terraform, and any live exploit. That is why the runtime-dependent claims are marked Suspected.
- **Skipped or partial:**
  - `web/blog` and `web/p4n4.com` are private and weren't cloned.
  - `projects/sandbox` isn't on `main`.
  - The Flutter UI was reviewed only along its auth, storage, settings and proxy paths.
  - Lockfiles were scanned by script, and most test bodies were only skimmed.
