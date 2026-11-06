# P4N4: TODO Alignment Review and Improvement Ideas

**Date:** 2026-11-04  
**Scope:** Current local files, static code review, and improvement proposals.  
**Change made:** This document only. No application code or TODO checkboxes changed.

## Baselines and interpretation

There is no root `TODO.md`. The primary baseline is `docs\TODO.md`, a historical
code-review register dated 2026-11-02, not a list of features that must already
exist. Its original findings and line numbers describe an earlier snapshot;
resolution notes describe subsequent fixes, including local changes.

| Source | How to use it |
|---|---|
| `docs\TODO.md:14-179` | Review register: six checked resolutions, with the remaining findings still open in the document. Unchecked does not prove the current implementation is broken. |
| `docs\TODO.md:330-355,609-620` | M3 and L7 closure notes: device authentication and bounded feedback storage. The anonymous, unpublished Docker-internal MQTT listener is an intentional compatibility policy. |
| `api\TODO.md` and `dashboard\TODO.md` | Component status and acceptance gaps; dated statements need reconciliation rather than blind reuse. |
| `docs\decisions\specs.md` | Feature roadmap; distinguish shipped, partial, superseded, and future criteria. |
| `RUST_REWRITE_PLAN.md:90-98,1607-1718` | Accepted 1.0 direction and phase gates, not proof that the replacement runtime exists. |
| `docs\decisions\spikes\README.md:3-18` | P0 progress notes: several spikes are partial or not started. Desk research is not a completed runtime qualification. |

The current documented platform uses Python library/CLI/API components, Flutter,
and cooperating Compose stacks. The Rust plan retains Flutter, makes a clean
break from v0.4 projects and clients, uses one Compose project, and keeps the
one-running-project-per-Docker-host restriction. These decisions are constraints
for the ideas below, not invitations to introduce a migration framework or a
second UI rewrite.

No git operations, network lookups, service starts, builds, or application tests
were performed for this review. Repository provenance, published artifact
contents, enabled remote CI, and live behavior have not been independently
verified. Historical test results in other documents are not results of this
review.

## Current-file TODO alignment

The code-review pass covered the six checked resolutions and the immediate
implementation/configuration/test paths for M13, M14, M17, M19, L23, and L26.
This is a bounded alignment review, not a re-audit or closure of all 71 historical
findings. Source references below describe the current local files.

All six checked resolutions have corresponding implementation changes.
**Implemented** here means static alignment, not a passing test run or a verified
release/commit state.

| Audit ID | Static alignment | Current-file evidence |
|---|---|---|
| H1 | Implemented: go2rtc restricts its API paths and drops capabilities. | `tools\templates\projects\mqtt-influx-grafana-ollama-go2rtc\extras\config\go2rtc\video.sh:61-64`; `tools\templates\projects\mqtt-influx-grafana-ollama-go2rtc\extras\docker-compose.yml:24-27`. |
| H2 | Implemented: n8n rejects the public signing key, generates/persists a random key when empty, checks length, and compares acknowledgement tokens in constant time. | `tools\templates\projects\mqtt-influx-grafana-n8n\ai\config\n8n\entrypoint.sh:40-47`; `tools\templates\projects\mqtt-influx-grafana-n8n\ai\config\n8n\workflows\acknowledge.json:74,129`. |
| H3 | Implemented: alert timestamps are bounded; recovery isolates incidents that cannot resume. | `stacks\ai\agent\src\request-id.ts:50-78`; `stacks\ai\agent\src\investigator.ts:313-334`. |
| M3 | Implemented under the documented policy: new CLI projects get publishing/read-only credentials; the device port maps to authenticated container 1884, while internal 1883 stays anonymous and unpublished. Template login support is opt-in; forced-on valve runtime/recovery protections are present. | `cli\p4n4\commands\init.py:383-391`; `stacks\iot\config\mosquitto\bridge.sh:146-180`; `stacks\iot\docker-compose.yml:36`; `tools\templates\projects\mqtt-nodered-influx-grafana\config\node-red\flows\flows.json:236-266,343-350`. |
| M18 | Implemented: both affected template agents default to loopback publication. | `tools\templates\projects\mqtt-influx-grafana-ollama\ai\docker-compose.yml:19`; `tools\templates\projects\mqtt-influx-grafana-ollama-go2rtc\ai\docker-compose.yml:19`. |
| L7 | Implemented: feedback persistence checks byte/file/set limits and file type, reports errors, and the broker caps packets. | `stacks\iot\config\node-red\feedback.js:6-28`; `stacks\iot\config\node-red\flows\flows.json:724`; `stacks\iot\config\mosquitto\bridge.sh:170`. |

