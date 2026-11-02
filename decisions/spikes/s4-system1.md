# Spike S4: System 1 parity

**Date:** 2026-11  
**Status:** Partial: desk research answers Q13 (the licence); the export, the port and the measurements are still to do  
**Plan:** [§9.1](https://github.com/raisga/p4n4/blob/main/RUST_REWRITE_PLAN.md#91-spikes-p0)  
**Code:** none yet; `spikes/s4-system1/` in the superproject once the hands-on part starts

## Questions

From the plan's S4 row:

- Export the pinned checkpoint to ONNX, including the fp16 embedding change.
- Port the prediction path and run the parity corpus against the tolerances in [§6.8](https://github.com/raisga/p4n4/blob/main/RUST_REWRITE_PLAN.md#68-system-1-p4n4-system1).
- Measure latency and memory on x86_64 and the Pi 5.
- Read Laya's licence ([Q13](https://github.com/raisga/p4n4/blob/main/RUST_REWRITE_PLAN.md#13-open-questions)): does it allow redistributing the System 1 ONNX export?

This note answers the last question only.

## Setup

Desk research; nothing ran but downloads. The sources, at the revisions p4n4 pins:

- The model repository [`convaiinnovations/laya`](https://huggingface.co/convaiinnovations/laya) at `aa8c91ca088ec597df95a0d1c76b3063cb2ae5e8`, the `SYSTEM1_REVISION` in `stacks/ai/.env.example`. Its `main` has since moved to `7b928d8` (2026-10-03).
- The `laya` 0.4.6 wheel from PyPI (`laya-0.4.6-py3-none-any.whl`), the version the System 1 image installs (`stacks/ai/system1/requirements.in`).
- The base encoders: [`jhu-clsp/mmBERT-base`](https://huggingface.co/jhu-clsp/mmBERT-base) (at `c5955035`) and [`JHU-CLSP/mmBERT`](https://github.com/JHU-CLSP/mmBERT) on GitHub for `multilingual`, the default checkpoint (`SYSTEM1_CHECKPOINT`); [`answerdotai/ModernBERT-large`](https://huggingface.co/answerdotai/ModernBERT-large) for `english`.

## Results

### The licences

- **Model repository.** The card declares `license: apache-2.0` and the tag `commercial-use`, and the repository is neither gated nor private. The README lists the weights as "Apache 2.0" and credits "Apache 2.0 · Convai Innovations". None of its 38 files is a LICENSE or NOTICE file: they are 3 weight files, configs, tokenizers, 3 Python files, the README, evaluation results and images. `main` still declares `apache-2.0`.
- **`laya` package.** The 0.4.6 wheel's metadata has `License: Apache-2.0`, the OSI Apache classifier and Convai Innovations as author, and the wheel includes the licence text. Its homepage is the model page; `convaiinnovations/laya` on GitHub isn't public.
- **Encoders.** ModernBERT-large's card declares `apache-2.0`. mmBERT-base's card declares `mit`, and its GitHub README shows an MIT badge, but neither repository has a LICENSE file, so there is no copyright line to reproduce.
- **In p4n4.** [decisions/system1.md §2.5](../system1.md#25-licences) records the same chain, and `stacks/ai/system1/train/card.py` writes it into each fine-tuned checkpoint's card. `typed-decisions` stays out, because its base model isn't stated.

### What redistributing an export takes

An ONNX export is a mechanical transformation of the weights, which Apache-2.0 §1 counts as Object form ("conversions to other media types"). With the fp16 embedding change, it is also a Derivative Work. Either way, §4 allows distributing it, modified or not, on these conditions:

1. Give recipients a copy of the Apache-2.0 licence.
2. Mark what changed: the conversion to ONNX and the fp16 embeddings.
3. Keep the source's attribution: the licence and "Convai Innovations".
4. Include the attributions of the work's NOTICE file. There is no NOTICE file at the pinned revision.

The grant is perpetual and irrevocable (§2), so a later change upstream doesn't affect the pinned revision. It gives no trademark rights (§6): a release may name the model as its source, but must not suggest an endorsement.

`multilingual` also derives from mmBERT-base, whose MIT licence asks for its notice to go with copies and substantial portions. As no copyright line is published, the release names mmBERT-base and its authors, and includes the MIT text.

### Upstream has no exporter

Found on the way, for the hands-on part: laya 0.4.6 runs exports but can't make them. `laya.onnx_agent.ONNXAgent` (`laya[onnx]`, on onnxruntime) stops with "Please run export_onnx.py first" when the file is missing. The wheel has no `export_onnx.py`, no `torch.onnx` call and no export command, and the pinned revision has no `.onnx` file. So S4 writes the `export-onnx` command in §6.8 itself. For `laya[onnx]`, the "Export plus ONNX Runtime inference" in decisions/system1.md §2.4 holds only for the runtime.

Leads from the wheel:

- **Exporting.** `common.py`'s `_DynamicMultiheadAttention` exists so that the sequence length stays dynamic under the legacy (TorchScript) ONNX exporter: laya's model code is written for that exporter.
- **Graph contract.** `ONNXAgent` feeds five sequential-layout inputs, so it refuses checkpoints with another `option_layout`. The exported head takes `topk(2)` of the option probabilities, so ONNX Runtime refuses a batch whose questions all have one option, which `ONNXAgent` works around (`onnx_agent.py`, lines 146–150 and 779–783).
- **Checking an export.** `laya-evals` can evaluate an export through `ONNXAgent` instead of the torch router (`evals_cli.py`, line 179), before the Rust port exists.

## Decision

**Q13: yes.** Apache-2.0 lets p4n4's release CI publish ONNX exports of the pinned `english` and `multilingual` checkpoints, on the conditions above. The `multilingual` export also carries mmBERT-base's MIT notice. `p4n4 system1 pull` downloads the published export, and the plan's local export remains for checkpoints without one. That leaves only K3's parity half.

This is a reading of the licences, not legal advice. The maintainer signs it off before the first release that attaches an export.

## Follow-ups

- **Plan sync:** answer Q13 in §13, and update §6.8's "Pulling", the release step in §8, and K3 in §11.
- **Release CI:** next to each export, ship the Apache-2.0 text and a notice naming the source repository and revision, and the changes. For `multilingual`, the notice also names mmBERT-base and includes the MIT text.
- **Mirroring:** as far as the licences go, the same terms allow mirroring the pinned files, open question 9 in [decisions/system1.md §13](../system1.md#13-open-questions).
- **Upstream (optional):** ask JHU-CLSP to add a LICENSE file with a copyright line to mmBERT.
- **Hands-on S4:** the export, the port, the parity corpus and the measurements, as the plan describes.
