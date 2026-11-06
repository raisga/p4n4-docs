# P4N4 Technology Use Cases

**Date:** 2026-11-05  
**Purpose:** Practical project ideas using P4N4's IoT, local AI, edge inference,
dashboard, and emulation technologies.

These are candidate applications, not claims of production readiness. Some have
existing reference templates; others need domain-specific adapters, models,
workflows, or UI work. No deployment or application tests were performed while
writing this document. Repository paths are relative to the P4N4 root.

## 1. What each technology contributes

| Technology | Useful role | When not to add it |
|---|---|---|
| Docker Compose | Package a repeatable local deployment and enable only the required services. | It is not a substitute for hardware qualification, backup, or network security. |
| Mosquitto / MQTT | Connect devices, ingestion services, rules, and inference through named messages. | Do not assume every message is stored, delivered exactly once, or authorized without configuration. |
| Node-RED | Normalize readings, apply explicit thresholds/hysteresis, and connect systems. | Exact safety limits should not be replaced by model-generated decisions. |
| InfluxDB + Grafana | Keep time-series evidence and display trends, thresholds, and operational history. | A graph does not establish sensor calibration or regulatory compliance. |
| Ollama | Explain queried data, answer operator questions, and draft reports locally. | Skip the LLM when a rule, chart, or calculated statistic answers the question. |
| Letta | Maintain an equipment register and contextual incident/maintenance memory. | Model memory must not replace authoritative measurements or approved work records. |
| Pi Durable investigator | Resume a bounded, read-only investigation after a process restart. | Its current investigator is not an autonomous action or approval system. |
| System 1 / Laya | Suggest a choice from a fixed menu, such as an incident route or operator intent. | Do not use it to determine safety, severity, permissions, or exact thresholds. |
| Edge Impulse / ONNX runner | Classify numeric feature vectors locally with a qualified model. | Raw camera/audio pipelines need preprocessing and compatible model inputs; they are not generic plug-ins. |
| go2rtc + camera ingestion | Provide authorized camera streams and a source for custom visual analytics. | Avoid identity tracking or keeping footage/plates when aggregate counts suffice. |
| API + Flutter dashboard | Give different users an operational view, histories, chat, and white-label presentation. | Cross-site analytics and unattended incident history are not automatically a complete fleet backend. |
| Hardware tools + emulator | Connect real devices and rehearse software under target resource limits. | Workstation quotas/QEMU are not a physical or GPU-equivalent digital twin. |

**Default pattern:** rules decide; System 1 routes; generative AI explains.
Add AI only after telemetry and its deterministic calculations are trustworthy.

Local inference can keep application data on the deployment, but initial image/
model downloads and configured email or external integrations can need a network.
"Self-hosted" does not automatically mean fully offline or internet-safe.

## 2. Candidate portfolio

**Template:** a documented reference starting point exists; it still needs setup
and validation. **Adaptation:** existing parts need a new domain integration.
**Incubating:** the component explicitly has qualification gaps.
**Development:** the use case is building/testing the platform, not operating a
physical process.

| ID | Use case | Starting point | Main value |
|---|---|---|---|
| UC01 | Greenhouse telemetry and supervised climate rules | Template | Explain conditions and rule decisions before connecting real controls. |
| UC02 | Cold-chain excursions and corrective-action records | Template | Detect threshold events, escalate, and preserve an operator's response. |
| UC03 | Home or small-business energy insights | Adaptation | Turn meter readings into understandable, evidence-backed usage reports. |
| UC04 | Pump/fan condition monitoring and maintenance assistance | Template | Combine measurements with equipment and maintenance context. |
| UC05 | Restart-resilient incident investigation | Template | Produce a bounded investigation without losing its progress on restart. |
| UC06 | Building comfort, water leaks, and facility monitoring | Adaptation | Give facilities teams one view of environmental and equipment events. |
| UC07 | Remote equipment and renewable-energy telemetry | Adaptation | Observe local assets where bandwidth or cloud dependence is undesirable. |
| UC08 | Privacy-oriented traffic and parking counts | Template adaptation | Derive aggregate utilization without making identity collection the goal. |
| UC09 | Manufacturing process-quality screening | Adaptation | Flag non-critical process deviations for human inspection. |
| UC10 | Home-lab or small data-room health monitoring | Adaptation | Correlate service health with temperature, capacity, and power. |
| UC11 | Read-only vehicle diagnostics lab | Incubating | Learn and inspect diagnostic telemetry using a simulator first. |
| UC12 | White-label monitoring for multiple customer sites | Adaptation | Reuse the same operational foundation with customer-specific presentation. |
| UC13 | IoT and local-AI teaching/prototyping lab | Development | Demonstrate the full pipeline without buying every target device. |
| UC14 | Deployment sizing and recovery rehearsal | Development | Measure resource headroom and failure behavior before hardware rollout. |
| UC15 | Multilingual operator intake and incident routing | Adaptation | Suggest structured intent/routes while keeping the operator in control. |