The current Rust workspace contains only `xtask` (`Cargo.toml:3-5`), whose exposed
commands capture/check golden fixtures (`xtask\src\main.rs:17-24`). It does not
establish replacement-runtime parity.

Explicitly open features remain open: API-layer registration/bootstrap
(`api\TODO.md:120`) and the general add-command stub
(`cli\p4n4\commands\add.py:13-18`). They are roadmap gaps, not newly introduced
regressions.

## Code-review findings

Six current correctness/integration issues corroborate still-open audit entries.
None is claimed to have been introduced by the checked fixes. **P2** is normal
code-review priority, not a revision of the audit's historical security severity.
Triggers and consequences were traced statically; executable confirmation remains
outstanding.

### R1 - P2: ONNX classification interprets the label output as probabilities

**Audit:** M19. **Location:** `stacks\ai\runner\runner.py:500-503`.

A numeric-label sklearn export returns a label tensor followed by probabilities.
The runner always post-processes `outputs[0]`, so a first output such as `[1]`
becomes one score mapped to configured class zero with confidence `1.0`, even
when another class won. The mapping is at
`stacks\ai\runner\runner.py:467-477`; existing ONNX cases only supply one output
(`stacks\ai\tests\test_runner.py:145-187`).

**Recommendation and check:** Select and validate the probability output using
metadata/shape, supporting ZipMap and float class tensors. Add a label-plus-
probability regression and a small real exported classifier with a known answer.

### R2 - P2: API lifecycle execution omits the CLI's required handling

**Audit:** M14. **Location:** `api\p4n4_api\jobs.py:116-125,160-168`.

The API builds its own Compose commands. Stop omits the all-profile handling used
by the library (`lib\p4n4_lib\compose.py:82-90`). Startup omits the CLI's
empty-layer filtering and network preparation
(`cli\p4n4\commands\lifecycle.py:104-119,166-168`). Consequently, Stop can leave
previously enabled services running, a host-run API cannot start an AI stack
without the shared network, and Start-all can fail on the empty extras skeleton
before reaching later targets (`stacks\extras\docker-compose.yml:13-17`).

**Recommendation and check:** Share version-aware operation planning/preflight
with the CLI. Test a disabled-profile service, an empty extras layer, and a fresh
host API startup without `p4n4-net`. Preserve non-destructive Stop behavior.

### R3 - P2: The default containerized dashboard/API pairing is unreachable on native Linux

**Audit:** M17. **Location:** `dashboard\docker-compose.yml:16,26-27`.

The dashboard targets the host-gateway address, but the API publishes its port on
host loopback by default (`api\docker-compose.yml:17`). On native Linux that
gateway connection cannot reach the loopback binding, so proxied API calls and
sign-in fail. Both containers already join the shared Docker network. nginx uses
the configured upstream directly
(`dashboard\docker\nginx\default.conf.template:98-104`).

**Recommendation and check:** Default the container pairing to
`http://p4n4-api:8000`, including `dashboard\.env.example:20`, while retaining an
explicit override for host development. Exercise sign-in through the shipped
dashboard on a native-Linux engine; Docker Desktop alone does not validate this
case. This networking consequence was not reproduced during the review.

### R4 - P2: CORS rejects implemented PUT operations

**Audit:** L26. **Location:** `api\p4n4_api\main.py:113-118`.

PUT is missing from the CORS method allowlist. An otherwise allowed cross-origin
browser therefore cannot save views, view settings, banners, or assistant
configuration, which use PUT
(`api\p4n4_api\routes\dashboard.py:130,162,194`;
`api\p4n4_api\routes\agents.py:240`). The existing preflight test requests only GET
(`api\tests\test_api.py:527-530`). This qualifies the completed CORS claim in
`api\TODO.md:25`; the same-origin dashboard path is not affected.

**Recommendation and check:** Allow PUT and test its preflight for an allowed
origin, rejection for a disallowed origin, and the authenticated update request.

