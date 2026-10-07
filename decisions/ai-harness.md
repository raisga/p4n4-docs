# p4n4 — AI Harness: Building p4n4 Projects with Coding Agents

**Version:** 0.2
**Date:** 2026-10-03
**Status:** Draft. Nothing in this document is implemented yet. The OBD-II spike (§6) has
done its desk research; the in-car work hasn't started.

> **About this version.** Version 0.1 (2026-09-23) was never committed. Other documents cite
> it: [Known Issues](../project/known-issues.md), the code review and [System 1](system1.md).
> This version is a rewrite that keeps those references valid:
> - §4 is the tool catalog (`edge_deploy_model`, `nodered_flows_apply`, `system1_decide`);
> - §8 is the gap table, and G3, G4, G5, G6, G8 and G9 keep their meanings;
> - open questions 3 and 4 keep their numbers.

Evidence tags used throughout:

| Tag | Meaning |
|-----|---------|
| **[source]** | Read in the p4n4 source (the working trees of 2026-10-03, v0.2.0 release prep) |
| **[upstream]** | Stated by a vendor, a datasheet, PyPI or a community source; listed in Appendix C |
| **[estimate]** | Our judgement, still to be measured |
| **[spike]** | To be measured in the OBD-II spike (§6) |

## Contents

1. [Summary](#1-summary)
2. [Goals and Non-Goals](#2-goals-and-non-goals)
3. [Reference Project: p4n4 Car](#3-reference-project-p4n4-car)
4. [The Harness](#4-the-harness)
5. [Extras: Community Stacks](#5-extras-community-stacks)
6. [Spike S-OBD: OBD-II on a 2011 Nissan Cube](#6-spike-s-obd-obd-ii-on-a-2011-nissan-cube)
7. [Security, Privacy and Safety](#7-security-privacy-and-safety)
8. [Gaps](#8-gaps)
9. [Phases](#9-phases)
10. [Evaluation](#10-evaluation)
11. [Risks](#11-risks)
12. [Open Questions](#12-open-questions)
- [Appendix A. Example `extra.yaml`](#appendix-a-example-extrayaml)
- [Appendix B. Example Tool Definition](#appendix-b-example-tool-definition)
- [Appendix C. Sources](#appendix-c-sources)

---

## 1. Summary

The AI harness (`p4n4-harness`) lets a coding agent build, run and operate a full p4n4
project from a written brief: ingest, storage, edge inference, generative AI and the
dashboard. It's an **MCP server**: typed tools with permission tiers, a dry run for every
change and an audit trail. It comes with **skills**, playbooks that tell an agent how to use
those tools for common jobs. Any MCP host can drive it, Claude Code included.

The harness stays thin. It doesn't reimplement p4n4. It calls the pieces that already exist:
- `p4n4-lib` to scaffold and validate projects;
- `p4n4-api` for state, telemetry, stack control, MQTT publishing and the audit log;
- the stacks' own APIs (the Node-RED Admin API, the edge runner's `/api/v1/*`).

Where those pieces are missing something, the gap goes in §8 rather than into the harness.

The design is tested against one concrete project, **p4n4 Car**: a Raspberry Pi 5 in a 2011
Nissan Cube. It does the following:
- reads the car over OBD-II;
- stores everything locally and encrypted;
- flags unusual engine behaviour with an edge model;
- explains trouble codes in plain language with a local LLM;
- shows it all on a phone over the Pi's own Wi-Fi.

No cloud, and no data leaves the car unless the owner syncs it at home.

The car needs a protocol adapter that p4n4 doesn't have. That leads to the second proposal:
**extras**, community-maintained stacks mounted at `stacks/extra/` from a new `p4n4-extras`
repository. Each extra declares a contract: hardware, MQTT topics, env keys, a safety class,
tests against a simulator and its own license. OBD-II is the first extra; Modbus and raw CAN
are next.

Before any of this is built, a spike (§6) answers the risky questions on the real car:
- which adapter and library to use;
- how fast the car answers;
- whether the CVT's Nissan-specific data can be read;
- how to power the Pi without draining the battery;
- whether the hardware survives a parked car.

## 2. Goals and Non-Goals

**Goals**

- **G-A.** An agent can go from a brief ("log my car, explain warning lights, keep the data
  private") to a running, validated project, using only harness tools and skills.
- **G-B.** Every change the agent makes:
  - is previewable (dry run);
  - is attributable (an audit entry in p4n4-api);
  - is reversible where possible;
  - can actuate anything only after a human approves it.
- **G-C.** New device protocols arrive as extras with a common contract, so the harness and
  the rest of p4n4 treat an OBD-II adapter, a Modbus meter and a CAN bus the same way.
- **G-D.** p4n4 Car works fully offline, stores data encrypted at rest, and survives power
  cuts and heat.

**Non-goals**

- Writing to the car. No ECU flashing, coding, actuator tests or anything that sends
  non-diagnostic frames. The only write the design allows is clearing trouble codes, behind
  approval (§7.3).
- Driving assistance or anything safety-critical. p4n4 Car is an observer.
- Replacing a professional scan tool. Nissan's dealer tooling stays the reference.
- A hosted harness. It runs on the developer's workstation or on the device, never as a
  shared service.
- Coupling to one agent product. Tools are plain MCP; skills are Markdown.

## 3. Reference Project: p4n4 Car

### 3.1 Why this project

p4n4 Car uses every layer:

- **Extra:** a new protocol (OBD-II) brought in by a new kind of stack.
- **IoT:** MQTT, ingest, InfluxDB and Grafana (the MING stack).
- **Edge:** an anomaly model on engine features, served by the edge runner.
- **AI:** Ollama explains trouble codes and summarises trips.
- **Dashboard:** a phone UI with a vehicle view.

It also adds constraints the platform hasn't had to meet so far:
- unreliable power;
- no network;
- high temperatures;
- a moving, stealable device that holds location data.

### 3.2 User stories

| # | As | I want | So that |
|---|---|---|---|
| C1 | driver | live gauges on my phone (speed, rpm, coolant, fuel trims, battery voltage, CVT temperature) | I see the car's health at a glance |
| C2 | driver | a plain-language explanation when the check-engine light comes on | I know whether to stop now or book a service |
| C3 | owner | trip summaries (distance, duration, idle time, fuel estimate, warnings) | I understand how the car is used and wearing |
| C4 | owner | the data encrypted on the device and kept only as long as I choose | a stolen device or a resale doesn't leak my routes |
| C5 | owner | the data synced to my home p4n4 when the car is on my Wi-Fi | I keep a long history without a cloud service |
| C6 | owner | a warning when the engine behaves unusually for this car | I catch problems (overheating, lean running, a failing CVT) early |
| C7 | developer | an agent to scaffold, extend and debug all of this | I spend my time on the car, not on plumbing |

### 3.3 Hardware (baseline)

| Part | Choice | Notes |
|---|---|---|
| Computer | Raspberry Pi 5, 8 GB | Matches `tools/hw` and the emulator's `rpi5` profile (4 CPUs, 7 GiB) [source] |
| Storage | NVMe SSD on an M.2 HAT | The SD card holds only the boot partition; data lives on the SSD (LUKS2, §3.6) |
| OBD-II adapter | STN-based USB adapter (OBDLink SX baseline, EX as an alternative) | Genuine chip, much faster serial link than ELM327 clones [upstream]; USB avoids Bluetooth pairing |
| Power | Car power HAT with ignition (ACC) sense and safe shutdown (e.g. CarPiHAT PRO 5) | Up to 5 A, as a Pi 5 can need [upstream]; picked in the spike (§6.4, S-OBD-3) |
| Clock | Pi 5 RTC with a backup battery | Correct timestamps without network time [upstream] |
| Location (optional) | USB GNSS receiver via `gpsd` | Off by default (§3.6) |
| Display | The owner's phone or a tablet | p4n4-dashboard over the Pi's Wi-Fi hotspot |

### 3.4 Architecture

```
  2011 Nissan Cube
  ECM / TCM ── CAN (ISO 15765-4) ── OBD-II port (DLC, under the dash)
                                       │ USB
  ┌────────────────────────────────────▼──────────────────────────────────────────┐
  │ Raspberry Pi 5  (Wi-Fi hotspot "p4n4-cube", services bound to it, §7)          │
  │                                                                                │
  │  extras/obd2 ──► MQTT  sensors/cube/<measurement>   (1–10 Hz per PID group)    │
  │                        sensors/cube/raw             (feature vector, 1 Hz)     │
  │                        alerts/cube/dtc               (on change)               │
  │                        status/cube/online            (retained)                │
  │                            │                                                   │
  │  iot:  Mosquitto ─► Node-RED ─► InfluxDB ◄── Grafana (optional, §12 Q8)        │
  │                            │        ▲                                          │
  │  edge: runner ◄────────────┘        │  inference/cube/result (anomaly score)   │
  │  ai:   Ollama (small model) ◄── n8n / p4n4-api agents: DTC explanations, trips │
  │  dashboard: Vehicle view ◄── p4n4-api (telemetry stream, agents, auth)         │
  │                                                                                │
  │  /data on NVMe, LUKS2 (§3.6)                                                   │
  └───────────────────────┬────────────────────────────────────────────────────────┘
                          │ at home only: MQTT bridge to the home p4n4 (§3.7, G11)
```

### 3.5 Data model

Topics follow the platform convention in specs §8.6 [source]: the device id and measurement
come from the topic, and the payload is a JSON object with the reading in `value`.

| Topic | Payload | Rate | Stored as |
|---|---|---|---|
| `sensors/cube/rpm` | `{"value": 2150, "unit": "rpm"}` | 5 Hz | `sensor_data`, tags `device=cube`, `sensor=rpm` |
| `sensors/cube/speed` | `{"value": 62, "unit": "km/h"}` | 5 Hz | same |
| `sensors/cube/coolant_temp` | `{"value": 91, "unit": "C"}` | 1 Hz | same |
| `sensors/cube/stft_b1`, `ltft_b1` | `{"value": -2.3, "unit": "%"}` | 1 Hz | same |
| `sensors/cube/maf` | `{"value": 7.9, "unit": "g/s"}` | 5 Hz | same |
| `sensors/cube/battery_v` | `{"value": 14.1, "unit": "V"}` | 0.2 Hz | same (adapter voltage reading, not a PID) |
| `sensors/cube/cvt_temp` | `{"value": 78, "unit": "C"}` | 0.2 Hz | same, **if** S-OBD-2 finds a way to read it |
| `sensors/cube/raw` | `{"values": [rpm, load, coolant, stft, ltft, maf, speed]}` | 1 Hz | not stored; feeds the edge runner |
| `inference/cube/result` | runner result with an anomaly score | 1 Hz | `inference`, tag `device=cube` |
| `alerts/cube/dtc` | `{"codes": ["P0171"], "mil": true, "freeze_frame": {...}}` | on change | new `events` measurement (G14) |
| `status/cube/online` | `{"online": true, "adapter": "OBDLink SX", "protocol": "ISO 15765-4 (CAN 11/500)"}` | retained | — |

Trips are derived, not published. A Node-RED or n8n flow opens a trip when rpm goes above
zero after ignition-on and closes it on ignition-off. It writes a `trips` row with distance,
duration, idle time, the maximum coolant temperature and any codes seen.

### 3.6 Secure local storage

The data includes where and when the car was driven, so it's personal data. The design:

- **Encryption at rest:** `/data` (InfluxDB, Node-RED, Ollama models, archives) is a LUKS2
  volume on the SSD. The device has to boot unattended in a car, so the unlock key can't be
  typed at boot. Three options, to decide in the spike (§12 Q6):
  1. a keyfile on a small USB dongle kept on the car keyring. It protects against theft of
     the device without the key, which is the common case;
  2. a key sealed in the Pi 5's one-time-programmable memory with signed boot. This needs no
     dongle, but if the whole Pi is stolen, the thief has the key too [estimate: not yet
     researched; check Raspberry Pi's secure-boot and OTP tooling];
  3. unlock from the phone over the hotspot. This is the strongest, but the car logs nothing
     until someone unlocks it.
- **Minimisation:**
  - GNSS is off unless the owner enables it;
  - raw 5 Hz data is downsampled to 1-minute aggregates after 30 days;
  - raw data is deleted after a retention period the owner sets (InfluxDB bucket retention,
    as the IoT stack already supports [source]).
- **Erase:** one dashboard action deletes a trip, and one wipes all vehicle data (and, with
  option 1, the dongle's key slot).
- **Power safety:** InfluxDB and the archive must survive a sudden power loss when the ACC
  sense fails. The spike measures it (S-OBD-3).

### 3.7 Home sync

When the Pi joins the owner's home Wi-Fi, it forwards `sensors/cube/#`, `alerts/cube/#` and
`trips` to the home p4n4 over MQTT with TLS. Today's IoT stack can only **pull** topics from
an external broker into the local one [source: `config/mosquitto/bridge.sh`, CLI
`--mqtt-remote`]. Pushing local topics out is gap G11. Queued messages are sent once and
then expire locally under the retention rules.

### 3.8 Smart interactions (first version: the dashboard)

The phone shows a **Vehicle** view (a new dashboard brand tab, G15):
- gauges, with the CVT temperature banded cold / OK / hot;
- the current codes with an explanation;
- the last trip;
- an "everything is fine / something needs attention" card for the normie role;
- an **Ask** box that calls a p4n4-api agent with telemetry tools (`/agents/{id}/chat`
  [source]), for example "why did the light come on yesterday?".

**DTC explanations:** when `alerts/cube/dtc` changes, a flow asks Ollama for a short
explanation of each code. The prompt includes the code, its generic definition (SAE J2012
table, shipped with the extra), the freeze frame and recent trends. The explanation is
stored next to the code, so it can be shown offline instantly. The prompt bans repair
advice beyond "safe to drive / stop soon / stop now" [estimate: a 1–3B model on a Pi 5 is
fast enough, because explanations aren't latency-critical].

Voice is a later phase (§9, H5).

## 4. The Harness

### 4.1 Shape

```
  MCP host (Claude Code, …) ── MCP (stdio, or streamable HTTP on localhost) ──► p4n4-harness
                                                                                   │
            ┌──────────────────────────────┬───────────────────────┬───────────────┤
            ▼                              ▼                       ▼               ▼
        p4n4-lib                       p4n4-api                Node-RED       edge runner
   scaffold, validate,          project, stacks, jobs,        Admin API       /api/v1/info
   manifest, env, layout        telemetry, mqtt/publish,      (token from     /api/v1/infer
                                agents, devices, audit        /auth/token)
```

- **Where it runs:** on the workstation during development, against a local project or the
  device over SSH or HTTPS; on the device for operations (§12 Q1).
- **Identity:** the harness signs in to p4n4-api with an account whose role matches what it
  may do (operator for read and write tools, admin only when an approved X tool needs it).
  Every call carries a `harness-session` id that ends up in `/audit` [source: the API has
  `/audit`].
- **No Docker socket.** Stack control goes through p4n4-api jobs
  (`POST /stacks/{stack}/{action}`, admin) [source]. The harness never mounts or calls
  Docker directly.
- **No secrets in tool output.** Env values are masked the way `p4n4 secret show` masks
  them [source]. A tool that needs a secret reads it server-side.

### 4.2 Tiers

| Tier | Meaning | Approval | Examples |
|---|---|---|---|
| **R** | Reads state; no side effects | none | `project_describe`, `telemetry_query`, `topic_sample` |
| **W** | Changes project files or configuration. Reversible: the dry run returns a diff and the harness keeps a backup | the MCP host's normal tool approval | `template_apply`, `extra_add`, `env_set`, `grafana_dashboard_apply` |
| **X** | Acts on running services or the outside world | explicit human approval, plus a server-side confirmation token (§7.2) | `stack_up`, `nodered_flows_apply`, `edge_deploy_model`, `mqtt_publish`, `obd_dtc_clear` |
| **N** | Never offered | — | ECU writes, raw CAN transmit, `commands/` topics for vehicle devices |

### 4.3 Tool catalog

| Tool | Tier | Calls | Notes |
|---|---|---|---|
| `project_describe` | R | lib `manifest`, api `GET /api/v1/project` | Layers, extras, template, dashboard block, versions |
| `project_validate` | R | lib `validate_project`, api `/project/validate` | Returns checks; never fixes |
| `stack_status` | R | api `GET /stacks`, `/stacks/{stack}` | Per-service state |
| `stack_logs` | R | api stack logs (admin) | Truncated and redacted; marked untrusted (§7.4) |
| `telemetry_query` | R | api `/telemetry` | Flux kept server-side; the tool takes measurement, tags, range and aggregate |
| `topic_sample` | R | MQTT subscribe for N seconds (≤ 30) | Shows real payloads before the agent writes a flow |
| `template_list`, `template_apply` | R, W | p4n4-templates registry, lib scaffold | `apply` dry-runs into a temp dir and returns the diff |
| `layer_add` | W | lib scaffold | Needs `p4n4 add` (G2) |
| `extra_list`, `extra_add`, `extra_remove` | R, W, W | p4n4-extras registry (§5) | Needs the extras registry in lib (G1) |
| `env_set` | W | lib `env` | Refuses keys read only at first setup (LIB-2) unless the stack is new |
| `nodered_flows_get` | R | Node-RED Admin API | Token from `POST /auth/token` (G8) |
| `nodered_flows_apply` | X | Node-RED Admin API `POST /flows` | Validates topics against §8.6 first; keeps the previous flows for rollback (G9) |
| `grafana_dashboard_apply` | W | provisioning files | Uses the fixed data source UIDs (`influxdb`, …) [source] |
| `edge_model_list` | R | runner `/api/v1/info`, `models/` | |
| `edge_infer` | R | runner `POST /api/v1/infer` | A 422 means the model rejected the input [source] |
| `edge_deploy_model` | X | copies the model, then **recreates** the runner container via an api job | A restart doesn't re-read `.env`, so a new model needs a recreate (KI-5) |
| `mqtt_publish` | X | api `POST /mqtt/publish` | `commands/<vehicle>/…` is refused (tier N) |
| `stack_up`, `stack_down`, `stack_restart` | X | api `/stacks/{stack}/{action}` jobs | Polls `/jobs/{id}` |
| `system1_decide`, `system1_eval` | R | System 1 service | From [System 1](system1.md) §3, U8 |
| `obd_probe` | R | obd2 extra HTTP API | Protocol, VIN, supported PIDs, adapter info |
| `obd_dtc_read` | R | obd2 extra | Stored, pending and permanent codes, freeze frame |
| `obd_dtc_clear` | X | obd2 extra | Engine off, two confirmations, and the freeze frame saved first (§7.3) |
| `audit_tail` | R | api `/audit` | What this session, or any other, changed |

**Every W and X tool:**
- takes `dry_run` (default `true` for X);
- returns what would change (a diff, the job plan, or the frames that would be sent);
- writes an audit entry when it runs for real;
- is idempotent where it can be.

Schemas are JSON Schema; Appendix B shows one.

### 4.4 Skills

Skills are Markdown playbooks that ship with the harness (as Claude Code project skills,
or as plain instructions for other hosts):

| Skill | Steps it encodes |
|---|---|
| `p4n4-new-project` | Brief → pick a template and layers → `template_apply` dry run → validate → bring up (approval) → check data flows with `topic_sample` and `telemetry_query` |
| `p4n4-add-extra` | Find or write an extra → check its contract (§5.3) → run its simulator test → `extra_add` → wire topics into storage |
| `p4n4-flow` | Sample real payloads → write the flow → check it against §8.6 → `nodered_flows_apply` with rollback |
| `p4n4-edge-model` | Collect features → train (Edge Impulse or ONNX) → `edge_infer` on held-out data → `edge_deploy_model` (approval) |
| `p4n4-protocol-spike` | The method in §6: desk research with sources, a bench test against a simulator, read-only tests on hardware, measurements and a written report |
| `p4n4-debug` | Status, logs and validate first; form a hypothesis; change one thing at a time; record what was ruled out |

### 4.5 Walkthrough: from brief to a running p4n4 Car

1. **Brief:** "Raspberry Pi 5 in my 2011 Nissan Cube. Live gauges on my phone, explain
   warning lights, encrypted storage, sync at home."
2. `template_list`, then `template_apply vehicle-obd2` (H3) with a dry run. The agent shows
   the diff and the user approves.
3. `extra_add obd2` with the adapter path `/dev/serial/by-id/…`, then `project_validate`.
4. `stack_up iot`, `stack_up extras/obd2` (approval each), then
   `topic_sample sensors/cube/#` to confirm real readings arrive.
5. `obd_probe` → supported PIDs. The agent trims the poll groups to what the car supports.
6. `grafana_dashboard_apply` and the dashboard's Vehicle tab settings (W).
7. After a week of driving: the `p4n4-edge-model` skill trains an anomaly model on normal
   driving, then `edge_deploy_model` (approval).
8. `audit_tail` lists everything the session changed.

## 5. Extras: Community Stacks

### 5.1 Why a new kind of stack

The four layers (`iot`, `ai`, `edge`, `dashboard`) are fixed in `p4n4_lib/layers.py`, with
repo URLs in `sources.yaml` [source]. Device protocols are numerous, niche and
hardware-bound. They don't belong in the core stacks, and the core team can't test them all.
Extras give them a home with a contract, so they plug into the rest of p4n4 the same way.

### 5.2 Repository

A new repository, `raisga/p4n4-extras`, mounted in the monorepo at `stacks/extra/`. It works
the same way `p4n4-templates` does:

```
stacks/extra/                       (p4n4-extras)
├── README.md                       catalog, maturity levels, how to contribute
├── schema/extra.schema.json        manifest schema (Appendix A)
├── scripts/validate.py             CI: schema, contract and license checks
├── obd2/                           first extra (§6)
│   ├── extra.yaml
│   ├── docker-compose.yml
│   ├── .env.example
│   ├── service/                    the adapter-to-MQTT bridge
│   ├── simulator/                  scenario for tests (recorded from the car, §6.7)
│   └── tests/
├── modbus/                         next: Modbus RTU/TCP → MQTT
└── canbus/                         next: SocketCAN + DBC decoding → MQTT
```

In a project, an extra is copied to `<project>/extras/<name>/` as its own Compose project on
`p4n4-net`, the same way multi-layer projects keep each layer in its own directory (ADR-002).

### 5.3 Contract

Every extra must meet these rules. `scripts/validate.py` checks the ones it can.

1. **Manifest:** `extra.yaml` validates against the schema. It declares:
   - its kind (`ingest`, `bridge`, `actuator` or `service`);
   - its hardware (device paths, interfaces);
   - the topics it publishes and subscribes to;
   - its env keys, required files and health endpoint;
   - its safety class and license.
2. **Topics:** it publishes only under §8.6 patterns: `sensors/<device-id>/<measurement>`,
   `alerts/…`, `status/<device-id>/online` (retained). Payloads are JSON objects with the
   reading in `value`. Subscribing to `commands/…` needs safety class `writes-bus` and
   maintainer review.
3. **Safety class:** `read-only` (the default) or `writes-bus`. Read-only extras must not be
   able to transmit anything other than diagnostic requests, and the tests prove it against
   the simulator.
4. **Container:** non-root, `cap_drop: [ALL]`, `no-new-privileges`, read-only root
   filesystem, only the declared devices mapped. It sits in a Compose profile of its own name,
   like every service in the stacks [source].
5. **Simulator:** a hardware-free test that runs in CI and under p4n4-emu. It publishes
   realistic data and covers disconnects.
6. **Health:** `GET /health` and a `status/<device-id>/online` heartbeat.
7. **License:** declared per extra and checked against an allowlist (§12 Q5). The repo itself
   is MIT.
8. **Ownership:** a `CODEOWNERS` entry. Extras without an active maintainer for 6 months move
   back to `incubating`.

### 5.4 Maturity

| Level | Meaning | Where it shows |
|---|---|---|
| `incubating` | Works for its author; contract checks pass | Listed with a warning |
| `community` | Tested by at least one more person on real hardware; simulator in CI | Listed |
| `official` | Maintained by the core team; images built and signed in CI; in the release notes | Listed first; may graduate to its own repo |

### 5.5 Platform changes

- **p4n4-lib:** an extras registry (from `p4n4-extras`, alongside the fixed layers), scaffold
  and validate support, and an `extras` list in `.p4n4.json` (`name`, `version`, `source`).
  The manifest probably moves to `schema_version: 2` (§12 Q7). Gap G1.
- **p4n4-cli:** `p4n4 extra list|add|remove` and `p4n4 up extras/<name>`. `p4n4 add` is a
  stub today [source] (G2).
- **p4n4-emu:** device passthrough off, simulator on, by default.
- **p4n4-dashboard:** extras can ship a brand tab (the obd2 Vehicle tab, G15).

### 5.6 Initial catalog

| Extra | Kind | Library (candidate) | Use case |
|---|---|---|---|
| `obd2` | ingest | python-OBD, or our own client (§6.3) | Vehicles, 2008+ in the US (CAN) |
| `modbus` | ingest | pymodbus (BSD-3-Clause) | Energy meters, PLCs, solar inverters |
| `canbus` | ingest | python-can (LGPL-3.0) + cantools (MIT, DBC decoding) | Raw CAN on vehicles and machines; also a faster OBD back end |
| `gps` | ingest | gpsd | Location for vehicles and assets |
| candidates | — | — | BACnet, Zigbee (through zigbee2mqtt), BLE sensors, 1-Wire |

## 6. Spike S-OBD: OBD-II on a 2011 Nissan Cube

### 6.1 Questions

| # | Question | Decides |
|---|---|---|
| S1 | Which protocol and which PIDs does this car support, and how many readings per second can we get? | Poll groups, rates (§3.5) |
| S2 | Which adapter and library: STN USB with python-OBD, our own client, or a CAN HAT? | The obd2 extra's design and license |
| S3 | Can we read the CVT's fluid temperature, which isn't a standard PID? | Whether C1's CVT gauge and the CVT part of C6 are possible |
| S4 | How do we power the Pi: ACC sense, shutdown timing, parasitic drain? | Power HAT choice, data safety |
| S5 | Does the hardware survive a parked car, summer and winter? | Enclosure, placement, whether to stay powered off while parked |
| S6 | Can an agent build the project with the harness, against a simulator, without the car? | H0–H2 scope (§9) |

### 6.2 Desk findings

- **Protocol.** All cars sold in the US from model year 2008 use ISO 15765-4 (CAN) for
  OBD-II, so the 2011 Cube should answer on CAN, most likely 11-bit IDs at 500 kbit/s
  [upstream]. S-OBD-1 confirms this with the adapter's protocol detection.
- **The car.** The 2011 Cube has the 1.8 L MR18DE engine with a 6-speed manual or an Xtronic
  CVT. Parts suppliers list the Jatco JF011E (Nissan RE0F10A) for 2007–2012 Cubes [upstream].
  We confirm it from the VIN and the transmission label (S-OBD-1).
- **CVT data isn't standard.** OEMs choose their own identifiers for transmission
  temperature, outside the standard mode 01 PIDs [upstream: OBDLink support]. Tools that read
  Nissan CVT data talk to the transmission control module directly. CVTz50 does this for some
  Jatco CVTs; it lists Murano fully and others partly, not the Cube. It warns that this needs
  ELM327 features that v2.0+ clones often lack [upstream]. So S3 is a real unknown, and the
  answer may be "not without dealer tooling".
- **Adapters.** STN-based OBDLink adapters accept the ELM327 command set plus their own
  extensions, with a much faster host link than ELM327 (up to 2 Mbit/s against 500 kbit/s)
  [upstream]. Cheap ELM327 clones vary in quality and are risky under long, heavy use
  [upstream]. The SX (STN1100) and EX (STN2230) are both USB; the EX also speaks Ford's
  MS-CAN, which we don't need.
- **Library.** python-OBD 0.7.3 (2025-04-07) supports ELM327-compatible adapters, has an
  `Async` mode, and detects the car's supported commands. Its license is **GPL-2.0-only**
  [upstream], while p4n4 is MIT. An extra that imports it is a derivative work, so that
  extra would be GPL-2.0 (§6.3, §12 Q5).
- **Simulator.** ELM327-emulator 4.0.0 (2026-09-17) emulates an ELM327 with several ECUs,
  OBD-II and UDS over ISO-TP. It serves python-OBD over a pseudo-terminal, TCP or Bluetooth,
  and accepts custom scenarios. Its license is **CC-BY-NC-SA-4.0** [upstream]. That's fine
  for running tests, but we shouldn't ship it in an image or vendor it into the MIT repo. CI
  installs it at test time.
- **Power.** OBD-II pin 16 is battery power, live with the ignition off, so a device on it can
  drain the battery over days [upstream]. Car power HATs for the Pi 5 take 12 V, sense ACC
  through opto-isolated inputs, supply up to 5 A and shut the Pi down safely [upstream].
  The design powers the Pi from a fused ACC/battery pair through such a HAT, not from the
  OBD port.
- **Heat.** The Pi 5 is specified for 0–70 °C ambient [upstream]. A parked car in the sun
  goes well past that, so the Pi must be off while parked and mounted out of the sun, for
  example under a seat. S-OBD-4 logs real temperatures.

### 6.3 Options

| | A. STN USB + python-OBD | B. STN USB + own client | C. CAN HAT + SocketCAN | D. Bluetooth ELM327 |
|---|---|---|---|---|
| Setup | Plug in | Plug in | Wire to DLC pins 6/14; terminate correctly | Pair |
| Throughput | Good [spike] | Good; can use STN batch commands [spike] | Best (raw frames) | Poor [upstream] |
| Nissan TCM access | Through raw ELM/STN commands | Same | Full (ISO-TP, UDS, Nissan services) | Unreliable on clones [upstream] |
| License | GPL-2.0 extra | MIT | MIT extra on python-can (LGPL-3.0-only), can-isotp, udsoncan, cantools (MIT) [upstream] | — |
| Risk of transmitting by mistake | Low | Low | Higher: a bug can put frames on the bus | Low |
| Cost | ~$60–100 [estimate] | same | ~$30–60 HAT plus wiring [estimate] | ~$10–30 |
| **Verdict** | **Baseline for S-OBD-1** | Decide after S-OBD-1 | Later `canbus` extra; S3 fallback | Rejected |

### 6.4 Plan

| Step | Where | What | Done when |
|---|---|---|---|
| S-OBD-0 | Bench | ELM327-emulator scenario → python-OBD → MQTT on a laptop and under p4n4-emu `rpi5` | Readings on `sensors/sim/#`; a disconnect is recovered |
| S-OBD-1 | Car, parked, engine running | Read-only: protocol, VIN (mode 09), the supported-PID bitmaps (01 00/20/40/60), the codes (03, 07, 0A); poll groups at increasing rates | PID list recorded; rate per group measured; transmission confirmed |
| S-OBD-2 | Car, parked | CVT temperature: try the requests community sources describe for this TCM, read-only, with a capture of every frame sent | A value that tracks a warm-up, or a documented "not possible" |
| S-OBD-3 | Car | Power: ACC sense, shutdown on ignition-off, a cut-power test with InfluxDB writing, drain with the ignition off | Clean shutdown in ≤ 20 s; no data loss in 20 cuts; drain < 5 mA off [estimate targets] |
| S-OBD-4 | Car, a week | Thermal: temperature logger in the enclosure, parked in the sun and driving | Max enclosure temperature; decision on placement and on staying off while parked |
| S-OBD-5 | Workstation | Harness dry run: an agent builds p4n4 Car against the S-OBD-0 simulator with the H0 tools | Project validates; data flows; no tier violations |

### 6.5 Measurements to record

- Adapter, firmware and protocol detected; the VIN's model year and transmission.
- Supported PIDs, and which of §3.5's measurements exist.
- Readings per second: one PID; groups of 4 and 8; with and without STN batch requests.
- Latency from request to MQTT publish (median, p95).
- CVT temperature: method, scaling, agreement with a reference (an IR thermometer on the pan
  after a drive).
- Boot-to-first-reading time (Pi 5 cold start); shutdown time; drain when off.
- Enclosure temperatures (max parked, max driving); any throttling.
- Pi 5 load with iot + obd2 + edge running; Ollama answer time for one DTC explanation.

### 6.6 Safety rules for the spike

- Read-only until S-OBD-1 has finished. No mode 04 (clear codes) during the spike.
- Tests that need the engine running are done parked, in the open, never in a closed garage.
  Tests while driving are done by a passenger, and the driver never looks at the laptop.
- Every frame sent to the TCM is logged. Unknown services aren't probed by brute force.
- A known-good scan tool stays in the car, to read and clear anything the spike might set.
- The adapter is unplugged when not in use until S-OBD-3 shows the drain is acceptable.

### 6.7 Deliverables

- §6 results filled in (this document, version 0.3).
- `stacks/extra/obd2` 0.1 (incubating), with the option chosen in §6.3.
- A `nissan-cube-2011` simulator scenario recorded from the car (our own capture, no VIN in
  it), under the extra's license.
- The PID and CVT findings shared with the community sources we used.

## 7. Security, Privacy and Safety

### 7.1 Threat model (p4n4 Car)

| Threat | Mitigation |
|---|---|
| Device stolen with the car, or removed | LUKS2 on `/data` (§3.6); erase action; no secrets on the SD card's boot partition |
| Someone near the car joins the hotspot | WPA2/WPA3 with a generated passphrase; services bound to the hotspot interface only; dashboard sign-in through p4n4-api; the MQTT and InfluxDB ports not exposed. This depends on the 0.2.1 security release (G13) |
| A prompt injection in data (a code description, a log line, a payload) steers the agent | Tool outputs from data sources are marked untrusted (§7.4); X tools need human approval regardless |
| The agent damages the car | Tier N for ECU writes and raw transmit; read-only extras; `obd_dtc_clear` gated (§7.3) |
| Location history leaks through sync | Sync only to the owner's home p4n4 over TLS; GNSS off by default; retention |

### 7.2 Approvals for X tools

The MCP host's tool approval is the first gate, but the harness doesn't rely on it alone. An
X tool called with `dry_run: false` returns a confirmation token bound to the exact plan, for
example "recreate `p4n4-ei-runner` with model `cube-anomaly-v3.onnx`". The call only runs
when it's repeated with that token within 5 minutes. A human sees the plan once in the host
and once as the token prompt (§12 Q2).

### 7.3 Clearing trouble codes

Clearing codes erases freeze frames and resets readiness monitors, which can fail an
emissions inspection and hide a real fault. `obd_dtc_clear` therefore:
- requires the engine off (rpm 0, ignition on);
- saves the codes and freeze frames to `events` first;
- shows them to the user;
- needs two approvals.

### 7.4 Untrusted output

The following tool results are wrapped as data, with a marker that the skills tell the agent
to respect:
- `stack_logs`, `topic_sample`, `nodered_flows_get`, telemetry strings;
- anything from an extra's device.

The harness also removes known secret patterns from them before returning anything.

## 8. Gaps

What p4n4 is missing for the harness and for p4n4 Car. G3–G9 keep their 0.1 meanings, as
cited by [Known Issues](../project/known-issues.md) and the code review. Status is as of
2026-10-03 (v0.2.0 release prep).

| # | Gap | Repos | Needed by | Status |
|---|---|---|---|---|
| G1 | No registry, scaffold or validation for extras; layers are fixed in `layers.py` | lib, cli | §5.5, `extra_*` tools | Open |
| G2 | `p4n4 add` / `remove` are stubs | cli, lib | `layer_add`, `extra_add` | Open |
| G3 | `p4n4 init --layer all` skipped edge (KI-3) | cli, lib | `template_apply`, edge tools | **Fixed** (CLI 0.2.0) |
| G4 | The edge runner had no inference API (KI-4) | edge | `edge_infer`, `edge_model_list` | **Fixed**: `GET /api/v1/info`, `POST /api/v1/infer` |
| G5 | Topic and data contract drifting between specs, flows and n8n triggers | iot, ai, docs | `nodered_flows_apply` checks, §3.5 | **Partly**: specs §8.6 adopted (KI-1, KI-2 fixed); an end-to-end data-contract test is still open (review §8.2) |
| G6 | Docs disagree with code on manifest fields and env key names | docs | Agents reading the docs | **Partly**: specs 0.1-draft3 aligned; no check keeps them aligned |
| G7 | No end-to-end test that starts the stacks (review CI-3) | all | §10 evaluation | Open |
| G8 | The Node-RED editor and Admin API had no login (KI-6) | iot | `nodered_flows_*` | **Fixed**: `adminAuth`; the harness takes a token from `POST /auth/token` |
| G9 | Deploying from the Node-RED editor failed (KI-7) | iot | `nodered_flows_apply` | **Fixed**: flows in `config/node-red/flows/` |
| G10 | p4n4-api has user accounts and roles but no scoped tokens for agents, and no confirmation-token flow | api | §4.1, §7.2 | Open |
| G11 | The MQTT bridge only pulls from an external broker; there's no push to a home broker | iot, cli | §3.7 | Open |
| G12 | No encryption at rest, retention UI or erase action | iot, dashboard, hw | §3.6 | Open |
| G13 | Services listen on all interfaces, MQTT is anonymous, secrets have fallback defaults | iot, ai | §7.1 (car hotspot) | Open: scheduled for 0.2.1 |
| G14 | Storage rules cover `sensors/` and `inference/` only; `alerts/` events (DTCs) aren't stored | iot | §3.5, C2 | Open |
| G15 | No Vehicle view in the dashboard; no way for an extra to ship a tab | dashboard | §3.8 | Open |
| G16 | No path to collect a dataset and train an edge model from stored telemetry (`p4n4 ei` is a stub) | edge, cli | C6, `p4n4-edge-model` | Open |

## 9. Phases

| Phase | Scope | Exit criteria |
|---|---|---|
| **H0** | Harness skeleton: MCP server, R tools, audit ids, the `p4n4-debug` and `p4n4-new-project` skills (read-only parts) | An agent describes, validates and queries a running project; S-OBD-5 read-only part |
| **H1** | W tools with dry runs and backups; `template_apply`, `env_set`, `grafana_dashboard_apply` | An agent scaffolds and configures a project from a brief; every change has a diff and an audit entry |
| **H2** | `p4n4-extras` repository, contract checks, the obd2 extra (incubating); G1, G2; spike S-OBD-0 to S-OBD-4 | p4n4 Car logs a real drive into encrypted storage |
| **H3** | X tools with confirmation tokens (G10); `nodered_flows_apply`, `edge_deploy_model`; the `vehicle-obd2` template in p4n4-templates; Vehicle view (G15); DTC explanations | A week of driving with explanations on the phone, plus an anomaly model deployed with approval |
| **H4** | Home sync (G11), retention and erase (G12), modbus and canbus extras (incubating) | History at home; a second protocol added by someone outside the core team |
| **H5** | Voice (wake word, local speech-to-text and text-to-speech), only after H3 is stable | Hands-free questions answered within a few seconds [estimate] |

The 0.2.1 security release (G13) comes before anyone runs p4n4 Car with the hotspot on.

## 10. Evaluation

The harness is judged by what agents do with it, not by its tool count.

- **Scenarios:** a set of written briefs, each with a checker:
  - "p4n4 Car against the simulator";
  - "greenhouse telemetry from a template";
  - "add a Modbus meter";
  - "debug: no data in Grafana" (seeded fault);
  - "deploy a new edge model".
- **Environment:** p4n4-emu with the `rpi5` profile, the extras' simulators and fresh volumes
  for every run. It's the same end-to-end harness the platform needs anyway (G7).
- **Checks per run:** `project_validate` passes; the scenario's smoke test passes; the
  expected data reaches InfluxDB; the audit log matches the actions; zero tier violations
  (no X without a token, no N attempts); no secrets in the transcript.
- **Metrics:** success rate, tool calls and wall time per scenario, the number of human
  approvals asked, and the rollbacks needed.
- **Regression:** the scenarios run in the harness's CI against the latest stacks' `main` and
  against the last release.

## 11. Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| The CVT temperature can't be read without dealer tooling | Medium [estimate] | Medium: C1 loses a gauge, C6 loses its best signal | Documented "not possible" is an acceptable spike result; the CAN HAT option as a fallback |
| python-OBD's GPL license doesn't fit the extras repo | Medium | Low | The per-extra license (§5.3), or option B (our own client) |
| The Pi overheats or the SSD corrupts in the car | Medium | High | S-OBD-3/4 before H2 exits; off while parked; LUKS plus filesystem checks on boot |
| Battery drain strands the owner | Low with ACC sense | High | Power from ACC through the HAT, a timeout, a measured drain |
| An agent makes destructive changes | Low | High | Tiers, dry runs, confirmation tokens, audit, backups |
| Extras rot without maintainers | High over time | Medium | Maturity levels, CODEOWNERS, demotion after 6 months |
| Scope creep into a full scan tool | Medium | Medium | Non-goals (§2); OBD-II mode 01/03/07/09/0A plus specific reads only |

## 12. Open Questions

1. **Where does the harness run in production?** On the workstation only (operations over
   SSH or HTTPS to the device), or also on the device for an on-board agent? The device
   option makes the MCP endpoint part of the attack surface (§7.1).
2. **Approval UX.** Are confirmation tokens (§7.2) enough, or should X tools also need an
   approval in the dashboard (a second device), which is useful when the agent runs
   unattended?
3. **Topic suffix as a tag or as the measurement** *(resolved)*. Specs §8.6 settled it: the
   suffix becomes the `sensor` tag on the `sensor_data` measurement, and the reading goes in
   `value` (KI-2's fix). §3.5 follows it.
4. **Edge inference API shape** *(resolved)*. The runner serves `GET /api/v1/info` and
   `POST /api/v1/infer` with a JSON feature vector, and answers 422 when the model rejects
   the input (KI-4's fix, specs §8.5). `edge_infer` uses it as is.
5. **Licenses in p4n4-extras.** Allow GPL-2.0 extras (python-OBD) with clear labelling, or
   require MIT/Apache-2.0 and write our own OBD client (option B)? What goes on the
   allowlist (§5.3)?
6. **LUKS key strategy** for unattended boot: a dongle, an OTP-sealed key or a phone unlock
   (§3.6)? Maybe the dongle by default, with the phone unlock as an option.
7. **Manifest version.** Do extras in `.p4n4.json` need `schema_version: 2`, and how do 0.2.x
   CLIs treat a project they don't fully understand?
8. **Grafana in the car.** Keep it (familiar, already provisioned) or rely on the dashboard
   only to save memory on the Pi?
9. **Extras images.** Who builds and signs community extras' images? GHCR under `raisga` for
   `official` only, with `community` built locally?

---

## Appendix A. Example `extra.yaml`

```yaml
# ==============================================================================
# Extra metadata, validated against schema/extra.schema.json
# ==============================================================================
schema_version: 1
name: obd2
version: 0.1.0
maturity: incubating
kind: ingest
title: OBD-II → MQTT
description: >-
  Reads a vehicle's OBD-II port through an ELM327-compatible adapter (STN
  recommended) and publishes readings, trouble codes and adapter status over MQTT.
license: GPL-2.0-only        # if it imports python-OBD; MIT with our own client (§6.3)
maintainers:
  - jraleman
safety: read-only            # transmits diagnostic requests only (modes 01, 03, 07, 09, 0A)

hardware:
  devices:
    - env: OBD_DEVICE         # e.g. /dev/serial/by-id/usb-ScanTool.net_OBDLink_SX_…
      description: USB OBD-II adapter
  simulator: simulator/       # ELM327-emulator scenario; used in CI and by p4n4-emu

services:
  - name: obd2
    profile: obd2
    health: http://obd2:8090/health

data:
  device_id_env: OBD_DEVICE_ID          # default: "car"
  publishes:
    - pattern: sensors/<device-id>/<measurement>
      payload: '{"value": 2150, "unit": "rpm"}'
    - pattern: sensors/<device-id>/raw
      payload: '{"values": [rpm, load, coolant, stft, ltft, maf, speed]}'
    - pattern: alerts/<device-id>/dtc
      payload: '{"codes": ["P0171"], "mil": true, "freeze_frame": {}}'
    - pattern: status/<device-id>/online
      retained: true
  subscribes: []

env:
  required: [OBD_DEVICE, OBD_DEVICE_ID, MQTT_HOST]
  optional: [OBD_POLL_FAST_HZ, OBD_POLL_SLOW_HZ, OBD_PIDS]

api:                          # used by the harness's obd_* tools
  - GET /health
  - GET /api/v1/probe         # protocol, VIN (hashed), supported PIDs, adapter
  - GET /api/v1/dtc           # stored, pending, permanent codes and freeze frame
  - POST /api/v1/dtc/clear    # X tier; refuses unless rpm == 0

smoke_test: tests/smoke.sh
```

## Appendix B. Example Tool Definition

```json
{
  "name": "edge_deploy_model",
  "description": "Deploy a model file to the edge runner. Recreates the runner container, because a restart does not re-read .env (KI-5). Tier X: call with dry_run=true first, then with the returned confirmation token.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "model_path": {"type": "string", "description": "Path in the project's edge models directory (.onnx or .eim)"},
      "backend": {"type": "string", "enum": ["auto", "onnx", "eim"], "default": "auto"},
      "dry_run": {"type": "boolean", "default": true},
      "confirmation": {"type": "string", "description": "Token returned by the dry run; required when dry_run is false"}
    },
    "required": ["model_path"],
    "additionalProperties": false
  },
  "outputSchema": {
    "type": "object",
    "properties": {
      "plan": {"type": "array", "items": {"type": "string"}},
      "confirmation": {"type": "string"},
      "expires_at": {"type": "string", "format": "date-time"},
      "job_id": {"type": "string"},
      "previous_model": {"type": "string"},
      "audit_id": {"type": "string"}
    }
  },
  "annotations": {"tier": "X", "destructiveHint": false, "idempotentHint": true}
}
```

## Appendix C. Sources

Desk research for §6.2, gathered 2026-10-03. Community sources are leads, not facts; the
spike confirms them on the car.

- OBD-II over CAN, mandatory for US vehicles from 2008:
  [SparkFun: Getting Started with OBD-II](https://learn.sparkfun.com/tutorials/getting-started-with-obd-ii/all),
  [Simma Software: ISO 15765-4](https://simmasoftware.com/iso-15765-4-code-the-complete-guide-to-obd-II-over-can/),
  [CSS Electronics: OBD2 explained](https://www.csselectronics.com/pages/obd2-explained-simple-intro).
- 2011 Cube transmission (JF011E / RE0F10A):
  [Berkeley Standard: JF011E](https://berkeleystandard.com/product/jf011e-transmission/),
  [GoPNH: CVT JF011E RE0F10A](https://gopnh.com/Nissan-Transmissions/CVT-JF011E-RE0F10A/6768).
- Transmission temperature identifiers are manufacturer-specific:
  [OBDLink support: transmission temperature gauge](https://support.obdlink.com/support/solutions/articles/43000708843-display-transmission-temperature-gauge).
- Nissan CVT data over ELM327, supported models and adapter caveats: [CVTz50](https://cvtz50.info/en/).
- STN vs ELM327 and clone quality:
  [ScanTool FAQs](https://www.scantool.net/faqs/),
  [FORScan: choosing ELM327-compatible adapters](https://forscan.org/forum/viewtopic.php?t=6142),
  [Carvitas: original ELM327 interfaces](https://carvitas.com/blog-and-news/original-elm327-obd2-interfaces).
- python-OBD (0.7.3, GPL-2.0-only): [PyPI `obd`](https://pypi.org/project/obd/),
  [docs](https://python-obd.readthedocs.io/), [repository](https://github.com/brendan-w/python-OBD).
- CAN and Modbus libraries, licenses from PyPI: [python-can](https://pypi.org/project/python-can/) 4.6.1 (LGPL-3.0-only),
  [can-isotp](https://pypi.org/project/can-isotp/) 2.0.7, [udsoncan](https://pypi.org/project/udsoncan/) 1.26.1,
  [cantools](https://pypi.org/project/cantools/) 44.1.0 (MIT), [pymodbus](https://pypi.org/project/pymodbus/) 3.15.0 (BSD-3-Clause).
- ELM327-emulator (4.0.0, CC-BY-NC-SA-4.0): [PyPI](https://pypi.org/project/ELM327-emulator/),
  [repository](https://github.com/Ircama/ELM327-emulator).
- In-car power for the Pi 5:
  [CarPiHAT PRO 5](https://thepihut.com/products/carpihat-pro-5-car-interface-dac-for-raspberry-pi-5),
  [RPiCarPowerHAT](https://github.com/Compizfox/RPiCarPowerHAT),
  [Raspberry Pi forums: powering a Pi 5 in a vehicle](https://forums.raspberrypi.com/viewtopic.php?t=361432).
- OBD port power and battery drain:
  [Wikipedia: Data link connector](https://en.wikipedia.org/wiki/Data_link_connector),
  [Engineer Fix: does an OBD2 device drain your battery?](https://engineerfix.com/does-an-obd2-device-drain-your-battery/).
- Pi 5 temperature range and RTC:
  [Raspberry Pi 5 product brief](https://datasheets.raspberrypi.com/rpi5/raspberry-pi-5-product-brief.pdf),
  [Raspberry Pi forums: Pi 5 operating temperature](https://forums.raspberrypi.com/viewtopic.php?t=392274).