## 3. Use-case sketches

### UC01. Greenhouse telemetry and supervised climate rules

**For:** growers, greenhouse technicians, and IoT learners.

**Pipeline:** temperature/humidity/soil readings -> MQTT -> Node-RED rules ->
InfluxDB -> Grafana/dashboard. Add Ollama for questions about measured conditions
and the recorded reasons behind rule decisions.

**Small MVP:** one simulated zone, a few readings, trend charts, and a visible
fan/valve decision log. Start with `mqtt-influx-grafana`; use
`mqtt-nodered-influx-grafana` for its simulated control loop, or
`mqtt-influx-grafana-ollama` for a read-only assistant.

**Proof of value:** known simulated events appear in the charts, every rule
decision is explainable, and stale readings are distinguishable from normal
conditions. Real actuation requires independently validated interlocks and
watchdogs; the LLM never opens a valve or overrides limits.

### UC02. Cold-chain excursions and corrective-action records

**For:** teams monitoring refrigerators, storage rooms, or temperature-sensitive
stock.

**Pipeline:** temperature readings -> MQTT/InfluxDB -> deterministic excursion
rules -> n8n notification/escalation -> signed acknowledgement and daily record.

**Small MVP:** one simulated refrigerator, an excursion grace period, an
acknowledgement flow, and a report containing the measurements and operator's
corrective-action note. Start with `mqtt-influx-grafana-n8n`; keep outbound email
in its demo mail sink until explicitly configured.

**Proof of value:** scripted excursions are detected, repeats do not create
confusing duplicate incidents, and acknowledgement does not erase the excursion
history. Sensor calibration, food-safety decisions, and regulatory certification
remain outside what this software example proves.

### UC03. Home or small-business energy insights

**For:** homeowners, workshops, and small-business facility operators.

**Pipeline:** meter/smart-plug readings -> MQTT -> validated power/energy
calculations -> InfluxDB/Grafana -> optional Ollama explanation.

**Small MVP:** monitor a few circuits or appliances, show daily consumption, and
answer "What changed this week?" from aggregated readings. Adapt
`mqtt-influx-grafana-ollama`; add a device connector and explicit unit/tariff
handling rather than assuming all meters emit the same schema.

**Proof of value:** energy totals agree with a known reference and every numeric
statement in the report can be traced to a query. Savings are a measured outcome,
not a promise. Collect readings through existing certified equipment; electrical
installation is not part of this prototype.

### UC04. Pump/fan condition monitoring and maintenance assistance

**For:** maintenance teams managing non-critical pumps, fans, or similar assets.

**Pipeline:** vibration/current/temperature -> MQTT -> qualified inference or
rules -> InfluxDB -> Grafana; Letta adds equipment and past-maintenance context,
and Ollama explains the evidence.

**Small MVP:** one asset, a baseline dataset, an anomaly history, and a read-only
assistant that retrieves previous work notes. Start with
`mqtt-influx-grafana-ollama-letta`.

**Proof of value:** the assistant distinguishes measurements, recorded work,
and hypotheses; known abnormal samples can be reviewed with the underlying
series. A classifier is not a proven remaining-life estimate, and an inferred
cause is not a maintenance instruction or permission to shut down equipment.