### R5 - P2: Foreground startup blocks subsequent layers

**Audit:** L23. **Location:** `cli\p4n4\commands\lifecycle.py:156-168`.

For a multi-layer project, `p4n4 up --no-detach` waits on the first running stack.
The sequential loop cannot start AI/dashboard layers while that foreground
process remains attached. The library uses blocking subprocess execution
(`lib\p4n4_lib\compose.py:60-78`).

**Recommendation and check:** Start all targets before following combined logs,
or explicitly reject unsupported multi-target foreground mode. Test with a first
target that remains running, and preserve the intended Ctrl-C behavior.

### R6 - P2: Validation accepts inconsistent shared InfluxDB configuration

**Audit:** M13. **Location:** `lib\p4n4_lib\validate.py:87-102`.

Validation checks required keys independently per layer but never compares shared
values. Changing only the AI token or organization in an otherwise complete
IoT/AI project still validates. IoT initializes the server with its own settings
(`stacks\iot\docker-compose.yml:92-95`), while AI services use theirs
(`stacks\ai\docker-compose.yml:229-231,302-305`), leading to authentication or
query/write failures.

**Recommendation and check:** Compare shared settings for layers targeting the
same local server, distinguishing intentionally external endpoints. Report layer
and key names without secret values. Test divergent token/org/bucket values and
run consistency validation before startup.

## Prioritized improvement backlog

These are proposals, not implemented features or new review defects. **First**
means documentation or narrow remediation to evaluate against the Python
critical-fix freeze. **Foundation** means an acceptance gate for the existing Rust
phases. **Later** means optional work that must not block the reliable core.

| ID | Priority | Improvement | Smallest useful outcome |
|---|---|---|---|
| B1 | First | Evidence-backed TODO closure | Each resolved finding has a current implementation reference and a reproducible regression check. |
| B2 | First / Foundation | One lifecycle contract | CLI and API produce equivalent Compose operations for the same project and enabled profiles. |
| B3 | First / Foundation | Validate cross-component invariants | Preflight names configuration mismatches and stale resources before starting anything. |
| B4 | First / Foundation | Reproducible release sets | An installation resolves a tested combination of library, assets, API, and dashboard. |
| B5 | Foundation | One strict vertical integration smoke | A fresh existing template proves the real sensor-to-dashboard path without hidden setup. |
| B6 | Foundation | Observable, bounded data paths | Operators can distinguish missing data, stale data, overload, and an unhealthy service. |
| B7 | First / Foundation | Qualify inference output contracts | Supported models return the correct label/probability mapping; unsupported shapes fail explicitly. |
| B8 | Foundation | Local, redacted diagnostics | A support report explains failures without collecting credentials or sending data away. |
| B9 | Later | Small, dependency-aware product extensions | Single-site history, safe approval UX, and reproducible demos grow only after core gates pass. |

### B1. Make TODO closure an evidence receipt

**Why:** `docs\TODO.md` preserves historical evidence, while `api\TODO.md:17,40`
still describes the Devices screen and dashboard request IDs as missing.
`dashboard\TODO.md:31,35,84` says they are built. This is documented status drift,
not evidence of a missing dashboard implementation.

**First slice:** Keep the historical audit intact. Add a short resolution receipt
per item: affected component, current source location, behavior changed,
regression command and result, validation platform, and residual limitations.
Use "implemented, not revalidated" when only code has been inspected. Reconcile
cross-component blockers in one pass rather than creating another competing
roadmap.

**Acceptance:** A contributor can explain why each checked item is closed without
reconstructing an old review. No status claims depend only on an outdated date,
an unchecked box, or a design promise.

### B2. Share lifecycle behavior, not just argument strings

**Why:** The audit identifies API/CLI lifecycle drift (M14), profile and
foreground-start problems (L23), and the API roadmap already requests shared
Compose builders in milestone M4.

**First slice:** Centralize the operation plan in the existing library for any
approved v0.4 regression fix. Both front ends should consume the same stack order,
enabled-service selection, network preparation, emulator overlays, and failure
semantics. In Rust, put that behavior behind the already planned `ComposeDriver`,
not a new orchestration framework.

