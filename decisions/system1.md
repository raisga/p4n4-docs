# p4n4 — System 1: Fast Typed Decisions with Laya

**Version:** 0.1  
**Date:** 2026-09-24  
**Status:** Draft. The spike (S0) is done; nothing is implemented in the stacks yet.

---

Evidence tags used throughout:

| Tag | Meaning |
|-----|---------|
| **[measured]** | Measured in the spike: x86-64 i9-14900KS, Docker Desktop on WSL2, CPU only, `--cpus 4`, 2 GiB, unless stated |
| **[source]** | Read in the source of laya 0.3.11 (0.3.17 where stated) or huggingface_hub 1.32.0 |
| **[upstream]** | Stated by the model card at revision `aa8c91c`, by PyPI or by upstream issues |
| **[estimate]** | Our judgement, still to be measured |

## Contents

1. [Summary](#1-summary)
2. [What Laya Is](#2-what-laya-is)
3. [Use Cases](#3-use-cases)
4. [Spike Results](#4-spike-results)
5. [Rules of Use](#5-rules-of-use)
6. [Architecture](#6-architecture)
7. [Integration by Repository](#7-integration-by-repository)
8. [Budgets per Board](#8-budgets-per-board)
9. [Security and Supply Chain](#9-security-and-supply-chain)
10. [Calibration and Evaluation](#10-calibration-and-evaluation)
11. [Phases](#11-phases)
12. [Risks](#12-risks)
13. [Open Questions](#13-open-questions)

Appendices: [A. Reference Wrapper](#appendix-a-reference-wrapper) · [B. Image](#appendix-b-image) · [C. Question Sets v2](#appendix-c-question-sets-v2) · [D. Calling System 1](#appendix-d-calling-system-1) · [E. Method Notes](#appendix-e-method-notes)

---

## 1. Summary

[Laya](https://huggingface.co/convaiinnovations/laya) is a small encoder model that answers *typed questions* about a state. You give it a JSON or text state and a question with a closed set of options. In one forward pass, it returns one of those options with probabilities and a confidence. It has no decoder, so it cannot write text or answer outside the options.

This document proposes **System 1** for p4n4: an opt-in `system1` service in the ai stack. It runs laya on the CPU next to Ollama and Letta and makes fast, bounded decisions, for example "which team should see this alert?" or "which pump is this operator message about?". The name follows laya's own API (`/v1/systemone`) and the fast/slow split. System 1 picks from a fixed menu in a fraction of a second. System 2 (Ollama, Letta) is slow and open-ended.

**Rules decide. System 1 routes. System 2 explains.**

Everything that must be exact stays in rules: severity, safety, actuation and thresholds. System 1 adds suggestions where the rules have no answer, and abstains when unsure. System 2 writes explanations and handles open-ended requests.

### Headline findings

- **It fits a Pi-class budget.** One multilingual checkpoint behind the reference wrapper uses 0.9–1.1 GiB within a 2 GiB limit. It answers question-set requests in 0.1–0.25 s on 2–4 x86 cores [measured]. A Pi 5 should be 4–6× slower [estimate].
- **Routing works when the menu is closed, the wording is right and the state holds only evidence.** Given the alert text alone, cause routing picked the right option for all 5 clear alerts and cleared the threshold on 4. It abstained on all 3 alerts that fit no option. Operator intent got 7 of 8 messages right in Spanish, Hindi and English [measured].
- **It is not a zero-shot decision engine.**
  - Seven wordings of one question scored from 2/8 to 7/8.
  - Fields that carry no evidence steer the answer. With a placeholder device name in the state, the 3 alerts that fit no option (a firmware notice, an open door, an e-stop) were routed to `connectivity` at confidence 0.70–0.90, and lorem ipsum got `connectivity` at 1.00.
  - 3,900 × "x" was called `electrical` at 0.76 [measured].
  - The model card agrees: the base checkpoints are near chance zero-shot on its own benchmark [upstream].
- **Gates and severity stay rule-based.** Yes/no gates lean towards "no", whatever the keys, option order or polarity. The best variant filtered 3 of 6 small-talk messages, and none above a 0.5 threshold. Severity was exact on 3 of 8 alerts [measured].
- **Stock `laya-serve` fails silently in ways that matter.**
  - It truncates long states and options without telling the caller.
  - It runs without auth when no key is set.
  - It accepts unbounded work and loads extra checkpoints on demand. In 0.3.11 its 2 MiB body cap read only a declared `Content-Length`, and a chunked 384 MiB request got it OOM-killed under a 2 GiB limit; 0.3.12 fixed that.

  The reference wrapper (Appendix A) closes these gaps. It was verified in a hardened, non-root, read-only container, including through the proposed Compose service [measured].
- **Upstream is days old and moving fast.** The model repository was created on 2026-09-18. Nine releases (0.3.9–0.3.17) shipped in 13.5 hours on 2026-09-23/24. The weights haven't changed since the tested revision [upstream], and the wrapper runs unchanged on 0.3.17 with identical answers [measured].

### Recommendation

Build S1 and S2 behind the `system1` Compose profile, off by default:
- **S1:** the service, incident routing and the decision log.
- **S2:** operator intake, the Node-RED subflow and the harness tool.

Every question set ships with an evaluation file and gates, enforced by CI and `p4n4 ai system1 eval`. Nothing safety-related depends on System 1, and every consumer still works when System 1 is down or abstains. Pin everything: the image, Python packages, laya version, model revision and question-set version.

---

## 2. What Laya Is

### 2.1 Checkpoints

The Hugging Face repository `convaiinnovations/laya` holds three checkpoints at the tested revision `aa8c91c`:

| Checkpoint | Encoder (licence) | Sequence / head tokens | State budget | Memory [measured] | Shipped calibration |
|------------|-------------------|------------------------|--------------|-------------------|---------------------|
| `english` (repo root) | ModernBERT-large (Apache-2.0) | 512 / 192 | 316 tokens | ~1.9 GiB fp32, ~1.7 GiB with fp16 tables | Per-bucket temperatures. `choice:11+` is 0.1006 and is clamped to 0.5 at load |
| `multilingual` | mmBERT-base (MIT) | 1024 / 256 | 764 tokens | 871 MiB idle, 1.1 GiB peak with the wrapper | None (T = 1) |
| `typed-decisions` | Not stated | 1024 / 256 | 764 tokens | ~1.7 GiB | Per-bucket temperatures |

- The state budget is `max_len − head_max_len − 4` special tokens [source].
- The downloads are 678 MB (647 MiB) for `multilingual` and 1.6 GB for all three. The root model has 421,293,827 parameters, stored as F16 [upstream].
- Language coverage: the model card rates 23 of 51 languages as usable (more than 3× random) for `english`, and 45 of 51 for `multilingual` [upstream].

**p4n4 defaults to `multilingual`.** It is the smallest, reads the longest states and covers the languages operators use.
- `english` is an option only for English-only sites, and only if their evaluation shows a gain.
- `typed-decisions` is excluded until its base model and licence are stated.

### 2.2 Question types

| Type | Input | Output | p4n4 use |
|------|-------|--------|----------|
| `choice` | Options as `{key: description}` or a list | `choice`, `probabilities`, `confidence` | **Yes.** The only type used in production |
| `score` | Ordered level descriptions, lowest first | `score` (expected level index), `legend`, `probabilities`, `confidence` | No. Upstream calls it the weakest primitive (SST-5: 0.372) [upstream]. Our severity scores bunched in the middle, 0.94–2.38 on a 0–3 scale [measured] |
| `noul` | A yes/no question | `noul` = P(yes), `confidence` = max(p, 1 − p) | No. It follows its labels and can return a confident "no" ([NandhaKishorM/laya#156](https://github.com/NandhaKishorM/laya/issues/156)). It scored 6/14 on our gate test [measured] |

Every answer also includes `action.act_probability`. It carries no usable signal ([NandhaKishorM/laya#185](https://github.com/NandhaKishorM/laya/issues/185)): it reads 1.0 almost always and has an AUROC of 0.30, against 0.77 for `confidence` [upstream]. **Ignore it.**

### 2.3 How a decision is computed [source]

- **Sequence.** Each question becomes its own sequence: `[CLS] "<type> question: <instructions>" [SEP] [MASK] key: description … [SEP] state [SEP]`, with one `[MASK]` per option. The option logits are read at the `[MASK]` positions, divided by a temperature and passed through a softmax.
- **Temperature.** It comes from the checkpoint's table, keyed by type and option count: `choice:2`, `choice:3-5`, `choice:6-10`, `choice:11+`, and the same four for `score` and `noul`. Values are clamped to [0.5, 5.0] at load.
- **Confidence.** For `choice` and `score`, `confidence` is `1 − H(p) / ln k`, one minus the normalised entropy. It is not the top probability. Two options at 0.9/0.1 give 0.53, so thresholds are stricter than they look. laya 0.3.12+ adds `answer_confidence`, the top probability [upstream].
- **Budgets.** They are enforced by silent truncation:
  - each option is cut to 48 tokens;
  - the instructions are cut to `max(8, option budget)`;
  - the state is cut to whatever is left, from the right for strings and objects and from the left for lists.

  The answer still comes back, based on text the caller never sees (§4.5).
- **Cost.** Questions in one request are independent, and the state is encoded once per question. Cost and peak memory therefore grow with questions × state tokens.

### 2.4 Ways to run it

| Surface | What it is | p4n4 use |
|---------|------------|----------|
| Python API | `Router`, `agent.predict(state, questions)` | Inside the wrapper |
| `laya-serve` (`laya[serve]`) | FastAPI app with `POST /v1/systemone` and `GET /health`. Bearer auth applies only when `LAYA_API_KEY` is set | Mounted by the wrapper (§6.2) |
| MCP (`laya[mcp]`) | stdio tools for MCP hosts | No. The harness exposes its own tools (U8) |
| ONNX (`laya[onnx]`) | Export plus ONNX Runtime inference | Not evaluated. The research found gaps in how it handles split weight files; revisit only for size |
| LangChain/LangGraph extras, `laya-ts` | Framework adapters, TypeScript client | No |
| Upstream Docker image | `python:3.11-slim-bookworm`, UID 10001, `USE_TF=0`, 8 GB RAM recommended, demo command | No. p4n4 builds its own (Appendix B) |

### 2.5 Licences

- **`laya` package:** Apache-2.0. The package metadata names Convai Innovations as author and requires Python ≥ 3.10, torch ≥ 2.0 and transformers ≥ 4.48 [source].
- **Model card:** `apache-2.0`. The encoders are ModernBERT-large (Apache-2.0) and mmBERT-base (MIT) [upstream].
- **`typed-decisions`:** its base model isn't stated, so its licence chain is incomplete. Don't ship it.
- **Fine-tuning material:** upstream also publishes a fine-tuning notebook and the dataset `LocalLLaMA/typed-decisions`. Check their licences before S4.
- **Code in the model repository:** the repository also holds Python files (`rl_agent_api.py`, `rl_common.py`, `email_utils.py`). The wrapper's `allow_patterns` never downloads them, and nothing runs remote code.

### 2.6 Maturity

| Date (UTC) | Event [upstream] |
|------------|------------------|
| 2026-09-18 | Model repository created |
| 2026-09-23 14:55 → 18:06 | 0.3.9, 0.3.10 and 0.3.11 released on PyPI; HF commit `aa8c91c` (0.3.11 notes). **This is what was tested** |
| 2026-09-24 03:25 / 03:47 | 0.3.12 and 0.3.13 released on PyPI; HF commits `80e2e0e` and `fe2b771` change only `README.md`. The safetensors files are identical |
| 2026-09-24 04:09 → 04:26 | 0.3.14, 0.3.15, 0.3.16 and 0.3.17 released on PyPI; HF README commits up to `bf2e02a` (now `main`). The weights are unchanged: `multilingual/model.safetensors` has the same sha256 |

0.3.12–0.3.17 add [upstream; source]:
- `agent.decide(state, schema=...)`;
- per-language calibration (`lang_temperatures=`);
- `answer_confidence`;
- a streamed body limit in `laya-serve`, which fixes F10 (§4.5), and a 401 for a malformed bearer header;
- a fix for option order in `predict_batch`;
- CUDA fast-path fixes.

**The wrapper runs unchanged on 0.3.17** [measured]. With the same wrapper and checkpoint, all 50 answers to the 22 set requests of §4.4 matched 0.3.11 exactly (Δp = 0), plus the new `answer_confidence` field. Latency and memory were the same. F1–F8 still hold in 0.3.17 [source]. S1 adopts the latest release that passes the regression suite and the evaluations (§10). Treat laya as pre-1.0 software from a young project: pin it, and read every diff before bumping it.

### 2.7 Limits stated upstream [upstream]

The model card's "Honest Limits" section at `aa8c91c` is unusually frank, and the spike agrees with every point:

- **"Laya is a fast base to specialise, not a zero-shot decision engine."** On the typed-decisions benchmark, the base checkpoints score 0.362 (`english`) and 0.352 (`multilingual`). Random scores 0.318, and always picking the majority option scores 0.461.
- **Long menus starve.** With Banking77's 77 options, each label gets 3–4 tokens, and accuracy is 0.425 against 0.870 for the card's comparison model (Jev).
- **Shipped probabilities are over-confident.** Per-bucket refits reduce the expected calibration error from 0.466 to 0.081 (`english`) and from 0.314 to 0.106 (`multilingual`).
- **Other types and checkpoints.** `score` is the weakest primitive, `noul` can be confidently wrong, `act_probability` is noise, and the root checkpoint is English only.
- **Checkpoint switching.** `Router(max_loaded=1)` reloads on every language switch, which takes 7–10 s; the lazy default is `max_loaded=2`. The wrapper serves exactly one checkpoint, so it never switches.

---

## 3. Use Cases

| ID | Use case | Where | Questions | Phase |
|----|----------|-------|-----------|-------|
| U1 | Likely cause and team for critical alerts | n8n `incident-escalation` v2 | `incident-triage.cause` (7 options) | S1 |
| U2 | Operator message intake, any language | n8n chat workflow | `operator-intake.intent`, `.asset` | S2 |
| U3 | System 2 router: does a request need the LLM or an agent, and which one? | n8n, harness | New set | S3 |
| U4 | Device class suggestion for a newly seen device | Node-RED, harness | New set | S3 |
| U5 | Log line triage | n8n, harness | New set | S3 |
| U6 | Screening signal: is this anomaly window worth an LLM explanation? | Node-RED | New set | S3 |
| U7 | Labels for past alerts (reports, Grafana annotations) | Batch job | Reuses U1 | S3 |
| U8 | Harness tools `system1_decide` and `system1_eval` | `p4n4-harness` | Any set | S2 |
| U9 | `p4n4-system1` subflow in the Node-RED AI palette | stacks/iot | Any set | S2 |

**U1: likely cause and team for critical alerts (S1).**
- **Today:** `incident-escalation` asks Ollama for free-form JSON with a severity. If the reply doesn't parse, `Parse Classification` sets `notify_team: false`, and `Notify Team?` drops the critical alert (`stacks/ai/config/n8n/workflows/incident-escalation.json:61`, `:80`).
- **In v2, every critical alert is escalated:**
  - severity comes from the topic and the site rules;
  - a rule table maps known alert types to teams;
  - System 1 adds a `likely_cause` and `likely_team` only when no rule routes the alert and the answer clears its threshold (§6.5).
- **Measured:** given the alert alone, all 5 clear causes got the right option, 4 of them above the threshold, and the 3 alerts outside the menu abstained. With a device name or type in the state, 2–3 of those 3 were confidently misrouted. So e-stop, door and firmware events are caught by rules first, and the state holds only the alert (§4.4).

**U2: operator intake (S2).**
- **What it does:** messages from a chat channel become a draft ticket or a status reply, using `intent` (fault report, status request, maintenance request) and `asset`.
- **Measured (Spanish, Hindi and English):**
  - all 8 requests were handled safely: 5 routed right, 3 abstained, none confidently wrong;
  - the asset was right on 6 of 8, and the other 2 abstained;
  - the `small_talk` gate filtered 0 of 6 greetings, and the intent question then routed them as requests at 0.58–0.90.
- **Design:** rules filter small talk first (message length plus a greeting list per language), and every draft needs the operator's confirmation.

**U3: System 2 router (S3).** Before calling Ollama or a Letta persona (specs F-0.2.1), decide whether the request needs one, and which. On a Pi, one skipped LLM call saves more time than System 1 costs [estimate]. A wrong route must only ever cost a slower answer.

**U4: device class (S3).** Suggest a `device_type` for a new device from its topic, payload keys and units, to pick templates and dashboards. The user always confirms, in the CLI or the harness.

**U5: log triage (S3).** Classify single log lines (network, auth, disk, memory, crash, config). Logs are token-dense: 2,940 and 3,900 characters of syslog came to 1,784 and 1,350 tokens [measured]. The caller therefore sends one pre-extracted line, never a log tail.

**U6: screening signal (S3).** Rank anomaly windows to pick which ones get an LLM explanation. This is a ranking signal only, never a gate (§4.4).

**U7: alert labels (S3).** Label past alerts in batch for reports and Grafana annotations, where an error is cheap and visible.

**U8: harness tools (S2).**
- `system1_decide` (tier R) lets an MCP host run a question set on project state cheaply.
- `system1_eval` (tier R) runs a set's evaluation.

**U9: Node-RED subflow (S2).** A `p4n4-system1` subflow for the AI palette in F-0.2.2 of `docs/decisions/specs.md`. It follows that section's subflow catalog (lines 407–413) and its failure contract: set `msg.error` and send a null payload (line 419).

**Not use cases:**
- severity or priority;
- safety interlocks, and anything published on `commands/…`;
- yes/no gates;
- numeric judgements (compute them in the caller);
- free text such as summaries or replies;
- anything without an evaluation file;
- states over the 764-token budget;
- languages the evaluation doesn't cover.

---

## 4. Spike Results

### 4.1 Setup

- **Host:** x86-64 i9-14900KS, Docker Desktop on WSL2, CPU only. Containers ran with `--cpus 4 --memory 2g` unless stated.
- **Software:**
  - Python 3.12.14, torch 2.14.0+cpu, transformers 5.17.0;
  - laya 0.3.11, huggingface_hub 1.32.0, safetensors 0.8.0;
  - fastapi 0.141.1, uvicorn 0.53.0.
- **Model:** revision `aa8c91ca088ec597df95a0d1c76b3063cb2ae5e8`, checkpoint `multilingual` unless stated.
- **Container flags:** `--read-only --tmpfs /tmp` with the models volume read-only. The hardened runs add `--user 10001:10001 --cap-drop ALL --security-opt no-new-privileges`.
- **Test sets:** small and hand-labelled, with 8–14 items per question. Repeat runs gave identical answers. **They find failure modes; they do not measure accuracy.** Accuracy comes from each site's evaluation files (§10).

### 4.2 Footprint [measured]

| Configuration | Resident after load | Peak |
|---------------|---------------------|------|
| Stock `laya-serve`, `multilingual` only | 1.69 GiB | — |
| Stock, `english` + `multilingual` loaded | 3.47 GiB | — |
| Stock, all three loaded | 4.85 GiB | — |
| Stock worst case: 64 questions × 877-token state | — | 4 GiB, 48 s |
| Wrapper, fp32, 4 questions per chunk | 1,623 MiB | 2,010 MiB |
| Wrapper, fp32, 1 question per chunk | — | 1,784 MiB |
| Wrapper, tables cast to fp16 after load | 1,239 MiB | 1,981 MiB at start-up |
| **Wrapper default:** fp16 tables read from safetensors, 2 questions per chunk | **871 MiB** | 998 MiB (1 question); 1,123 MiB (2–8 questions × 754 tokens); 1,614 MiB at start-up |
| Same, through the Compose service in §7.1, after two requests | 965 MiB | — |

- **Memory limit:** a 1,600 MiB limit is OOM-killed during start-up and 1,792 MiB passes. **The recommended limit is 2 GiB.**
- **Rejected options:**
  - fp32 with 4 questions × 2,000 characters was OOM-killed at 2 GiB;
  - int8 dynamic quantisation flipped 3 of 7 answers;
  - casting the whole model to fp16 crashed on the CPU.
- **Why fp16 tables are safe:** the checkpoint is stored as F16, so keeping the vocabulary tables in fp16 is lossless. Answers were bit-identical.

### 4.3 Latency and tokens [measured]

| Request (`multilingual`) | 4 threads | 2 threads | 1 thread |
|--------------------------|-----------|-----------|----------|
| 1 question × 170-token state | 112 ms | 178 ms | 327 ms |
| 1 question × 754-token state | 553 ms | 945 ms | 1,728 ms |
| 8 questions × 754-token state | 4.1–4.5 s | — | — |

- **Question-set requests** with short states took 84–251 ms. Through Compose, with 2 threads and 2 CPUs, they took 169 ms and 251 ms.
- **`english` (ModernBERT-large):** 407 ms for 1 question × 252 tokens.
- **Concurrency:** one worker serves requests in order. Four concurrent callers waited 212 ms each, against 55 ms for one alone, so callers need timeouts (§6.5).
- **Chunking:** running questions one at a time instead of two at a time costs about 7% more latency.
- **Start-up:** healthy 5–12 s after start when the checkpoint is already on disk. The first pull took 32 s.
- **Pi 5:** should be 4–6× slower [estimate]. A native measurement is an S1 exit criterion.

**Tokens** (`multilingual` tokenizer):
- 1,700 characters of alert text ≈ 754 tokens.
- Syslog is denser: 2,940 characters = 1,784 tokens, and 3,900 characters = 1,350 tokens.
- Repeated characters are cheap: 3,900 × "x" = 365 tokens.
- The `english` tokenizer turns 1,700 characters into 513 tokens, but that checkpoint reads only 316.

**Rule of thumb:** a `multilingual` state holds about 1,500 characters of prose, and less of logs.

### 4.4 Quality [measured]

**Intent wording.** Three intents, 8 requests in Spanish, Hindi and English:

| Variant | Right |
|---------|-------|
| A: bilingual descriptions + "anything else" (v1) | 5/8 |
| B: bilingual descriptions, no catch-all | 7/8 |
| C: English descriptions + "anything else" | 4/8 |
| **D: English descriptions, no catch-all (v2)** | **7/8** |
| E: bare labels + "other" | 2/8 |
| F: bilingual + a concrete "greeting, thanks or small talk" option | 3/8 |
| G: English + a concrete "greeting, thanks or small talk" option | 3/8 |

**Gates.** 14 messages: 8 requests and 6 greetings or thanks.

| Variant | Right | Requests kept | Small talk caught |
|---------|-------|---------------|-------------------|
| G1: "Does the operator need anything?" `yes`/`no` | 6/14 | 0/8 | 6/6 (always "no") |
| `noul`, same question | 6/14 | — | — |
| G3: descriptive keys (`request` / `small_talk`) | 8/14 | — | — |
| **G4: inverted, "Is the message only a greeting or thanks?" (v2)** | **11/14** | 8/8 | 3/6, and 0/6 at threshold 0.5 (confidence 0.09–0.43) |
| G5: closed 4-way intent including `small_talk` | 9/14 | 3/8 | — |
| G6: neutral `A`/`B` keys, upstream's workaround | 6/14 | 0/8 | 6/6 (always B, the "no" description) |
| G7: neutral `A`/`B` keys, inverted | 10/14 | 8/8 | 2/6, and 0/6 at 0.5 |
| G8: G7 with the options swapped | 10/14 | — | — |

The model leans towards whichever description starts with "no", whatever the keys or their order. Upstream's neutral-key workaround (NandhaKishorM/laya#156) doesn't fix this. A gate is safe only when "no" is the safe answer, and even then it rarely clears a threshold.

**Severity.** 4 levels, 8 alerts; we judged the expected levels:

| Form | Exact | Within one level | Mean abs. error | Confidence or range |
|------|-------|------------------|-----------------|---------------------|
| `choice` (low/medium/high/critical) | 3/8 | 6/8 | 0.88 | Confidence 0.07–0.51 |
| `score` (4 ordered levels) | 2/8 | 7/8 | 0.88 | Scores 0.94–2.38 |

**Cause routing.** `incident-triage` v2 with threshold 0.5, on the same 8 alerts, with three states:
- the alert alone;
- the alert plus a placeholder device name and site, `"device": "dev-1", "site": "plant-2"`, as in the 0.3.11/0.3.17 comparison;
- the alert plus a realistic device name, `device_type` and site.

Each cell gives the answer and its confidence. "abst." marks an abstention, and bold marks a confident answer to an alert outside the menu:

| Alert | Expected | Alert alone | + `dev-1`, `plant-2` | + realistic device and type |
|-------|----------|-------------|----------------------|-----------------------------|
| Hydraulic oil at 92 °C, 12 °C above its limit | thermal | thermal 0.66 | thermal 0.35, abst. | thermal 0.51 (`hydraulic_press`) |
| Battery at 20%, sensor still reporting | sensor_fault | sensor_fault 0.48, abst. | connectivity 0.32, abst. | thermal 0.32, abst. (`temperature_sensor`) |
| `connect ECONNREFUSED …:1883`, no telemetry for 10 min | connectivity | connectivity 1.00 | connectivity 1.00 | connectivity 1.00 (`gateway`) |
| Smoke in electrical cabinet 3 | electrical | electrical 1.00 | electrical 1.00 | electrical 1.00 (`smoke_detector`) |
| Vibration 7.1 mm/s RMS on compressor-1 | mechanical | mechanical 0.98 | mechanical 0.95 | mechanical 0.99 (`compressor`) |
| Firmware update available for gw-01 | Outside the menu (rules) | unknown 0.21, abst. | **connectivity 0.90** | **connectivity 0.90** (`gateway`) |
| Cold-room door open for 3 min | Outside the menu (rules) | electrical 0.36, abst. | **connectivity 0.70** | electrical 0.27, abst. (`door_sensor`) |
| Emergency stop pressed on line 4 | Outside the menu (rules) | connectivity 0.29, abst. | **connectivity 0.81** | **electrical 0.62** (`plc`) |
| **Confidently wrong** | | **0 of 8** | **3 of 8** | **2 of 8** |

- **The alert alone did best.** It picked the right option for all 5 clear alerts and cleared the threshold on 4. It abstained on all 3 alerts outside the menu, and `unknown` won the firmware notice.
- **A device name is a prior, not evidence.**
  - With `"device": "dev-1"` and no site, 7 of the 8 alerts went to `connectivity` at 0.67–1.00, including the smoke alert (0.91). Only the vibration alert got another answer.
  - A bare "Lorem ipsum dolor sit amet" with any of three device names got `connectivity` at 0.51–1.00 (garbage table below).
  - Renaming the field to `asset`, or dropping "device" from the `connectivity` description, weakened the pull without removing it. With either change, lorem ipsum with `dev-1` still got `connectivity` at 0.79–1.00, and 1–2 of the alerts outside the menu were still answered confidently.
- **Device types help some alerts and hurt others.** `hydraulic_press` lifted the hydraulic alert from 0.35 (with `dev-1`) to 0.51, but the alert alone gave 0.66. `plc` sent the e-stop to `electrical` at 0.62. Lorem ipsum from a `gateway` got `connectivity` at 0.97, and from a `plc` `electrical` at 0.69.
- **So the `incident-triage` state is the alert alone** (rule 6). The device and site stay in the workflow for the rules and the escalation (§6.5). A site that wants a device type in the state must show a gain in its evaluation, including uninformative alerts for every device type (§10.1).

Phrasing matters too. In the regression set, whose states carry device fields, an overheating alert got `thermal` at 0.58, and a reworded copy got 0.46, below the threshold. A differently worded battery alert got `sensor_fault` at 0.40 and abstained.

**Operator intake.** `operator-intake` v2 on the same 14 messages:
- Requests: all 8 handled safely. 5 were routed right, 3 abstained, and none was confidently wrong.
- Asset: 6 of 8 right, and the other 2 abstained.
- Small talk: 0 of 6 filtered at the 0.5 threshold. The intent question then routed them as status or maintenance requests at 0.58–0.90.

**Garbage and edge inputs** (`incident-triage`):

| State | Answer |
|-------|--------|
| 3,900 × "x" as the alert | `electrical`, P 0.90, confidence 0.76 |
| Lorem ipsum as the alert alone, four wordings | Abstained (confidence 0.20–0.43); `unknown` won on 2 |
| A bare "Lorem ipsum dolor sit amet" with `dev-1`, `pump-3` or `sensor-12` as the device | `connectivity` at 0.51–1.00, not abstained |
| Lorem ipsum as the alert of a `gateway` / a `plc`, with device name and type | `connectivity` at 0.97 / `electrical` at 0.69 |
| Lorem ipsum as the alert of five other device types | Abstained |
| Empty, or `{"alert": "ok"}` | Abstained |

**Exactness:**
- Splitting a request's questions into chunks changed no probability: Δp = 0, and 0 of 42 answers flipped.
- The fp16 vocabulary tables gave bit-identical answers.
- The calibration hook works: `choice:2` at T = 1.9 moved P(A) from 0.4671 to 0.4826. The expected value, sigmoid(logit(p) / 1.9), is 0.4827.

### 4.5 Silent failure modes and the wrapper's fixes

| # | Stock behaviour (laya 0.3.11, huggingface_hub 1.32) | Risk | Wrapper |
|---|------------------------------------------------------|------|---------|
| F1 | A state over the budget is truncated and still answered | A decision based on evidence nobody sees | 422 giving the token count and the budget |
| F2 | Options over 48 tokens are cut | Two options can become identical | 422 naming the option |
| F3 | Instructions are trimmed when options crowd the head | The question silently changes | 422 when instructions plus options exceed `head_max_len` |
| F4 | `laya-serve` serves without auth if `LAYA_API_KEY` is unset | An open decision service | Refuses to start without a key of 32+ characters |
| F5 | laya has no revision argument, so downloads always follow the default branch | Weights change under you | Downloads a 40-character commit sha itself and loads that local copy; offline at run time |
| F6 | No limit on the number of questions or the state size | 64 questions × 877 tokens took 48 s and 4 GiB | At most 8 questions and 4,000 characters; questions run 2 at a time |
| F7 | The router loads other checkpoints on demand (`max_loaded=2`) | 3.5–4.9 GiB of memory | One pinned checkpoint, `max_loaded=1`; the caller's `model` is ignored |
| F8 | Out-of-range calibration temperatures are clamped to [0.5, 5] with only a Python warning, and an unknown bucket name is ignored | A typo becomes a different model | Start-up fails on an unknown bucket or an out-of-range value |
| F9 | An offline start fails if the pull ran as another user (see §4.6) | Service down after a "successful" pull | Pull and serve both run as UID 10001; a clear message and exit when the checkpoint is missing |
| F10 | The 2 MiB body cap is checked only against a declared `Content-Length`, so a chunked body is read and parsed in full. Fixed upstream in 0.3.12 | A key holder can exhaust memory: 256 MiB chunked took the server from 1,039 to 1,829 MiB, and 384 MiB got it OOM-killed at 2 GiB | Counts every body as it arrives, on every route, and returns 413 over 64 KiB |
| F11 | Confident answers on garbage, or on device fields alone (3,900 × "x" → `electrical` at 0.76; lorem ipsum with the device name `dev-1` → `connectivity` at 1.00) | Plausible nonsense | Can't be fixed in the service. Use rules first, evidence-only canonical states and evaluation (§5) |

The wrapper also adds:
- server-side question sets (`POST /v1/sets/<name>`), with a version and per-question thresholds that set an `abstain` flag;
- a check at start-up of every set against the model's budgets;
- fp16 vocabulary tables;
- an optional site calibration file;
- a `pull` mode.

### 4.6 Hardened deployment [measured]

- **The regression suite passes.**
  - Every fail-closed start exits with status 1 and a clear message.
  - Responses are 401 without a token; 404 for an unknown set or a GET; 400 for a body without `state` or malformed JSON; 413 over 64 KiB; 422 for budget violations.
  - The 413 applies on every route, chunked or not. A chunked 384 MiB body was refused in 0.6 s, and the server's memory didn't move.
  - `/health` needs no token, and there were no OOM kills.
- **Non-root with no privileges.** As UID 10001 with all capabilities dropped and `no-new-privileges`, the container shows `CapEff` 0 and `NoNewPrivs` 1. The root filesystem is read-only, and the answers are unchanged (smoke in cabinet 3 → `electrical`, 0.99).
- **Pull and serve must run as the same user.**
  - huggingface_hub 1.32 writes its offline index of a revision (`hub/models--convaiinnovations--laya/trees/<sha>.json`) with mode 0600.
  - After a pull as root, UID 10001 can't read that index. It logs "Ignoring corrupted tree cache file", and the offline `snapshot_download(allow_patterns=…)` then raises `OfflineModeIsEnabled`.
  - The fix, verified end to end: the image creates `/models` owned by 10001, a new named volume inherits that owner, and the `system1-pull` service runs as 10001.
- **Telemetry off.** With `HF_HUB_DISABLE_TELEMETRY=1`, huggingface_hub neither fetches its AI-agent "harness" registry (`/api/agent-harnesses`) nor writes `.agent_harnesses.json` into `HF_HOME` [source: `huggingface_hub/utils/_headers.py:183-189`; measured].
- **The Compose service in §7.1 was run exactly as written:**
  - `docker compose config --quiet` passes with no `.env`, as in the stack's CI. `${VAR:?}` would fail there even with the profile off, which is why the key uses `${SYSTEM1_API_KEY:-}` and the service fails closed instead.
  - An empty `COMPOSE_PROFILES=` starts nothing, and `up` never starts `system1-pull`.
  - Before a pull, the service restart-loops and stays unhealthy, logging `system1: multilingual checkpoint at aa8c91ca088e is not available (OfflineModeIsEnabled: …); run the pull step as the service user`.
  - `docker compose run --rm system1-pull` took 32 s, and the service was healthy 12 s after `up`.
  - The limits applied: UID 10001, read-only root, 2 GiB, 2 CPUs, all capabilities dropped, no published ports.
- **Production image.** The image in Appendix B builds in about a minute and is 1.7 GB; with the onnx and mcp extras, it is 1.89 GB.

### 4.7 arm64

An arm64 image builds (1.74 GB), but PyTorch segfaults under QEMU emulation, so nothing is known yet about arm64 behaviour or speed. **A native arm64 build and a run on a Pi 5 are S1 exit criteria.** Hosted arm64 CI runners can build and smoke-test the image [estimate: availability for this organisation still to be confirmed].

---

## 5. Rules of Use

Every question set and every consumer of System 1 follows these thirteen rules. Each rule comes from a measured failure in §4 unless it is marked otherwise.

1. **Advisory, with a fallback.** Every consumer still works when System 1 is down, slow, unauthorised or abstaining, and a timeout counts as an abstention. The fallback is the behaviour without System 1: escalate to on-call, or ask the operator.
2. **Rules first.** Anything a rule can decide exactly never reaches System 1: topics, alert types, the device registry, thresholds, e-stop and door events. System 1 sees only what the rules leave open. *(Evidence: with a device name or type in the state, 2–3 of the 3 alerts outside the menu were answered at confidence 0.62–0.90.)*
3. **`choice` only; gates only where "no" is safe.** Don't use `score` or `noul`, and ignore `act_probability`. A yes/no `choice` may gate something only when "no" is the safe answer, and a rule still goes in front of it. *(Evidence: gates G1–G8; severity MAE 0.88.)*
4. **Closed menus without catch-alls.** List concrete options and let abstention cover the rest; "anything else" and "other" options pull right answers away. *(Evidence: A 5/8 vs B 7/8; C 4/8 vs D 7/8; E 2/8.)* Keep menus short: upstream's 77-option result is 0.425.
5. **English descriptions, local names as aliases.** Write criteria in English, and put the names operators use in parentheses, for example `pump 2 (bomba 2)`. Bilingual descriptions scored the same (B = D = 7/8) and cost option tokens.
6. **Canonical, validated states with evidence only.** The caller builds the state from known fields, in the order listed in the set's description, trimmed and normalised. Never send a raw payload or a free-form dump. Leave out names and IDs, which act as priors, and add a field only when the set's evaluation shows that it helps. *(Evidence: rewording one alert moved it from 0.58 to 0.46; a device name alone sent a smoke alert to `connectivity` at 0.91; garbage gets confident answers.)*
7. **Pre-computed numbers.** Compare values with their limits in the caller, and pass the result as words or flags: "12 °C above its limit", `"over_limit": true`. An encoder reads digits as tokens; it doesn't do arithmetic [estimate].
8. **Per-question thresholds, from evaluation.** Each question's abstain threshold comes from its evaluation file (§10), not from intuition. `confidence` is a normalised entropy, so the same number means different things with 2 options and with 7.
9. **Budgets are errors.** Never let laya truncate. The service rejects over-budget states, options and instructions with a 422. The caller then shortens them deterministically, for example by taking a log's first line or the last N readings.
10. **Evaluate every change.** Any change to the wording, options, thresholds, checkpoint, revision, laya version or calibration bumps the set's `version`. It must then pass the set's gates, both in CI and in `p4n4 ai system1 eval`.
11. **Log every decision.** Publish every call to the decision log (§6.7), whether it was answered or abstained, used or ignored, together with the set version and revision. This is what drift checks and audits rely on.
12. **Untrusted text stays data.** Operator messages, log lines and device strings go only into the state, never into instructions or option descriptions. The only output is one of the declared keys, so injected text can at most steer a choice. Rules 1 and 2 and the confirmations bound what that choice can do.
13. **Pin everything.** Pin the image digest, the hashed Python lock, the laya version, the model commit, the checkpoint and the set version. Run offline.

---

## 6. Architecture

### 6.1 Overview

```text
 Node-RED --> alerts/<device-id>/critical --> n8n incident-escalation v2
                                                  |
   1. normalise: canonical state {alert}; device and site stay in the workflow
   2. site rules: severity, known alert types -> team                (always first)
   3. no rule matched?  POST /v1/sets/incident-triage ---------> p4n4-system1
                        <-- choice, confidence, abstain -------  laya multilingual, CPU,
                                                                 no host port, p4n4-net only
   4. publish alerts/escalated                                       (on every path)
   5. publish system1/decisions/incident-triage                      (when System 1 was asked)
   6. optional, async: Ollama writes a summary (System 2)            (never blocks step 4)
                                                  |
 system1/decisions/# --> Node-RED --> InfluxDB ai_events (system1_decision) --> Grafana
```

### 6.2 The service

`p4n4-system1` runs `system1_app.py` (Appendix A). It uses laya's own FastAPI app (`laya.serve.create_app`) with a `PinnedRouter`, plus an ASGI layer for question sets.

- **One pinned checkpoint.** It is pinned by commit and read offline from a read-only volume. The `pull` mode, run by the `system1-pull` service, is the only step that goes online.
- **Start-up checks, all fatal.** The key, the revision, the calibration file, and every question set against the model's budgets. A bad set never serves.
- **Request checks.** Bodies over 64 KiB get a 413 on every route, counted as they arrive. Then each of these is a 422: more than 8 questions or 4,000 state characters, a state over the token budget, and options or heads over theirs.
- **Memory.** Questions run two at a time, which is exact and bounds peak memory. Vocabulary tables stay in fp16, which is lossless and saves about 750 MiB.
- **One uvicorn worker.** `LAYA_THREADS` sets torch's thread count; Compose sets it from `SYSTEM1_THREADS`, default 2.

### 6.3 Question sets

A set is a versioned file in the ai stack at `config/system1/questions/<name>.json`. It reaches projects through the ai layer's `config` copy path (`lib/p4n4_lib/layers.py:57`). Appendix C has the two v2 sets.

```json
{
  "version": 2,
  "description": "What the set is for, and the state fields it expects, in order.",
  "questions": {
    "cause": {
      "type": "choice",
      "instructions": "What is the most likely cause of the alert?",
      "criteria": {"thermal": "overheating, temperature above its limit, cooling failure", "...": "..."}
    }
  },
  "thresholds": {"cause": 0.5},
  "eval": {
    "min_examples": 50,
    "gates": {"cause": {"min_coverage": 0.6, "min_answered_accuracy": 0.9, "max_confident_wrong": 0.03}}
  }
}
```

- `version` is an integer that goes up on every change. Responses and the decision log carry it.
- `thresholds` set each answer's `abstain` flag: it is true when `confidence` is below the threshold.
- `eval` is proposed for S1; the S0 wrapper ignores unknown keys.
- Examples live next to the set in `<name>.eval.jsonl`, one per line: `{"state": {...}, "expect": {"cause": "connectivity"}}`. The wrapper loads only `*.json` files, so evaluation files never serve.

### 6.4 API

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| `POST` | `/v1/sets/<name>` | Bearer | Runs a question set on `{"state": …}`. Adds `set` to the response and `abstain` to each answer |
| `POST` | `/v1/systemone` | Bearer | laya's own endpoint for ad hoc questions (harness, evaluation), with the same limits |
| `GET` | `/health` | None | Returns `{"status":"ok","loaded":["multilingual"],"device":"auto"}` |
| `GET` | `/v1/sets` | Bearer | S2: lists sets with their versions and thresholds |

Callers inside `p4n4-net` use `http://p4n4-system1:8000`. Errors are 400, 401, 404, 413, 422 or 500, with a `detail` message (Appendix D).

### 6.5 incident-escalation v2

1. **Trigger:** `alerts/+/critical`, unchanged.
2. **Normalise:** build the canonical state `{alert}`: the alert's first line, at most 500 characters. The device, its type and the site come from the payload, the topic and the device registry, and stay in the workflow for the rules and the escalation. They join the state only if the set's evaluation shows a gain, and the device name never does, because it acts as a prior (§4.4).
3. **Rules:** a site table maps alert types and devices to teams. E-stop, door, firmware and other known events are routed here and never reach System 1.
4. **System 1**, only when no rule routed the alert: `POST /v1/sets/incident-triage` with a 5 s timeout [estimate: size it from the Pi 5 measurement]. An error, a timeout, any 4xx and `abstain` all mean "no suggestion".
5. **Escalate:** always publish `alerts/escalated`. It carries:
   - the severity, from the topic and the rules;
   - either `team`, from a rule, or `likely_cause` and `likely_team`, marked as suggested;
   - the on-call recipient, in every case.
6. **Log:** publish `system1/decisions/incident-triage` whenever System 1 was asked.
7. **Explain (optional):** Ollama writes a short summary as a follow-up message. It is never on the path to step 5.

The map from cause to team is part of the rule table too (for example `electrical` → electrical maintenance), so System 1 never names a team itself.

S1 tests cover:
- System 1 stopped;
- a wrong key;
- a 422 for an oversized state;
- a timeout;
- an abstention;
- a confident answer;
- a rule match;
- an alert storm, where queued requests time out and every alert is still escalated.

### 6.6 Operator intake (S2)

1. **Filter by rules.** Empty messages, stickers, and greetings or thanks get a canned reply or nothing. Greetings and thanks are matched with a list per language plus a length limit.
2. **Ask System 1.** Send `POST /v1/sets/operator-intake` with `{text, channel}`. `small_talk` stays only as a second line of defence: it is safe, but it rarely fires.
3. **Clarify.** When `intent` or `asset` abstains, ask with buttons, for example "Which pump? 1 / 2 / 3".
4. **Confirm.** The operator confirms the draft (a ticket or a status query) before anything happens. The confirmation or correction is published as feedback (§6.7).

### 6.7 Decision log

Callers publish one message per request to `system1/decisions/<set>`, with QoS 1 and not retained, because a decision is an event rather than a state:

```json
{
  "id": "2f1c7d9e-6a0b-4d8e-9a51-0c3f5b7e2a14",
  "ts": "2026-09-23T18:20:11.512Z",
  "set": "incident-triage",
  "set_version": 2,
  "checkpoint": "multilingual",
  "revision": "aa8c91ca088e",
  "caller": "n8n-incident-escalation",
  "subject": "gw-01",
  "state_sha256": "9b1f…",
  "latency_ms": 114,
  "answers": {"cause": {"choice": "connectivity", "confidence": 1.0, "abstain": false}},
  "used": true
}
```

A Node-RED flow in the iot stack writes one point per answer to the `ai_events` bucket, which the iot layer creates (`cli/p4n4/commands/init.py:142`). The measurement is `system1_decision`:
- **Tags:** `set`, `question`, `choice`, `caller`, `checkpoint`. Each must match `^[a-z0-9_-]{1,64}$`, or the point is dropped with a warning. This keeps series cardinality bounded by the menus.
- **Fields:** `confidence` (float); `abstain` and `used` (bool); `set_version` and `latency_ms` (int); `id`, `revision`, `subject` and `state_sha256` (string).
- **Privacy:** states aren't logged by default, because operator messages can contain personal data. `state_sha256` links a decision to an evaluation example.
- **Feedback (S2):** `system1/feedback/<set>` carries `{id, question, expected}` from operator confirmations and corrections, and feeds the evaluation files (§10).

---

## 7. Integration by Repository

### 7.1 `stacks/ai`

**Compose.** In `stacks/ai/docker-compose.yml`, add the two services before the NETWORKS banner, and add `system1-models:` to the VOLUMES list. This block was run exactly as written (§4.6). It passes the stack's `docker compose config --quiet` and yamllint (relaxed, 120 columns).

```yaml
  # ------------------------------------------------------------------------------------
  # Fast Typed Decisions (System 1, laya): opt-in with COMPOSE_PROFILES=system1
  # ------------------------------------------------------------------------------------
  system1:
    image: ${SYSTEM1_IMAGE:-ghcr.io/raisga/p4n4-system1:0.1.0}
    container_name: p4n4-system1
    profiles: ["system1"]
    restart: unless-stopped
    user: "10001:10001"
    read_only: true
    tmpfs:
      - /tmp
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
    environment:
      LAYA_API_KEY: ${SYSTEM1_API_KEY:-}
      LAYA_THREADS: ${SYSTEM1_THREADS:-2}
      SYSTEM1_REVISION: ${SYSTEM1_REVISION:-aa8c91ca088ec597df95a0d1c76b3063cb2ae5e8}
      SYSTEM1_CHECKPOINT: ${SYSTEM1_CHECKPOINT:-multilingual}
    volumes:
      - system1-models:/models:ro
      - ./config/system1/questions:/config/questions:ro
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request as u; u.urlopen('http://127.0.0.1:8000/health')"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 60s
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 2g
    networks:
      - p4n4-net

  # One-shot download of the pinned checkpoint (p4n4 ai system1 pull runs this)
  system1-pull:
    image: ${SYSTEM1_IMAGE:-ghcr.io/raisga/p4n4-system1:0.1.0}
    profiles: ["system1-pull"]
    user: "10001:10001"
    read_only: true
    tmpfs:
      - /tmp
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
    environment:
      HF_HUB_OFFLINE: "0"
      SYSTEM1_REVISION: ${SYSTEM1_REVISION:-aa8c91ca088ec597df95a0d1c76b3063cb2ae5e8}
      SYSTEM1_CHECKPOINT: ${SYSTEM1_CHECKPOINT:-multilingual}
    command: ["python", "/app/system1_app.py", "pull"]
    volumes:
      - system1-models:/models
    networks:
      - p4n4-net
```

- **Opt-in:** `profiles: ["system1"]` keeps the service off unless the ai `.env` sets `COMPOSE_PROFILES=system1`, and that file is the only switch. The pull service has its own profile, so `up` never runs it; `docker compose run --rm system1-pull` does.
- **No `ports:`.** Callers reach `http://p4n4-system1:8000` on `p4n4-net`.
- **Model volume:** the server mounts it read-only. Only `system1-pull` writes it, as the same UID (§4.6).
- **Key:** `LAYA_API_KEY: ${SYSTEM1_API_KEY:-}`. An empty key makes the service refuse to start, and CI's `config --quiet` still passes without a `.env`.
- **Healthcheck:** it uses Python because the image has neither curl nor wget.

**`.env.example`.** Append this block. Every key is uncommented for two reasons: `scripts/check_env_example.py` requires every `${VAR}` used in Compose to appear, and `lib/p4n4_lib/env.py:23-40` keeps only the keys the template defines.

```dotenv
# ------------------------------------------------------------------------------
# System 1 (laya): fast typed decisions. Off unless COMPOSE_PROFILES includes system1;
# `p4n4 ai system1 enable` sets the profile and the key, then pulls the model.
# ------------------------------------------------------------------------------
COMPOSE_PROFILES=
# Bearer token for callers (n8n, Node-RED): 32+ characters, empty = service refuses to start
SYSTEM1_API_KEY=
SYSTEM1_IMAGE=ghcr.io/raisga/p4n4-system1:0.1.0
# Model commit on huggingface.co/convaiinnovations/laya: a 40-character sha, never a branch
SYSTEM1_REVISION=aa8c91ca088ec597df95a0d1c76b3063cb2ae5e8
# multilingual (default) or english
SYSTEM1_CHECKPOINT=multilingual
SYSTEM1_THREADS=2
```

Checked by merging both blocks into temporary copies of the stack files:
- `docker compose config --quiet` passes;
- `check_env_example.py` reports all 13 variables;
- yamllint passes `docker-compose.yml`.

**Pre-existing issue:** yamllint 1.38.0, the version CI installs, already fails the unmodified `.env.example` (line 17: `expected '<document start>'`), because a dotenv file isn't YAML. S1 should lint `.env.example` with a dotenv linter instead.

**Image.** `stacks/ai/system1/` holds the Dockerfile (Appendix B), `system1_app.py` and one hashed lock per architecture. CI:
- builds `linux/amd64` and `linux/arm64` natively;
- runs the regression suite against each image;
- publishes `ghcr.io/raisga/p4n4-system1:<version>` with an SBOM and provenance.

Releases pin the digest.

**Question sets.** `stacks/ai/config/system1/questions/` ships `incident-triage.json` with its evaluation file; S2 adds `operator-intake.json`. CI checks the JSON, and the image test loads the sets, because checking budgets needs the tokenizer.

**Workflow.** `stacks/ai/config/n8n/workflows/incident-escalation.json` becomes v2 (§6.5). CI's existing JSON check already covers it.

**Makefile.** Add a `system1-pull` target (`docker compose run --rm system1-pull`) next to `pull-models`.

### 7.2 `lib`

- **`p4n4_lib/secrets.py`:** add `SYSTEM1_API_KEY` to `ROTATABLE_KEYS` (lines 7–16). `rotation_value` gives 64 hex characters for keys that aren't passwords (24–26), above the 32-character minimum.
- **`p4n4_lib/layers.py`:** don't add `SYSTEM1_API_KEY` to the ai layer's `required_env_keys` (64–73), because `validate.py` (44–56) would then fail every existing ai project. The `config` copy path (57) already carries the question sets.
- **Manifest:** unchanged. `validate.py` compares `schema_version` for exact equality (22–26), so System 1 adds no manifest field. `COMPOSE_PROFILES` in the ai `.env` is the single source of truth.
- **New `p4n4_lib/system1.py`,** shared by the CLI and the harness. It covers:
  - loading question sets and checking their shape;
  - evaluation metrics and gates (§10);
  - the decision-log payload;
  - profile edits that keep other entries in `COMPOSE_PROFILES`.

### 7.3 `cli`

There is no `ai` command group yet: `cli/p4n4/cli.py:20-22` registers `ei`, `template` and `secret`. Specs §8.5 already names `p4n4 ai agent init/list`, and `system1` joins that group:

| Command | What it does |
|---------|--------------|
| `p4n4 ai system1 enable [--checkpoint multilingual] [--no-pull]` | Adds `system1` to `COMPOSE_PROFILES` and generates `SYSTEM1_API_KEY` if it is empty. Copies any missing default sets and creates or updates the n8n credential. Then pulls, starts the service and waits for health |
| `p4n4 ai system1 disable [--purge]` | Removes the profile and stops the service. `--purge` also deletes the model volume, after confirmation |
| `p4n4 ai system1 pull` | Runs `docker compose run --rm system1-pull` |
| `p4n4 ai system1 status` | Shows health, checkpoint, revision, sets and versions, and the last evaluation |
| `p4n4 ai system1 decide <set> --state FILE\|-` | Runs a set on a state and prints the answers |
| `p4n4 ai system1 eval [<set>] [--no-gates] [--fit-calibration] [--fit-thresholds]` | Runs `<set>.eval.jsonl` and prints the metrics (§10.2). Exits with code 5 (validation error, specs §8.4) when a gate fails. The `--fit-*` options print proposed temperatures or thresholds and never write them |

- **Reaching the service:** `decide` and `eval` call it through `docker compose exec -T system1 python -` with a small client. The CLI never reads the key, and no host port is needed.
- **`init`:** add `SYSTEM1_API_KEY` to `ai_env_values` (`cli/p4n4/commands/init.py:150-160`), generated with `secretutil.token()` (64 hex characters) like the other secrets, so enabling System 1 later needs no new secret.
- **`secret rotate`** (`cli/p4n4/commands/secret.py:56-94`): it generates one new value per key, shared by every stack's `.env` (62–68), writes them (85–88), then asks for `p4n4 down && p4n4 up`. For `SYSTEM1_API_KEY` it must also update the n8n credential, so add a post-rotation hook.
- **The n8n credential:**
  - It is a Header Auth credential with a fixed id (`p4n4-system1`), imported by the CLI with `n8n import:credentials` inside the n8n container. Workflows refer to it by type and name only (specs F-0.2.3, line 446).
  - To verify in S1: that importing plain `data` stores it encrypted.
  - Rejected alternative: passing the key as an n8n environment variable. That needs `$env` enabled in expressions, which would expose every variable of the n8n container to every workflow.

### 7.4 `stacks/iot`

- **Flow `system1-decision-log`:** subscribes to `system1/decisions/#`, validates the payload and tags, and writes `system1_decision` points to `ai_events` (§6.7).
- **S2, subflow:** the `p4n4-system1` subflow (U9). `SYSTEM1_API_KEY` joins the iot `.env` as a shared key, rotated with the ai one.
- **S2, dashboard:** a Grafana "System 1" dashboard showing answers per set and question, abstain rate, confidence distribution, latency and drift (§10.4).

### 7.5 `tools/emu`

The overlay template splits each board between services: Ollama gets 0.60 of the CPU and 0.55 of the memory, Letta 0.20 / 0.20 and n8n 0.20 / 0.15. `render_overlay(profile, stack, blkio_device)` (`tools/emu/p4n4_emu/overlays/generator.py:18`) renders with only `profile` and `blkio_device` (35–38), so it can't know whether `system1` is enabled.

The change:
- Add a `services` argument, read with `docker compose config --services` in the stack directory. The caller, `tools/emu/p4n4_emu/commands/up.py:84`, knows that directory.
- Add a `system1` entry with fixed limits (2 GiB, 1–2 CPUs). The other shares shrink to make room for it (§8).

### 7.6 `docs`

- **`stacks/ai-stack.md`:** a System 1 row in the Services table (no host port), an "Enabling System 1" section, and the new variables.
- **`decisions/specs.md`:**
  - F-0.2.7 "System 1 typed decisions" in §5 and in the §10.2 index;
  - dependencies in §3.1 and §3.2: F-0.1.2 → F-0.2.7 → F-0.2.2, F-0.2.3;
  - the endpoints in §8.5, the topics `system1/decisions/<set>` and `system1/feedback/<set>` in §8.6, and the variables in §8.7;
  - the §5.2 milestone text, which says "All six features" (line 580).
- **`decisions/adr/ADR-003.md`:** "An encoder classifier for bounded decisions (System 1)". It records §5's rules and the rejected alternatives: an LLM with JSON output, `score` and `noul`, and stock `laya-serve`.
- **`decisions/ai-harness.md`:** `system1_decide` and `system1_eval` rows in the §4 tool table, tier R. H1 covers all R tools, so they join H1, or the first harness release after S2.

### 7.7 Harness

`system1_decide(set, state)` and `system1_eval(set)` are tier R tools backed by `p4n4_lib.system1`. They return the answers with their `abstain` flags. Tier R means no side effects, so they don't publish to the decision log; the harness audit log already records every call (ai-harness §6).

---

## 8. Budgets per Board

| Board (emu profile) | CPUs / memory | System 1 | Suggested use |
|---------------------|---------------|----------|---------------|
| Raspberry Pi 5 (`rpi5`) | 4 / 7,168 MiB | 2 GiB, 1–2 CPUs | System 1 plus Ollama with a 1–1.5B model, or System 1 instead of Ollama on sites that only need routing [estimate; measure in S1] |
| Raspberry Pi 4 (`rpi4`) | 4 / 3,584 MiB | Not recommended | 2 GiB is 57% of the board, leaving no room for an LLM, Letta and n8n |
| Intel NUC (`nuc`) | 4 / 14,336 MiB | 2 GiB, 2 CPUs | Everything |
| MCU-class (`mcu-class`) | 1 / 256 MiB | No | — |
| Jetson | No emu profile | Unverified | CUDA wheels and arm64 untested |

On a Pi 5, expect roughly 0.7–1.1 s for one question on a short state with 2 threads [estimate: 178 ms × 4–6]. That is fine for alert routing and operator intake, but too slow for a control loop, which is out of scope anyway.

---

## 9. Security and Supply Chain

**The service (verified in §4.6):**
- **Access:**
  - A bearer key of 32+ characters is required, and the service refuses to start without one.
  - Both endpoints compare keys in constant time: the wrapper for sets, and laya itself for `/v1/systemone` (`laya/serve.py:181`).
  - `/health` is open and shows only the loaded checkpoint and the device.
- **Container:**
  - Runs as UID 10001 with a read-only root filesystem and `/tmp` on tmpfs.
  - All capabilities are dropped, with `no-new-privileges`.
  - Limited to 2 GiB and 2 CPUs.
- **Exposure:** no published port; only callers on `p4n4-net` can reach it.
- **Model:**
  - Served offline from a read-only volume and pinned to a commit.
  - Only config, weights (safetensors, never pickle) and tokenizer files are downloaded.
  - The repository's Python files are never fetched, and no remote code runs.
- **Body limits:** every body, on every route, is counted as it arrives and refused over 64 KiB (F10). In 0.3.11, laya's own 2 MiB cap (`laya/serve.py:49`) checked only a declared `Content-Length` (196–199). 0.3.12 counts streamed bodies too, but the wrapper keeps its lower limit, which doesn't depend on the laya version.
- **Telemetry:** off (`HF_HUB_DISABLE_TELEMETRY=1`). Nothing is sent at run time.

**Gap to close in S1:** at start-up, check the weights' sha256 against a value pinned with the revision. The Hub's tree listing gives it as `lfs.oid`; for `multilingual/model.safetensors` at `aa8c91c` it is `9d628fd971b700382ac6f65920a86f149777b2e748e0c955fb3b19695aa8f204`. The local cache names that blob by its Xet hash instead, so the service has to hash the file itself. That took 0.3 s on the spike host [measured].

**Untrusted input.** See rule 12. Injected text can only steer a choice among declared keys. Its reach is bounded by rules first, advisory use and confirmations. When Ollama explains a decision, the state is quoted to it as data, never as instructions.

**Data.** The decision log stores hashes, not states. Evaluation files can contain real operator messages. Keep them in the project (never in templates or issues) and redact them.

**Supply chain:**
- **laya:** a package days old, from a single vendor. Pin it with hashes. Read the diff before every bump, then rerun the regression suite and the evaluations.
- **Model and image:** the model is pinned by commit on the Hub, and releases pin the image by digest.
- **laya internals:** the wrapper relies on `_check_question`, `_to_internal`, `render_options`, `temperature_by_options`, `cfg` and `tok`, and the regression suite is the contract for them. Upstream could make these public: a strict no-truncation mode and a public question validator would remove most of the wrapper.
- **Licences:** Apache-2.0 for laya and the model card, MIT for mmBERT. Keep `typed-decisions` out until its base model is stated.

---

## 10. Calibration and Evaluation

How good System 1 is depends on a question set and a site's traffic, not on the model alone. The spike's small test sets find failure modes (§4.1). The files and gates below decide whether a set may serve.

### 10.1 Evaluation files

Each set has `<name>.eval.jsonl` next to it (§6.3), with one example per line:

```json
{"state": {"alert": "Vibration on compressor-1 at 7.1 mm/s RMS: alarm level 7.1, trip level 11."}, "expect": {"cause": "mechanical"}, "source": "history"}
{"state": {"alert": "Status changed."}, "expect": {"cause": null}, "source": "edge"}
```

- **`expect`** gives the right key for each question. It is `null` when the only right outcome is no answer, because the input fits no option or says too little. A question left out of `expect` isn't scored for that example.
- **`source`** is `history`, `feedback` or `edge`, so the metrics can be split by origin.
- **What goes in:** what reaches System 1 after the rules, including what the rules miss.
  - `history`: past alerts or messages, labelled with what the site found (the cause, the asset visited).
  - `feedback`: confirmations and corrections from `system1/feedback/<set>` (S2), joined to their decisions by `id`.
  - `edge`: uninformative alerts such as "Status changed.", and one for every device type the site has if the state carries a device type (§4.4); wordings of rule-handled events that a rule could miss; every language operators use; states near the token budget.
- **Size:** at least `eval.min_examples` (50), with at least 5 examples per option and 10 `null` ones [estimate]. With fewer, `eval` fails with exit code 5 and says why.
- **Shipped and site files.** The ai stack ships each set with a generic file, and CI runs its gates on every change (rule 10). A project extends its copy with its own examples, and `p4n4 ai system1 eval` runs the gates on that.
- **Privacy:** remove names, phone numbers and e-mail addresses from real messages. Keep the files in the project, never in templates or issues (§9).

### 10.2 Metrics and gates

The metrics are computed per question, over the examples that score it. n₁ counts the examples with a key and n₀ those with `null`:

| Metric | Definition | Default gate [estimate] |
|--------|------------|-------------------------|
| Coverage | Answered n₁ examples ÷ n₁ | ≥ 0.6 |
| Answered accuracy | Right answers ÷ answered n₁ examples | ≥ 0.9 |
| Confident-wrong rate | (Wrong answers on n₁ examples + any answer on n₀ examples) ÷ (n₁ + n₀) | ≤ 0.03 |

- **Answered** means `abstain` is false and the key isn't in the set's `eval.no_answer_keys`. Consumers treat those keys as "no suggestion"; for `incident-triage` that's `{"cause": ["unknown"]}`. Don't use them in `expect`: use `null`.
- **Output.** `eval` also prints the abstention rate on `null` examples, a confusion matrix, the split by source and the latency p50 and p95.
- **Gates.** They live in the set (`eval.gates`, §6.3), so a set that must be stricter says so. `eval` exits with code 5 when any gate fails.
- **When `null` examples fail the confident-wrong gate,** the first fix is a rule for that kind of event (rule 2), not a higher threshold.
- **`--fit-thresholds`** proposes, for each question, the lowest threshold that meets `max_confident_wrong` and `min_answered_accuracy`, and prints the coverage it gives.
  - It fits on two thirds of the examples and reports the gates on the other third, with a fixed seed, so the reported numbers don't come from the data that chose the threshold [estimate].
  - It never writes the set. A person does, and bumps the version (rule 10).

**Which confidence to threshold.** laya 0.3.12+ also returns `answer_confidence`, the top probability. Upstream calls it the calibrated one [source: laya 0.3.17 `agent.py:670-675`]. The v2 sets threshold `confidence`. On the 8 alerts of §4.4, given the alert alone, with laya 0.3.17:

| Alert | Answer | `confidence` | `answer_confidence` |
|-------|--------|-------------:|--------------------:|
| Firmware update available | `unknown` (no answer) | 0.21 | 0.42 |
| Emergency stop | `connectivity` (outside the menu) | 0.29 | 0.45 |
| Cold-room door open | `electrical` (outside the menu) | 0.36 | 0.61 |
| Battery at 20% | `sensor_fault` (right) | 0.48 | 0.52 |
| Hydraulic oil 12 °C above its limit | `thermal` (right) | 0.66 | 0.82 |
| Vibration on compressor-1 | `mechanical` (right) | 0.98 | 1.00 |
| Smoke in a cabinet; `ECONNREFUSED` | Right | 1.00 | 1.00 |

- **`confidence` separates better here.** It ranks every right answer above every alert outside the menu, so any threshold from 0.36 to 0.48 splits them. The default 0.5 gives 4 right answers and no wrong ones. `answer_confidence` ranks the door alert (0.61) above the right battery answer (0.52), so no threshold on it can do both.
- **The state matters more than the metric.** With the placeholder device name of §4.4 in the state, both metrics ranked the alerts identically. Only a threshold above the firmware notice's score, 0.90 on `confidence` or 0.96 on `answer_confidence`, then refused all three alerts outside the menu. Evidence-only states (rule 6) and rules for known events (rule 2) come first.
- **Proposal.** `--fit-thresholds` fits both metrics, and the set records its choice as `"threshold_on": "confidence"` or `"answer_confidence"`, with `confidence` as the default. This needs laya ≥ 0.3.12.

### 10.3 Calibration

- **What it is.** laya divides each question's option logits by a temperature T before the softmax, so pᵢ′ = pᵢ^(1/T) / Σⱼ pⱼ^(1/T). T > 1 flattens the probabilities and T < 1 sharpens them. The top option never changes: calibration moves answers across thresholds, but never changes a choice.
- **Why.** `multilingual` ships no temperatures (T = 1). Upstream measured an expected calibration error of 0.314 for it, and 0.106 after refitting per bucket [upstream].
- **The site file.**
  - `SYSTEM1_CALIBRATION` names a JSON file of temperatures per bucket, for example `{"choice:2": 1.9, "choice:6-10": 1.4}`.
  - The buckets are those of §2.3, and the values must lie in [0.5, 5]. Anything else stops start-up (F8). The hook was verified in §4.4.
  - In Compose, S3 adds `SYSTEM1_CALIBRATION: ${SYSTEM1_CALIBRATION:-}`, where empty means none, with a matching `.env.example` line, and mounts the file read-only.
- **Fitting.** `eval --fit-calibration` fits one T per bucket over [0.5, 5], minimising the negative log-likelihood of the expected keys on the non-null examples.
  - It first undoes any temperature T₀ already in the site file: raising the returned probabilities to the power T₀ and renormalising gives the uncalibrated ones.
  - laya rounds probabilities to 4 decimals [source: laya 0.3.17 `agent.py:683`], so the fit clips them at 5·10⁻⁵ [estimate].
- **Order.** Calibrate first, then fit thresholds, because thresholds depend on the calibrated probabilities. A calibration change applies to every set, so every set is evaluated again and its version bumped (rule 10).
- **Buckets are shared.** A bucket covers every question with its type and option count, in every set. For example, `incident-triage.cause` (7 options) shares `choice:6-10` with any other question of 6 to 10 options. S2 adds per-question temperatures to the set, applied by the wrapper.
- **Per language.** laya 0.3.12+ accepts `lang_temperatures`, a table per language, but applies it only when it knows the language: from a `lang` argument, or from detection when no model is named. laya-serve passes neither, and the wrapper always names its model, so these tables never apply today [source: laya 0.3.17 `router.py:532-534`, `serve.py:276`]. For multi-language sites, S2 evaluates them. The wrapper would then detect the language (`laya.detect_language`) or take it from the caller, and pass `lang`.

### 10.4 Drift

The decision log (§6.7) is the input, and the S2 dashboard shows it. S3 adds a Grafana alert rule per set and question, which compares the last 7 days with the 28 days before [estimate]. It fires when:
- the abstention rate doubles;
- any option's share of the answers moves by more than 20 points;
- latency p95 goes above half the caller's timeout.

A drift alert is a warning to run the evaluation again with recent feedback, never an automatic change. If the gates fail on recent data, raise the threshold or remove the set. Consumers then fall back (rule 1) until a new version passes.

### 10.5 Fine-tuning (S4)

Upstream presents laya as "a fast base to specialise". It publishes a fine-tuning notebook and the `LocalLLaMA/typed-decisions` dataset (§2.5). S4 happens only if S1–S3 show that the base checkpoint is the limit, for example when no threshold meets the confident-wrong gate with coverage above 0.6.
- **Licences first.** Check the notebook's and the dataset's licences, and keep the chain back to mmBERT (MIT) and laya (Apache-2.0) intact.
- **Train off the device,** on a workstation or a GPU runner, never on the edge board.
- **Data.** Use site evaluation files and feedback, and hold out a fixed evaluation split that training never sees.
- **Ship it like the base checkpoint.**
  - Publish it in a repository or artifact store that p4n4 controls, pinned by commit and sha256, with a model card that lists the data sources and metrics.
  - It must beat the base checkpoint on the same held-out file, and pass the same regression suite and gates.

---

## 11. Phases

| Phase | Use cases | Deliverables | Status |
|-------|-----------|--------------|--------|
| **S0: spike** | — | This document; the wrapper, image and Compose service; the v2 sets; the regression suite | Done; re-checked on laya 0.3.17 |
| **S1: service and incident routing** | U1 | The image and its CI; the Compose services; the CLI; `incident-escalation` v2; the decision log; the docs | Next |
| **S2: intake and tooling** | U2, U8, U9 | The intake workflow and feedback; the harness tools; the Node-RED subflow; the dashboard; `GET /v1/sets`; per-question temperatures; the emu overlay | — |
| **S3: more sets** | U3–U7 | New sets with evaluation files; site calibration; drift alerts | — |
| **S4: specialise** | Any | Fine-tuning, only if the evaluations call for it (§10.5) | — |

### 11.1 S1 scope

- **Image.** `stacks/ai/system1/` holds the Dockerfile, the wrapper and a `--require-hashes` lock per architecture, and the base image is pinned by digest. CI builds amd64 and arm64 natively, runs the regression suite on each and publishes to GHCR with an SBOM and provenance (§7.1).
  - **laya version:** start from the latest release that passes the suite and the evaluations, which is 0.3.17 today.
- **Wrapper:** the S1 additions in Appendix A, including the sha256 check (§9).
- **Stack:** the Compose services, the `.env.example` block and the `system1-pull` Makefile target (§7.1).
- **Routing:** `incident-triage` v2 with its evaluation file, and `incident-escalation` v2 with the tests listed in §6.5.
- **Decision log:** the iot flow that writes `system1_decision` points (§7.4).
- **Library and CLI** (§7.2, §7.3):
  - `p4n4_lib/system1.py` and the `p4n4 ai system1` commands;
  - `SYSTEM1_API_KEY` in `ROTATABLE_KEYS`, with a rotation hook;
  - the n8n credential.
- **Docs:** the ai-stack page, F-0.2.7 in the specs and ADR-003 (§7.6).

### 11.2 Exit criteria

**S1:**
- On a native Raspberry Pi 5, `incident-triage` requests of up to 200 input tokens (Appendix D's is 150) have a p95 latency of at most 1.5 s, within the 2 GiB limit.
- The regression suite passes on native amd64 and arm64.
- The `incident-triage` gates pass on at least 50 examples from a real site.
- With System 1 stopped, slow, unauthorised or abstaining, every critical alert is still escalated.
- A modified weights file stops the service at start-up.

**S2:**
- The intake gates pass on a site's file, in every language the site uses.
- The subflow follows the failure contract of F-0.2.2.
- The harness tools appear in the ai-harness tool table.

**S3:** each new set passes its gates, and the drift alerts fire when a recorded log with a shifted mix is replayed.

**S4:** a fine-tuned checkpoint beats the base checkpoint on a held-out site file, with the same pins.

---

## 12. Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| **Upstream churn.** Nine releases in 13.5 hours, and the wrapper uses laya internals (§9) | High | Medium | Pin with hashes, and treat the regression suite as the contract. Read every diff. Ask upstream for a strict no-truncation mode and a public question validator |
| **Phrasing sensitivity.** Rewording moves answers across thresholds (§4.4) | High | Medium | Canonical states, versioned sets and gates on every change (rules 6, 8 and 10) |
| **Confidently wrong answers** on alerts outside the menu or with too little text, often steered by fields such as the device name (F11) | High | Medium, because System 1 is advisory | Rules first, evidence-only states, `null` examples in every evaluation file and the confident-wrong gate. Nothing safety-related depends on System 1 (rules 1, 2 and 6) |
| **Automation bias.** People treat a suggestion as a decision | Medium | Medium | Suggestions are marked as suggested, the on-call recipient is always notified, and every intake draft needs confirmation (§6.5, §6.6) |
| **Pi 5 performance** is only estimated (§8) | Medium | Medium | An S1 exit criterion, callers' timeouts and a fixed 1–2 CPU budget |
| **arm64 is untested.** PyTorch segfaults under emulation (§4.7) | Medium | High | Native arm64 CI and a Pi 5 run in S1 |
| **Memory next to Ollama** on a 7 GiB board | Medium | Medium | The 2 GiB limit, the emu overlay (§7.5) and the board guidance (§8) |
| **Upstream availability.** The model repository is days old, and could move, change its licence or disappear | Low | High | The pinned commit and sha256. Whether to mirror the pinned files is an open question (§13) |
| **`typed-decisions` licence** isn't stated | Low | Medium | Not shipped. From S1, the wrapper refuses it (Appendix A) |
| **One key for several callers:** n8n, and Node-RED from S2 | Medium | Low | Reachable only on `p4n4-net`, rotated together, and the outputs are advisory. Per-caller keys would need the wrapper to authenticate `/v1/systemone` itself |
| **Personal data** in evaluation files | Medium | Medium | Redaction, files kept in the project, and hashes in the decision log (§6.7, §9) |

---

## 13. Open Questions

1. **Pi 5 latency and memory.** What are p50 and p95 for `incident-triage` on a native Pi 5, and what timeout should n8n use? S1 measures them.
2. **arm64 CI.** Are hosted arm64 runners available to the organisation, or does S1 need a self-hosted Pi?
3. **n8n credentials.** Does `n8n import:credentials` encrypt plain `data` when it stores it (§7.3)?
4. **Threshold metric.** Should each set threshold `confidence` or `answer_confidence` (§10.2)?
5. **Per-language calibration.** Are `lang_temperatures` worth the extra step on multi-language sites (§10.3)?
6. **The sha256 pin.** Should it live in `.env` next to the revision, or in a manifest built into the image?
7. **Fields in the state.** On the 8 alerts, the alert alone did best, and a device name acted as a prior (§4.4). Is `{alert}` right for every site, or do some sites gain from a `device_type` in their evaluations?
8. **Ollama next to System 1 on a Pi 5.** Which Ollama model sizes still fit alongside n8n and Letta (§8)?
9. **Mirroring.** Should p4n4 keep a copy of the pinned model files, and where?
10. **The dotenv linter.** Which linter replaces yamllint for `.env.example` (§7.1)?

---

## Appendix A. Reference Wrapper

This is `system1_app.py` exactly as tested in §4, on laya 0.3.11 and 0.3.17. It is configured through environment variables:

| Variable | Default | Meaning |
|----------|---------|---------|
| `LAYA_API_KEY` | Required | Bearer key of 32+ characters; without it the service exits. Compose sets it from `SYSTEM1_API_KEY` |
| `SYSTEM1_REVISION` | Required | A 40-character commit sha of `SYSTEM1_REPO` |
| `SYSTEM1_REPO` | `convaiinnovations/laya` | Hub repository |
| `SYSTEM1_CHECKPOINT` | `multilingual` | `multilingual` or `english`. `typed-decisions` also loads today, but is excluded (§2.5) |
| `SYSTEM1_QUESTIONS` | `/config/questions` | Folder of `<name>.json` question sets |
| `SYSTEM1_CALIBRATION` | Unset | JSON file of `{bucket: T}` with T in [0.5, 5] (§10.3) |
| `SYSTEM1_MAX_QUESTIONS` | `8` | Questions per request |
| `SYSTEM1_MAX_STATE_CHARS` | `4000` | State characters per request |
| `SYSTEM1_MAX_BODY_BYTES` | `65536` | Body bytes per request, on every route |
| `SYSTEM1_QUESTION_CHUNK` | `2` | Questions per forward pass |
| `SYSTEM1_FP16_EMBEDDINGS` | `1` | Keeps the vocabulary tables in fp16 |
| `LAYA_THREADS` | `0` (torch's default) | torch threads. Compose sets it from `SYSTEM1_THREADS`, default 2 |
| `LAYA_DEVICE` | Unset (laya chooses) | Passed to laya's router; `/health` then reports `auto` |
| `LAYA_HOST`, `LAYA_PORT`, `LAYA_LOG_LEVEL` | `0.0.0.0`, `8000`, `info` | uvicorn settings inside the container, which publishes no port |

The image (Appendix B) also sets `HF_HOME=/models`, `HF_HUB_OFFLINE=1` (the pull service sets 0), `HF_HUB_DISABLE_TELEMETRY=1` and `USE_TF=0`.

```python
"""p4n4 System 1 service: laya-serve's HTTP API with a pinned revision, one resident checkpoint,
request limits sized for small hosts, no silent truncation, server-side question sets and
optional site calibration. `python system1_app.py pull` downloads the checkpoint and exits."""
import hmac
import json
import os
import sys

import torch
import uvicorn
from huggingface_hub import snapshot_download
from laya import Router
from laya.common import QTYPES, TEMP_MAX, TEMP_MIN, render_options
from laya.serve import create_app

REPO = os.environ.get("SYSTEM1_REPO", "convaiinnovations/laya")
REVISION = os.environ.get("SYSTEM1_REVISION", "")  # a commit sha, never a branch
CHECKPOINT = os.environ.get("SYSTEM1_CHECKPOINT", "multilingual")
SUBFOLDER = {"english": None, "multilingual": "multilingual", "typed-decisions": "typed-decisions"}[CHECKPOINT]
FILES = ("rl_agent_config.json", "model.safetensors", "tokenizer/*", "encoder/*")
MAX_QUESTIONS = int(os.environ.get("SYSTEM1_MAX_QUESTIONS", "8"))
MAX_STATE_CHARS = int(os.environ.get("SYSTEM1_MAX_STATE_CHARS", "4000"))
MAX_BODY_BYTES = int(os.environ.get("SYSTEM1_MAX_BODY_BYTES", "65536"))
CHUNK = int(os.environ.get("SYSTEM1_QUESTION_CHUNK", "2"))
FP16_EMBEDDINGS = os.environ.get("SYSTEM1_FP16_EMBEDDINGS", "1") == "1"
QUESTIONS_DIR = os.environ.get("SYSTEM1_QUESTIONS", "/config/questions")
OPTION_TOKENS = 48  # laya's build_sequence reads at most this many tokens of each option
BUCKETS = {"%s:%s" % (t, n) for t in QTYPES for n in ("2", "3-5", "6-10", "11+")}  # laya's temp_bucket


class PinnedRouter(Router):
    """One resident checkpoint, bounded work per request, no silent truncation.

    Each question is encoded as its own sequence with the full state, so cost and peak memory
    grow with questions x state tokens. Questions are independent, so chunking them is exact.
    """

    tok = None
    agent = None
    head_budget = 0
    state_budget = 0

    def check_question(self, qid, qdef):
        """Reject a question that laya would answer from a cut-down copy of its own text."""
        self.agent._check_question(qid, qdef)
        q = self.agent._to_internal(qdef)
        opts = render_options(q)
        sizes = [len(self.tok(" " + o, add_special_tokens=False)["input_ids"]) for o in opts]
        long = [o[:40] for o, n in zip(opts, sizes) if n > OPTION_TOKENS]
        if long:
            raise ValueError("question %r: options over %d tokens would be cut: %s" % (qid, OPTION_TOKENS, long))
        head = len(self.tok("%s question: %s" % (q["t"], q["ins"]), add_special_tokens=False)["input_ids"])
        used = head + sum(n + 1 for n in sizes)  # each option is [MASK] + its tokens
        if used > self.head_budget:
            raise ValueError("question %r: instructions and options are %d tokens; the %s checkpoint reads "
                             "at most %d" % (qid, used, CHECKPOINT, self.head_budget))

    def predict(self, state, questions, model=None, **kw):
        if not isinstance(questions, dict) or not isinstance(state, (str, dict, list)):
            return super().predict(state, questions, model=CHECKPOINT, **kw)  # laya's own 422
        text = state if isinstance(state, str) else json.dumps(state, ensure_ascii=False)
        if len(questions) > MAX_QUESTIONS or len(text) > MAX_STATE_CHARS:
            raise ValueError("system1 accepts at most %d questions and %d state characters per request"
                             % (MAX_QUESTIONS, MAX_STATE_CHARS))
        for qid, qdef in questions.items():
            self.check_question(qid, qdef)
        tokens = len(self.tok(text, add_special_tokens=False)["input_ids"])
        if tokens > self.state_budget:
            # laya would cut the state to fit and still answer, from evidence the caller never sees
            raise ValueError("state is %d tokens; the %s checkpoint reads at most %d, send a shorter "
                             "canonical state" % (tokens, CHECKPOINT, self.state_budget))
        ids, out = list(questions), None
        for i in range(0, len(ids), CHUNK):
            part = super().predict(state, {k: questions[k] for k in ids[i:i + CHUNK]}, model=CHECKPOINT, **kw)
            if out is None:
                out = part
            else:
                out["answers"].update(part["answers"])
                out["usage"]["input_tokens"] += part["usage"]["input_tokens"]
        return out


class HalfEmbedding(torch.nn.Module):
    """A vocabulary table stored in fp16 that returns fp32 rows (lookup only)."""

    def __init__(self, weight, padding_idx=None):
        super().__init__()
        self.register_buffer("weight", weight)
        self.padding_idx = padding_idx

    def forward(self, ids):
        return torch.nn.functional.embedding(ids, self.weight, self.padding_idx).float()


def use_half_embeddings(model, weights_file, min_rows=50000):
    """Keep large vocabulary tables in fp16, read straight from the checkpoint.

    The checkpoints are stored in F16, so this is lossless. Each fp32 table is released before
    its fp16 copy is read, so start-up never peaks above the plain fp32 footprint.
    """
    from safetensors import safe_open
    names = [n for n, m in model.named_modules()
             if isinstance(m, torch.nn.Embedding) and m.num_embeddings >= min_rows]
    with safe_open(weights_file, framework="pt") as f:
        for name in names:
            parent_name, _, attr = name.rpartition(".")
            parent = model.get_submodule(parent_name)
            padding_idx = getattr(parent, attr).padding_idx
            setattr(parent, attr, torch.nn.Identity())
            setattr(parent, attr, HalfEmbedding(f.get_tensor(name + ".weight").to(torch.float16), padding_idx))


def load_question_sets(router, folder):
    """Versioned question sets from <folder>/<name>.json, each checked against the model at start."""
    sets = {}
    for fn in sorted(os.listdir(folder)) if os.path.isdir(folder) else ():
        if not fn.endswith(".json"):
            continue
        name = fn[:-5]
        with open(os.path.join(folder, fn), encoding="utf-8") as f:
            spec = json.load(f)
        qset = {"name": name, "version": int(spec["version"]), "questions": spec["questions"],
                "thresholds": {k: float(v) for k, v in spec.get("thresholds", {}).items()}}
        bad = sorted(k for k, v in qset["thresholds"].items() if k not in qset["questions"] or not 0 <= v <= 1)
        if bad:
            sys.exit("system1: question set %s: thresholds %s must name a question and lie in [0, 1]" % (name, bad))
        try:
            router.predict({}, qset["questions"])
        except ValueError as e:
            sys.exit("system1: question set %s: %s" % (name, e))
        sets[name] = qset
    return sets


async def _reply(send, status, obj):
    body = json.dumps(obj).encode()
    await send({"type": "http.response.start", "status": status,
                "headers": [(b"content-type", b"application/json"), (b"content-length", str(len(body)).encode())]})
    await send({"type": "http.response.body", "body": body})


async def _read_body(receive):
    """The request body, or None as soon as it passes MAX_BODY_BYTES. laya-serve checks only a
    declared Content-Length, so a chunked body would otherwise be buffered and parsed in full."""
    body, more = b"", True
    while more:
        msg = await receive()
        body, more = body + msg.get("body", b""), msg.get("more_body", False)
        if len(body) > MAX_BODY_BYTES:
            return None
    return body


def _replay(body, receive):
    pending = [{"type": "http.request", "body": body, "more_body": False}]

    async def inner_receive():
        return pending.pop() if pending else await receive()

    return inner_receive


class QuestionSets:
    """POST /v1/sets/<name> {"state": ...} runs a server-side question set through laya's own
    /v1/systemone handler (same auth, limits and single worker). The response names the set and
    version, and marks answers whose confidence is under the set's threshold as abstained.
    Every request body, on any route, is counted as it arrives and cut off at MAX_BODY_BYTES."""

    def __init__(self, app, sets, api_key):
        self.app, self.sets, self.key = app, sets, ("Bearer " + api_key).encode()

    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            return await self.app(scope, receive, send)
        if not scope["path"].startswith("/v1/sets/"):
            body = await _read_body(receive)
            if body is None:
                return await _reply(send, 413, {"detail": "request body too large"})
            return await self.app(scope, _replay(body, receive), send)
        if not hmac.compare_digest(dict(scope["headers"]).get(b"authorization", b""), self.key):
            return await _reply(send, 401, {"detail": "invalid or missing bearer token"})
        qset = self.sets.get(scope["path"][len("/v1/sets/"):])
        if qset is None or scope["method"] != "POST":
            return await _reply(send, 404, {"detail": "unknown question set"})
        body = await _read_body(receive)
        if body is None:
            return await _reply(send, 413, {"detail": "request body too large"})
        try:
            state = json.loads(body)["state"]
        except (ValueError, KeyError, TypeError):
            return await _reply(send, 400, {"detail": "request body must be an object with a 'state' field"})
        inner = json.dumps({"state": state, "questions": qset["questions"]}).encode()
        scope = dict(scope, path="/v1/systemone", raw_path=b"/v1/systemone",
                     headers=[(k, v) for k, v in scope["headers"] if k != b"content-length"]
                     + [(b"content-length", str(len(inner)).encode())])
        start, chunks = {}, []

        async def inner_send(msg):
            if msg["type"] == "http.response.start":
                start.update(msg)
                return
            chunks.append(msg.get("body", b""))
            if msg.get("more_body"):
                return
            out = b"".join(chunks)
            if start["status"] == 200:
                result = json.loads(out)
                result["set"] = {"name": qset["name"], "version": qset["version"]}
                for qid, answer in result["answers"].items():
                    answer["abstain"] = answer["confidence"] < qset["thresholds"].get(qid, 0.0)
                out = json.dumps(result, ensure_ascii=False).encode()
            headers = [(k, v) for k, v in start.get("headers", []) if k != b"content-length"]
            await send({"type": "http.response.start", "status": start["status"],
                        "headers": headers + [(b"content-length", str(len(out)).encode())]})
            await send({"type": "http.response.body", "body": out})

        await self.app(scope, _replay(inner, receive), inner_send)


def snapshot():
    """Local path of the pinned checkpoint, downloaded first unless HF_HUB_OFFLINE=1."""
    if len(REVISION) != 40:
        sys.exit("system1: SYSTEM1_REVISION must be a 40-character commit sha")
    prefix = f"{SUBFOLDER}/" if SUBFOLDER else ""
    try:
        return snapshot_download(REPO, revision=REVISION, allow_patterns=[prefix + f for f in FILES])
    except OSError as e:  # hub errors: offline and not pulled, unknown revision, no network
        sys.exit("system1: %s checkpoint at %s is not available (%s: %s); run the pull step as the service user"
                 % (CHECKPOINT, REVISION[:12], type(e).__name__, (str(e).splitlines() or [""])[0][:160]))


def build_app():
    if len(os.environ.get("LAYA_API_KEY", "")) < 32:
        # laya-serve runs without auth when the key is unset; this service never does
        sys.exit("system1: LAYA_API_KEY must be set (32+ chars) when the system1 profile is enabled")
    threads = int(os.environ.get("LAYA_THREADS") or 0)
    if threads > 0:
        torch.set_num_threads(threads)
    local = snapshot()
    router = PinnedRouter(models={CHECKPOINT: (local, SUBFOLDER)}, default=CHECKPOINT,
                          max_loaded=1, device=os.environ.get("LAYA_DEVICE") or None)
    agent = router.load(CHECKPOINT)
    router.tok, router.agent = agent.tok, agent
    router.head_budget = agent.cfg.get("head_max_len", 192)
    # laya's build_sequence is [CLS] head [SEP] options [SEP] state [SEP]; check_question keeps
    # head + options within head_max_len, so every question leaves at least this much for the state
    router.state_budget = agent.cfg.get("max_len", 512) - router.head_budget - 4
    if FP16_EMBEDDINGS:
        use_half_embeddings(agent.model, os.path.join(local, SUBFOLDER or "", "model.safetensors"))
    calibration = os.environ.get("SYSTEM1_CALIBRATION")
    if calibration:
        with open(calibration, encoding="utf-8") as f:
            fitted = json.load(f)  # {"choice:2": 1.9, ...}: site-fitted temperatures per laya bucket
        for bucket, t in fitted.items():
            if (bucket not in BUCKETS or isinstance(t, bool) or not isinstance(t, (int, float))
                    or not TEMP_MIN <= t <= TEMP_MAX):
                sys.exit("system1: calibration %s=%r: use a bucket in %s with a temperature in [%g, %g]"
                         % (bucket, t, sorted(BUCKETS), TEMP_MIN, TEMP_MAX))
            agent.temperature_by_options[bucket] = float(t)
    sets = load_question_sets(router, QUESTIONS_DIR)
    return QuestionSets(create_app(router), sets, os.environ["LAYA_API_KEY"])


if __name__ == "__main__":
    if sys.argv[1:] == ["pull"]:
        # run once online, as the service user: huggingface_hub keeps its offline index of the
        # revision (trees/<sha>.json) readable by its owner only
        print("system1: %s checkpoint ready in %s" % (CHECKPOINT, snapshot()))
    else:
        uvicorn.run(build_app(), host=os.environ.get("LAYA_HOST", "0.0.0.0"),
                    port=int(os.environ.get("LAYA_PORT", "8000")),
                    log_level=os.environ.get("LAYA_LOG_LEVEL", "info"))
```

**To add in S1:**
- a sha256 check of the weights before they load, against a value pinned with the revision (§9);
- `threshold_on` (§10.2), which needs laya ≥ 0.3.12;
- one structured log line per request with the route, set and version, status, latency, input tokens and the state's sha256, but never the state;
- clear start-up errors for an unknown `SYSTEM1_CHECKPOINT` (today a `KeyError` at import) and a missing calibration file (today a traceback);
- refusing `typed-decisions` until its licence is stated.

**To add in S2 and later:**
- `GET /v1/sets` (§6.4);
- per-question temperatures (§10.3), and `lang` for `lang_temperatures` if the S2 evaluation shows a gain;
- replacing the laya internals with public APIs once upstream has them (§9).

---

## Appendix B. Image

`stacks/ai/system1/Dockerfile`, as built and tested in §4.6:

```dockerfile
# p4n4 System 1: laya 0.3.11 behind system1_app.py (CPU, non-root, offline at run time)
FROM python:3.12-slim
ENV PIP_NO_CACHE_DIR=1 PIP_DISABLE_PIP_VERSION_CHECK=1 PYTHONUNBUFFERED=1 PYTHONDONTWRITEBYTECODE=1 \
    USE_TF=0 HF_HOME=/models HF_HUB_OFFLINE=1 HF_HUB_DISABLE_TELEMETRY=1
COPY constraints.txt /tmp/constraints.txt
RUN pip install --index-url https://download.pytorch.org/whl/cpu "torch==2.14.0" \
 && pip install -c /tmp/constraints.txt "laya[serve]==0.3.11" \
 && rm /tmp/constraints.txt
# pull and serve must run as the same user: huggingface_hub writes its offline index with mode 0600
RUN groupadd --system --gid 10001 system1 \
 && useradd --system --uid 10001 --gid 10001 --no-create-home --home-dir /nonexistent \
      --shell /usr/sbin/nologin system1 \
 && mkdir -p /models /config/questions && chown 10001:10001 /models
COPY system1_app.py /app/system1_app.py
USER 10001:10001
EXPOSE 8000
CMD ["python", "/app/system1_app.py"]
```

The installed packages, from `pip freeze` in the laya 0.3.11 image:

```text
annotated-doc==0.0.5
annotated-types==0.8.0
anyio==4.15.1
certifi==2026.7.22
click==8.5.0
fastapi==0.141.1
filelock==3.32.3
fsspec==2026.7.0
h11==0.16.0
hf-xet==1.6.0
httpcore==1.0.9
httpx==0.28.1
huggingface_hub==1.32.0
idna==3.20
Jinja2==3.1.6
laya==0.3.11
markdown-it-py==4.2.0
MarkupSafe==3.0.3
mdurl==0.1.2
mpmath==1.3.0
networkx==3.6.1
numpy==2.5.3
packaging==26.3
pydantic==2.13.5
pydantic_core==2.46.5
Pygments==2.21.0
python-multipart==0.0.32
PyYAML==6.0.3
regex==2026.9.10
rich==15.0.0
safetensors==0.8.0
setuptools==78.1.0
shellingham==1.5.4
starlette==1.7.0
sympy==1.14.0
tokenizers==0.23.2
torch==2.14.0+cpu
tqdm==4.70.1
transformers==5.17.0
typer==0.27.2
typing-inspection==0.4.4
typing_extensions==4.16.0
uvicorn==0.53.0
```

- **Constraints.** `constraints.txt` is the spike's full freeze, which also pins the onnx and mcp extras. The image installs only `laya[serve]`, so it resolves to the list above.
- **laya 0.3.17.** That build differs only in the `laya` pin, in the Dockerfile and in the freeze. It passed the regression suite and gave identical answers (§2.6).
- **S1** replaces the constraints with a `--require-hashes` lock per architecture, pins `python:3.12-slim` by digest, and adds an SBOM and provenance when publishing (§7.1).

---

## Appendix C. Question Sets v2

`config/system1/questions/incident-triage.json`:

```json
{
  "version": 2,
  "description": "Probable cause of an alert on alerts/<device>/<type>, used to route it to a team. Severity is not asked: it comes from the topic and site rules. State: {alert}, the alert's first line. Add a field only when the evaluation shows it helps, and not the device name: it acts as a prior.",
  "questions": {
    "cause": {
      "type": "choice",
      "instructions": "What is the most likely cause of the alert?",
      "criteria": {
        "thermal": "overheating, temperature above its limit, cooling failure",
        "mechanical": "vibration, wear, leak, crack, blockage, broken part",
        "electrical": "power loss, voltage, current, fuse, relay, motor drive",
        "connectivity": "device offline, timeouts, refused connections, network or broker down",
        "sensor_fault": "implausible, stuck or missing readings, low battery, calibration drift",
        "process": "setpoint, recipe, material or operator action",
        "unknown": "cannot tell from the alert"
      }
    }
  },
  "thresholds": {
    "cause": 0.5
  }
}
```

`config/system1/questions/operator-intake.json` (S2):

```json
{
  "version": 2,
  "description": "Operator messages in any language. State fields, in this order: text, channel. Criteria are in English; asset aliases list the local names.",
  "questions": {
    "small_talk": {
      "type": "choice",
      "instructions": "Is the message only a greeting or thanks?",
      "criteria": {
        "yes": "only a greeting or thanks, nothing to do",
        "no": "it reports a problem or asks for something"
      }
    },
    "intent": {
      "type": "choice",
      "instructions": "What does the operator want?",
      "criteria": {
        "report_fault": "reports a fault, damage, noise, leak or abnormal behaviour",
        "status_request": "asks for the current status or a reading",
        "maintenance_request": "asks for maintenance, a spare part or a visit"
      }
    },
    "asset": {
      "type": "choice",
      "instructions": "Which asset is the message about?",
      "criteria": {
        "pump-1": "pump 1 (bomba 1)",
        "pump-2": "pump 2 (bomba 2)",
        "pump-3": "pump 3 (bomba 3)",
        "compressor-1": "the air compressor (el compresor)",
        "none": "no specific asset (ningún equipo)"
      }
    }
  },
  "thresholds": {
    "small_talk": 0.5,
    "intent": 0.5,
    "asset": 0.6
  }
}
```

- **`unknown`** is a concrete option ("cannot tell from the alert"), not a catch-all. Given the alert alone, it won on the firmware notice, the empty state and 2 of 4 lorem-ipsum wordings (P 0.38–0.51, all abstained), but other garbage got other options (§4.4). On the 5 clear alerts, its probability stayed under 0.04 [measured]. The rule table maps it to no team, so it is an `eval.no_answer_keys` entry (§10.2).
- **Asset aliases** follow rule 5: English descriptions, with local names in parentheses.
  - `none` caught the 5 greetings and thanks that name no asset, at 0.92–0.97.
  - Both asset abstentions were status questions about the compressor, in Spanish and Hindi. The S2 evaluation will show whether more aliases help.
- **Thresholds** are placeholders until each site's evaluation (rule 8). `asset` starts higher (0.6), because a wrong asset sends someone to the wrong machine [estimate].
- **`small_talk`** stays only as a second line of defence behind the rules (§6.6).
- **The `eval` block** of §6.3 is added in S1. The S0 wrapper ignores it.

Sample evaluation lines for `operator-intake`:

```json
{"state": {"text": "La bomba 3 hace un ruido raro desde esta mañana y la presión está bajando.", "channel": "telegram"}, "expect": {"small_talk": "no", "intent": "report_fault", "asset": "pump-3"}, "source": "history"}
{"state": {"text": "Gracias, ya quedó todo bien.", "channel": "telegram"}, "expect": {"small_talk": "yes", "intent": null, "asset": "none"}, "source": "edge"}
```

The second line is a thank-you that got past the rules, so `intent` must not answer. In the spike, it answered `status_request` at 0.89 (§4.4), which is the kind of error the confident-wrong gate catches.

---

## Appendix D. Calling System 1

**From n8n.** Use an HTTP Request node at typeVersion 4.2, like the existing workflows. The node below is a sketch: S1 checks its parameters with the round-trip import in specs F-0.2.3.
- It uses the Header Auth credential that the CLI imports (§7.3), referenced by type and name.
- Its error output leads to the "no suggestion" path (§6.5).

```json
{
  "parameters": {
    "method": "POST",
    "url": "http://p4n4-system1:8000/v1/sets/incident-triage",
    "authentication": "genericCredentialType",
    "genericAuthType": "httpHeaderAuth",
    "sendBody": true,
    "specifyBody": "json",
    "jsonBody": "={{ JSON.stringify({ state: $json.state }) }}",
    "options": {
      "timeout": 5000
    }
  },
  "name": "System 1 Likely Cause",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "onError": "continueErrorOutput",
  "credentials": {
    "httpHeaderAuth": {
      "id": "p4n4-system1",
      "name": "p4n4 System 1"
    }
  }
}
```

**From the CLI or a shell.** The client runs inside the container, which already has the key, so the key never leaves it and no port is needed (§7.3). From `stacks/ai`:

```bash
docker compose exec -T system1 python - <<'EOF'
import json, os, urllib.request

state = {"alert": "connect ECONNREFUSED 172.18.0.5:1883, no telemetry received for 10 minutes"}
req = urllib.request.Request(
    "http://127.0.0.1:8000/v1/sets/incident-triage",
    data=json.dumps({"state": state}).encode(),
    headers={"Content-Type": "application/json", "Authorization": "Bearer " + os.environ["LAYA_API_KEY"]})
answer = json.load(urllib.request.urlopen(req, timeout=10))["answers"]["cause"]
print(answer["choice"], answer["confidence"], "abstain" if answer["abstain"] else "use")
EOF
```

It prints `connectivity 1.0 use` [measured].

**Responses** [measured, laya 0.3.17]. laya 0.3.11 returns the same without `answer_confidence`.

`POST /v1/sets/incident-triage` with the state above, in 99 ms:

```json
{
  "model": "laya-rl-agent",
  "answers": {
    "cause": {
      "type": "choice",
      "choice": "connectivity",
      "probabilities": {
        "thermal": 0.0,
        "mechanical": 0.0,
        "electrical": 0.0,
        "connectivity": 1.0,
        "sensor_fault": 0.0,
        "process": 0.0,
        "unknown": 0.0
      },
      "confidence": 1.0,
      "answer_confidence": 1.0,
      "action": {
        "act_probability": 1.0
      },
      "abstain": false
    }
  },
  "usage": {
    "input_tokens": 150,
    "output_tokens": 0
  },
  "routing": {
    "model": "multilingual",
    "repo": "/models/hub/models--convaiinnovations--laya/snapshots/aa8c91ca088ec597df95a0d1c76b3063cb2ae5e8/multilingual",
    "reason": "explicit model='multilingual'",
    "detection": null,
    "workflow": null
  },
  "set": {
    "name": "incident-triage",
    "version": 2
  }
}
```

`POST /v1/systemone` with an ad hoc question, in 46 ms. Its answers have no `abstain` field, so the caller applies its own threshold.

```json
{"state": {"text": "Can someone replace the filter on pump 2 next week?"},
 "questions": {"intent": {"type": "choice", "instructions": "What does the operator want?",
               "criteria": {"report_fault": "reports a fault", "status_request": "asks for a status",
                            "maintenance_request": "asks for maintenance"}}}}
```

```json
{
  "model": "laya-rl-agent",
  "answers": {
    "intent": {
      "type": "choice",
      "choice": "maintenance_request",
      "probabilities": {
        "report_fault": 0.0013,
        "status_request": 0.0259,
        "maintenance_request": 0.9728
      },
      "confidence": 0.8815,
      "answer_confidence": 0.9728,
      "action": {
        "act_probability": 1.0
      }
    }
  },
  "usage": {
    "input_tokens": 55,
    "output_tokens": 0
  },
  "routing": {
    "model": "multilingual",
    "repo": "/models/hub/models--convaiinnovations--laya/snapshots/aa8c91ca088ec597df95a0d1c76b3063cb2ae5e8/multilingual",
    "reason": "explicit model='multilingual'",
    "detection": null,
    "workflow": null
  }
}
```

**Errors** [measured, laya 0.3.11 and 0.3.17]:

| Status | Route | `detail` | When |
|--------|-------|----------|------|
| 400 | Sets | `request body must be an object with a 'state' field` | No `state`, or the body isn't JSON |
| 400 | `/v1/systemone` | `request body must be valid JSON`, or `'questions' must be an object` | laya's own checks |
| 401 | Both | `invalid or missing bearer token` | Missing, malformed or wrong key |
| 404 | Sets | `unknown question set` | Unknown set, or a method other than POST |
| 413 | Any | `request body too large` | Over 64 KiB, counted as the body arrives |
| 422 | Both | `system1 accepts at most 8 questions and 4000 state characters per request` | Too many questions or state characters |
| 422 | Both | `state is 1350 tokens; the multilingual checkpoint reads at most 764, send a shorter canonical state` | State over the token budget |
| 422 | `/v1/systemone` | `question 'q': options over 48 tokens would be cut: [...]` | An option that laya would cut |
| 422 | `/v1/systemone` | `question 'q': instructions and options are 405 tokens; the multilingual checkpoint reads at most 256` | Head over its budget |
| 422 | `/v1/systemone` | `question 'q': unknown type 'rank'; use one of ['choice', 'noul', 'score']` | laya's own question check |
| 422 | `/v1/systemone` | `question 'q': no 'instructions'; add the text the model should answer` | laya's own question check |
| 500 | Both | `inference failed` | Any other error; laya hides the details [source] |

Sets are checked against the budgets at start-up, so a question's own 422s can only come from `/v1/systemone`.

---

## Appendix E. Method Notes

- **Host and runs:** as in §4.1. Every figure comes from a script run against a spike container, and repeated runs gave identical answers.
- **Memory.**
  - Peaks are the server's `RssAnon`, read from `/proc/1/status` every 2 ms by a sampler inside the container during the request. The body-limit test sampled `VmRSS` the same way.
  - Resident figures are `RssAnon` at idle, or `docker stats`.
  - Docker Desktop's VM here uses cgroup v1, so there was no `memory.peak` to read.
- **Latency:** the wall-clock time of each HTTP call from the Windows host, through Docker Desktop's port forwarding. A 422 that needs no tokenizer took 2 ms, so transport adds little.
- **Quality sets:** 8 alerts and 14 operator messages (8 requests and 6 greetings or thanks) in English, Spanish and Hindi, labelled by us, plus the wording and gate variants of §4.4. They find failure modes; they don't measure accuracy.
- **State fields:** the 8 alerts and lorem ipsum were sent with the alert alone, with a site, with placeholder and realistic device names, with device types, with the device field renamed, and with "device" removed from the `connectivity` description (§4.4). The last two used `/v1/systemone` with the set's questions, which gave the same answers as the set route.
- **0.3.11 against 0.3.17:** the same wrapper, sets and checkpoint in two images that differ only in the laya pin. 22 set requests (8 alerts on `incident-triage`, 14 messages on `operator-intake`) gave 50 answers, and every choice and probability was compared.
- **Chunked bodies:** raw HTTP/1.1 requests with `Transfer-Encoding: chunked` and no `Content-Length`, with and without a key, while the server's memory was sampled.
- **Regression suite:** fail-closed starts, the status codes in Appendix D, set loading, the hardened container and the Compose service. It passed on the 0.3.11 and 0.3.17 images.
- **Scripts:** the spike's scripts and raw outputs aren't in any repository yet. S1 ports the regression suite and the body-limit test into `stacks/ai/system1/tests/`.

---

*Draft for discussion. Once agreed, the design parts move to `raisga/p4n4-docs` (specs F-0.2.7 and ADR-003).*