### UC05. Restart-resilient incident investigation

**For:** operators who need contextual incident summaries from an edge node.

**Pipeline:** anomaly result -> Node-RED alert -> Pi Durable investigator ->
bounded InfluxDB/read-only tools -> Ollama finding -> MQTT and stored history.
System 1 and Letta are optional sources of routing/history context.

**Small MVP:** inject one synthetic incident, interrupt the investigator, and
resume it. Start with `mqtt-influx-grafana-ollama-pidurable`; surface the stored
finding in Grafana before proposing a new dedicated dashboard workflow.

**Proof of value:** progress resumes, repeated alerts in the same incident window
are recognizable, and failed investigations are visible. Results can be
redelivered: consumers must deduplicate using the incident identifier. Human
approval and effectful agent tools are future work, not current guarantees.

### UC06. Building comfort, water leaks, and facility monitoring

**For:** building managers, schools, offices, and community facilities.

**Pipeline:** temperature/humidity/CO2/leak/equipment readings -> MQTT -> Node-RED
thresholds -> InfluxDB/Grafana -> optional n8n notification and local summary.

**Small MVP:** a few rooms, agreed thresholds, a stale-sensor indicator, and a
daily facilities report. Adapt `mqtt-influx-grafana` and add only the required
sensor connectors and workflow.

**Proof of value:** a missing sensor is not shown as a normal room, scripted
events reach the responsible operator, and aggregate room trends answer real
facilities questions. Do not identify occupants or infer individual health;
certified fire/life-safety systems remain independent.

### UC07. Remote equipment and renewable-energy telemetry

**For:** teams observing solar equipment, weather stations, or field machinery.

**Pipeline:** local device readings -> MQTT/Node-RED -> local InfluxDB/Grafana;
optional bounded AI summaries run on a suitably sized local hub.

**Small MVP:** one site with production/output, equipment temperature, weather,
and connectivity indicators. Adapt the telemetry template; add the equipment's
protocol adapter and identify which computations must work without internet.

**Proof of value:** local observation remains useful during an internet outage
and missing data is explicit. Durable device buffering, store-and-forward,
cross-site synchronization, and energy scheduling require separate design;
MQTT plus a local database does not establish those capabilities automatically.

### UC08. Privacy-oriented traffic and parking counts

**For:** site owners studying authorized entry points or parking utilization.

**Pipeline:** authorized camera -> go2rtc/custom visual ingestion -> aggregate
count events -> MQTT/InfluxDB -> Grafana -> optional local explanation.

**Small MVP:** one camera and timestamped counts, with no person identification.
The `mqtt-influx-grafana-ollama-go2rtc` reference is a starting point, but it
includes plate processing: an aggregate-only design needs an adapted ingress
that does not collect plates, not an assumed privacy toggle.

**Proof of value:** counts match a labeled test recording within an agreed
tolerance, retention is bounded, and camera views are authorized. Lighting,
occlusion, model rights, and privacy review are deployment gates; streaming a
camera does not by itself implement object detection.

### UC09. Manufacturing process-quality screening

**For:** small manufacturers inspecting non-critical packaging or process
consistency.

**Pipeline:** sensor/image/audio input -> domain preprocessing -> compatible
feature-vector classifier -> MQTT results -> human inspection queue/charts.

**Small MVP:** classify one observable condition, such as part presence or a
packaging defect, from a labeled test dataset. Reuse IoT storage and `ei-runner`;
the acquisition adapter, preprocessing, model, and inspection workflow are new
work. `factory-baseline` is planned, not an existing runnable template.

**Proof of value:** measure false accepts/rejects on held-out examples and show
unsupported or failed inference explicitly. Qualify input/output shapes and
labels, especially multi-output ONNX models. Do not automate safety-critical
release, machine motion, or worker assessment from this prototype.

### UC10. Home-lab or small data-room health monitoring

**For:** self-hosters and operators of small server/equipment rooms.

**Pipeline:** API/container health plus authorized environment/power readings ->
InfluxDB/Grafana -> deterministic alerts -> optional incident explanation.