**Acceptance:** Focused tests cover empty profiles, explicit service selection,
start/stop order, an unavailable daemon, cancellation, and one intermediate
failure. Foreground logging must not prevent later layers from starting. CLI and
API must neither execute different plans nor report partial work as success.

### B3. Validate relationships before mutating state

**Why:** M12, M13, M21, and L25 concern volume identity, shared InfluxDB settings,
network ownership, and interrupted scaffolding. Checking each file in isolation
cannot establish that a deployment is coherent.

**First slice:** Extend existing validation/diagnostics to compare shared settings
across active v0.4 layers, check reachable endpoints from the actual caller, and
identify existing containers and volumes belonging to another or older project.
Report variable names and affected layers, never their secret values. For 1.0,
implement the planned root `.env`, project ownership checks, atomic writes, and
repair journal rather than carrying forward per-layer synchronization complexity.

**Acceptance:** A mismatched token/org/bucket, wrong API upstream, stale resource,
or incomplete project gets a specific error before startup. Failure leaves user
files unchanged. The existing one-project-per-host constraint is explained; the
check does not promise concurrent deployments.

### B4. Make the release set reproducible

**Why:** M6-M10 and L33 describe unlocked builds, moving scaffold sources,
dependency compatibility, CI gaps, and template drift. The rewrite's K15 also
recognizes that future dashboard defaults can break the frozen Python CLI.

**First slice:** Verify bounded Python dependencies and locked image builds; test
the published dependency combination in addition to any development-head job.
Record component versions, source/content digests, image digests, template
identity, and schema versions together, without environment values. Preserve the
planned final v0.4 compatibility release before Dashboard 2.0 becomes the default.

**Acceptance:** Reinstalling a release produces the same non-secret assets and a
compatible component set. An incompatible major version fails before startup.
The Rust release embeds qualified assets, as already planned; it does not fetch
mutable core configuration at runtime.

### B5. Turn one existing template into a strict vertical smoke

**Why:** The dashboard TODO says several screens were tested only with mocked
responses. The audit's L15 resolution discussion records an end-to-end check
that needed archive directories to be created outside the shipped bootstrap.

**First slice:** Use an existing greenhouse template with simulated devices,
an isolated project, generated test credentials, and disposable test-owned
resources. Start from its documented setup, publish one sample, verify exactly
one stored point, and read it through the API/dashboard. Exercise a device login,
read-only MQTT permissions, one log stream, and shutdown. Test simulated control
safety separately; never connect the smoke to physical actuators.

**Acceptance:** The fixture works from an empty project without undocumented
directory creation or production resources. It fails on a dropped or duplicate
point, broken proxy, unauthorized write, or bootstrap error. Expand to all
templates and minimum/latest Compose versions through the existing P1-P3 gates,
not through an unbounded new test matrix.

**Boundary:** Demo recordings intentionally fail soft
(`docs\decisions\recording.md`). A recording is not evidence that this strict
smoke passed.

### B6. Expose boundedness and data freshness

**Why:** L1, L7, L18, and M20 concern admission control, disk growth, blocked
servers, and log retention. The dashboard already warns about stale readings,
but a healthy container does not establish that telemetry is flowing.

**First slice:** Carry queue capacity, dropped-message counts, last successful
ingest/inference time, storage-full errors, and dependency reachability into the
planned runtime metrics. Bound queues, feedback sets, log storage, and generation
concurrency. Preserve L7's explicit storage-full error and existing examples.

**Acceptance:** A controlled simulator overload has bounded memory/disk growth,
visible drop or rejection counters, and recovery after load stops. Operators can
tell "no recent samples" from "service down." Do not claim lossless delivery or
exactly-once telemetry where the design specifies best effort.

### B7. Treat model outputs as a qualified contract

**Why:** M19 identifies the ONNX multi-output classifier issue. P0 S3 explicitly
requires qualification of ZipMap/TreeEnsemble and inference backends by target.

**First slice:** Add small known-answer models for tensor probabilities,
label-plus-probability outputs, and ZipMap. Check feature ordering/count, labels,
probability interpretation, and unsupported-output diagnostics before accepting
a model. Reuse those examples for the Rust backend qualification.

**Acceptance:** The same input maps to the expected label and scores on every
declared backend/target within an explicitly measured tolerance. A malformed or
unsupported result returns a clear error, never a plausible first-class result.
Benchmark on qualified hardware before publishing latency or memory claims.