**Small MVP:** one host, service-health cards, temperature/capacity trends, and a
rule that distinguishes "down" from "no recent reading." Use existing telemetry
and dashboard components; external-host exporters and long-term host metrics
need explicit connectors/storage.

**Proof of value:** a controlled service stop or capacity event is visible with
its timestamps and source. Unattended outage history must be collected server
side; dashboard-observed incidents alone do not cover periods when no screen was
watching. The LLM does not receive unrestricted Docker administration.

### UC11. Read-only vehicle diagnostics lab

**For:** learners and technicians exploring their authorized diagnostic data.

**Pipeline:** ELM327 simulator/qualified adapter -> OBD-II extra -> MQTT ->
InfluxDB/Grafana -> optional explanatory report.

**Small MVP:** simulator-only warmup, a few diagnostic readings, a trouble-code
change, and an adapter-disconnect event. Use `stacks\extras\obd2`; its README
labels it incubating and the current dependency has GPL-2.0-only licensing.

**Proof of value:** diagnostic samples and reconnect events appear correctly,
unsupported readings remain explicit, and read-only restrictions hold in the
simulator. Real-vehicle qualification and professional interpretation are
separate. Do not clear codes, modify vehicle control, or make the prototype a
driving-time attention task.

### UC12. White-label monitoring for multiple customer sites

**For:** integrators delivering local monitoring to several small deployments.

**Pipeline:** one local stack/API per site -> branded Flutter presentation ->
site-specific charts, services, and read-only assistance.

**Small MVP:** two simulated deployments and one brand, with role-appropriate
views and explicit endpoint selection. Reuse a suitable template and dashboard
theme tooling; add customer-specific schemas and content.

**Proof of value:** switching deployments never mixes credentials, histories,
or conversations, and each role sees only its allowed functions. The existing
client/deployment view is not evidence of centralized fleet authentication,
federation, push notifications, or an audited fleet backend. Start with separate
hosts/engines rather than assuming concurrent projects share one Docker host.

### UC13. IoT and local-AI teaching/prototyping lab

**For:** workshops, students, and teams learning the architecture.

**Pipeline:** synthetic device -> MQTT -> Node-RED -> InfluxDB/Grafana; optionally
compare a deterministic rule, mock inference, a qualified classifier, and a
local explanation of the same event.

**Small MVP:** one scripted device and guided exercises for a normal reading,
bad payload, disconnected sensor, and abnormal sample. Use the telemetry/control
templates and emulator; keep all actions inside simulations.

**Proof of value:** learners can trace an event end to end and distinguish model
output from a measurement or rule. Mock inference validates plumbing, not model
accuracy. Course materials and safe fixtures are additions, not built-in lessons.

### UC14. Deployment sizing and recovery rehearsal

**For:** maintainers deciding what fits on an edge node before purchasing or
rolling out hardware.

**Pipeline:** existing template + synthetic workload -> emulator resource
profile -> service metrics/reports -> target-hardware qualification.

**Small MVP:** compare a telemetry-only deployment with one optional AI service
under a Raspberry Pi-class profile. Measure memory peaks, message latency, stale
data, restart behavior, and recovery under controlled process interruption.

**Proof of value:** limits are actually applied, failures are visible, and the
report identifies headroom and bottlenecks. Quotas do not reproduce exact board
timing, peripheral behavior, thermals, or GPU acceleration. Confirm the result
on target hardware; do not reuse an x86 benchmark as a Pi/Jetson guarantee.

### UC15. Multilingual operator intake and incident routing

**For:** maintenance/help-desk teams with a small, predefined set of asset and
issue categories.

**Pipeline:** operator message + constrained evidence -> System 1 fixed-choice
question set -> suggestion/abstention -> human confirmation -> approved workflow.
Ollama may draft a summary after the structured fields are confirmed.

**Small MVP:** a few routing choices plus "unknown," an evaluation dataset in the
actual site languages, and a confirmation screen. Existing System 1 intake/
feedback infrastructure is a starting point; domain labels, evaluation, and any
work-order connector are custom work.