### B8. Make diagnostics useful without exporting secrets

**Why:** Request IDs already exist in the dashboard; they should not be proposed
as a new feature. The remaining opportunity is connecting a reported failure to
configuration, service status, data freshness, and operation outcomes.

**First slice:** Extend the planned `doctor` output and existing Diagnostics view
with a local, opt-in, redacted report: versions, enabled profiles, service/data
health, recent operation error codes, and request IDs. Redact at collection time
and list what is included. Keep the report on the user's machine by default.

**Acceptance:** A report distinguishes Docker unavailable, wrong upstream,
authentication failure, stale telemetry, and unsupported hardware. Tests prove
that passwords, tokens, private camera URLs, raw sensor payloads, and user
identifiers are excluded. Do not add automatic uploads or a remote dependency.

### B9. Grow the product without reopening core scope

These extend existing roadmaps rather than adding a parallel product plan.

| Opportunity | Small first deliverable | Dependency and acceptance |
|---|---|---|
| Unattended single-site history | Persist service transitions with a retention limit; show gaps rather than inventing uptime while the server was offline. | Planned server aggregation in P3; history continues while no dashboard is open. Keep fleet federation after 1.0. |
| Human-approved agent actions | After 1.0, a read-only approval preview showing proposed action, evidence, expiry, and operator decision. | Existing Phase 2 design, explicit authorization, receiver-side durable deduplication, and crash tests precede any effect. No autonomous actuator control as an MVP. |
| Reproducible demos and onboarding | After 1.0, implement the existing declarative recording proposal using isolated demo data. | Keep recording separate from strict tests. Recordings must not reveal real accounts, secrets, or client data. |

## Validation status and follow-up checks

Only the new document's encoding, structure, and cited-file existence were
checked. No application test result or runtime behavior is claimed. A changeset
comparison was intentionally unavailable because all git operations were
forbidden.

The following are existing component commands or Windows-path equivalents,
listed for future validation, **not executed in this task**:

| Working directory | Command | Result in this review |
|---|---|---|
| `api` | `pytest --cov=p4n4_api --cov-report=term-missing` | Not run. Prioritize PUT preflights and lifecycle regression fixtures. |
| `cli` | `pytest tests\ -v` | Not run. Prioritize multi-layer foreground startup and lifecycle equivalence. |
| `stacks\ai` | `pytest tests\ -v` | Not run. Prioritize multi-output ONNX and real-model qualification. |
| `stacks\ai\agent` | `npm test` | Not run. Revalidate timestamp/recovery behavior. |
| `stacks\ai\agent` | `npm run lint` | Not run. No TypeScript changes were made. |
| `stacks\ai\agent` | `npm run build` | Not run. No TypeScript changes were made. |
| `dashboard` | `flutter test` | Not run. No Dart changes were made. |
| `dashboard` | `flutter analyze` | Not run. No Dart changes were made. |

For M3/L7 revalidation, the broker integration suite includes template cases only
when `P4N4_TEMPLATE_ROOT` is set to the local `tools\templates` checkout
(`stacks\iot\tests\mqtt.test.cjs:85-96`). It starts containers and writes fixtures;
run it only with isolated, test-owned resources. Include the bounded feedback
tests and real retained-message/permission checks rather than treating client
exit codes as sufficient.

Native-Linux pairing, Compose profile/network behavior, real ONNX exports,
template runtime protections, and the historical passing-test counts remain
unverified. No source control metadata or remote CI state was inspected.

## Suggested order and boundaries

1. Resolve verified current correctness gaps and reconcile the audit's closure
   evidence. Evaluate any Python changes against its critical-fix freeze.
2. Finish P0's hands-on Compose, proxy, inference, Windows, and Docker Desktop
   qualification before promoting design assumptions to supported behavior.
3. Use one vertical smoke and shared contracts to gate P1-P3; add hardware and
   durability checks at their existing phase boundaries.
4. Consider B9 only after the reliable core. Do not add Kubernetes, federation,
   another message bus, a Flutter replacement, or v0.4-to-v1 migration machinery
   to this work.

Success means fewer integration surprises and evidence-backed claims, not a
larger feature count. The brainstorm itself does not authorize or implement any
of the proposed changes.