**Proof of value:** measure routing mistakes and abstentions, including ambiguous
and out-of-scope messages. Operators can correct the suggestion. Confidence is
not proof of correctness; severity, access checks, and safety decisions remain
deterministic and independent of the model.

## 4. Where to start

| Goal | Recommended first project | Add later, only if it earns its cost |
|---|---|---|
| Learn the whole platform | UC13 with one telemetry template | One qualified inference model or grounded assistant. |
| Deliver a useful monitoring pilot | UC01 telemetry-only, UC03, or UC06 | Domain alerts, then explanations over verified data. |
| Demonstrate workflow value | UC02 in the mail-sink demo | Authorized notification delivery and formal operator records. |
| Demonstrate local AI | UC04 or UC05 with synthetic history | Site-specific evaluation and measured edge capacity. |
| Prepare an edge rollout | UC14 against an existing template | Hardware-in-the-loop qualification and backup/recovery checks. |
| Explore a specialized extra | UC11 in its simulator | Authorized real-hardware qualification and license review. |

For every pilot, define the device/input contract, the human who benefits, one
observable success criterion, and the behavior when data/AI/network access is
unavailable. Start read-only. Move from simulation to a limited monitored pilot
before considering real-world actuation.

## 5. Shared constraints and implementation boundaries

- Use generated device credentials, explicit access rules, appropriate transport
  protection, and controlled networks. Current deployments are trusted-network
  development systems, not certified public-facing or safety-critical products.
- Bound messages, logs, model work, feedback data, footage, and report retention.
  Mark data freshness and distinguish simulated values from real measurements.
- Treat generated explanations as hypotheses grounded in referenced records.
  Calculate numbers outside the LLM and require explicit human decisions for
  effects; the current durable investigator is read-only.
- Keep sensors, protective controls, and authorized operator actions independent
  of generative AI. Do not infer compliance, health, identity, or equipment
  safety from a demo or classifier confidence.
- Enable only services that fit the measured workload. Inference does not require
  Ollama, Letta, or n8n; a telemetry project may need none of them.
- Existing template directories are reference recipes. Template search/install,
  `p4n4 ei` deployment commands, and some integration/automatic-bootstrap work
  remain incomplete; follow the actual template/component README rather than
  treating roadmap command examples as implemented.
- Resolve relevant existing defects and qualify the chosen deployment path.
  `docs\project\review-and-improvements.md` records known ONNX, lifecycle,
  networking, CORS, foreground-start, and shared-configuration limitations.
- The Rust rewrite is a planned platform implementation change, not another
  application domain or an available prerequisite. These use cases should guide
  acceptance tests while respecting its clean-break and phase decisions.

## 6. Local references

| Reference | What it establishes |
|---|---|
| `README.md` | Platform scope, service families, trusted-network stance, and examples. |
| `docs\reference\template-registry.md` | Seven documented template starting points and planned/stub template commands. |
| `tools\templates\projects` | Existing reference project directories named in the sketches. |
| `tools\templates\docs\use-cases\greenhouse-telemetry.md` | Existing greenhouse telemetry walkthrough. |
| `tools\templates\docs\use-cases\greenhouse-assistant.md` | Existing grounded-assistant walkthrough. |
| `tools\templates\docs\use-cases\road-traffic.md` | Existing camera/traffic reference and its domain-specific ingestion. |
| `docs\stacks\edge-inference.md` | Numeric feature-vector contract, backend selection, mock mode, and deployment limits. |
| `docs\reference\agent.md` | Read-only investigation, restart recovery, bounded tools, and redelivered results. |
| `docs\decisions\system1.md` | Closed-choice routing, abstention, evaluation, and rule/model boundaries. |
| `docs\reference\emulator.md` | Resource profiles, calibration, reports, and limits of workstation emulation. |
| `stacks\extras\obd2\README.md` | Read-only extra, simulator, licensing, and hardware-qualification gaps. |
| `api\TODO.md` and `dashboard\TODO.md` | Remaining integration, server-history, and fleet-related work. |
| `docs\project\review-and-improvements.md` | Current review findings and the improvement backlog. |
| `RUST_REWRITE_PLAN.md` | Future implementation decisions; not a deployed runtime. |
