# System 1: next steps (handoff, 2026-10-13)

State at the end of the session that built S3 and S4 of System 1 (laya typed decisions).
Design and measured results: `docs/decisions/system1.md` §16 (S3) and §17 (S4).

## Where things stand

- **S1, S2, S3 infrastructure and S4 are implemented, but not committed.** No repo has
  had its S3 or S4 commit yet (see "Commit" below).
- **The S4 checkpoint is `ft2`:** `multilingual` fine-tuned for 2 epochs on a generic
  synthetic corpus of 2,421 rows (sha256 `7f37c396…`). On every set's held-out file it
  beats the base. It is **not published and not served by default**: it waits for the
  user to push it to a Hugging Face repo p4n4 controls.
- **Bundle to publish:** `stacks/ai/system1/train/out/checkpoint/` (647 MB, gitignored):
  `multilingual/`, `README.md` (model card), `p4n4-train.json`, `train_report.json`. Its
  hashes match the manifest. The judge runs are in `system1/train/out/{base,tuned}.json`.
- **Docker leftovers:** volume `s4-out` holds `/out/ft1` and `/out/ft2`. `ft1` is
  superseded; drop the volume with `docker volume rm s4-out` once the bundle is pushed.
- **Shipped sets:** `incident-triage` v3 (threshold 0.9); `operator-intake` v4 (intent
  0.95, raised from 0.92 in S4, asset 0.8). Both pass on the base and on `ft2`.
- **Draft sets** (`stacks/ai/system1/drafts/`, never loaded): `device-class`,
  `log-triage`, `system2-router`, `anomaly-screen`. With `ft2` they pass on the whole
  file but hold out in only 1 of 3 splits (anomaly passes its AUC gate, 0.83).

## Results (generic held-out files, base → ft2)

| Set | Base | ft2 |
|-----|------|-----|
| incident-triage | coverage 0.28, held out 0/3 | coverage 0.94, 3/3 |
| operator-intake intent (0.95) | 0.48 (en 0.52, es 0.44), 1/3 | 0.96 (en 1.00, es 0.92), 2/3 |
| operator-intake asset | 0.87, 0/3 | 1.00, 3/3 |
| device-class (draft) | accuracy 0.76 | coverage 0.98, 1/3 |
| log-triage (draft) | accuracy 0.83 | coverage 0.73 from 0.97, 1/3 |
| system2-router (draft) | accuracy 0.56 | coverage 0.95 from 0.85, 1/3 |
| anomaly-screen (draft) | AUC 0.67 | AUC 0.83 |

The same person wrote the templates and the eval files, so these overstate real-site results.

## 1. Commit (the user runs these; never commit for them)

S3 and S4 touch the same files, so one commit per repo covers both. Push lib first.

```
git -C lib add -A && git -C lib commit -m "feat: System 1 ranking gate and site calibration"
git -C stacks/iot add -A && git -C stacks/iot commit -m "feat: System 1 drift alerts in Grafana"
git -C stacks/ai add -A && git -C stacks/ai commit -m "feat: System 1 calibration, drift alerts, fine-tuning"
git -C cli add -A && git -C cli commit -m "feat: System 1 calibration, labels, fine-tuned status"
git -C docs add -A && git -C docs commit -m "docs: System 1 S3 and S4 implementation"
git add stacks/ai stacks/iot && git commit -m "chore: bump submodules for System 1 S3 and S4"
```

- docs already had staged files from earlier work (e.g. `decisions/adr/ADR-004.md`).
- In the root repo, `cli/`, `lib/` and `docs/` show as untracked folders, not
  submodules, so the root bump covers only `stacks/ai` and `stacks/iot`.

## 2. Publish ft2 (system1.md §17.5)

The user does steps 1–2; never push to Hugging Face or any external service.

1. Create a Hugging Face repo p4n4 controls (license apache-2.0).
2. Push `stacks/ai/system1/train/out/checkpoint/` to it. Note the commit's 40-character sha.
3. From `stacks/ai`: `python3 system1/train/pin.py <owner>/<name> <sha> system1/train/out/checkpoint`.
   That adds the hashes to `system1/pins.json` from the local files.
4. Rebuild the image (`make system1-build`): the pins are part of it.
5. Set `SYSTEM1_REPO=<owner>/<name>` and `SYSTEM1_REVISION=<sha>`, run `make system1-pull`
   (or `p4n4 ai system1 pull`), restart `system1`, then `p4n4 ai system1 eval` on every set.
6. Bump every set's version (rule 10: a checkpoint change bumps versions):
   `incident-triage` v4, `operator-intake` v5. Re-run `eval`.
7. Optional, to make it p4n4's default: change the `SYSTEM1_REPO`/`SYSTEM1_REVISION`
   defaults in `stacks/ai/.env.example` and both services in `docker-compose.yml`
   together, then update §17 and the ai-stack page.
8. Run `make system1-test` (the regression suite) against the rebuilt image.

## 3. Still open

- **S4 exit:** a fine-tuned checkpoint beats the base on a held-out *site* file, with the
  same pins. It needs a real site's evaluation examples, and the pins from step 2.
- **Drafts:** they need held-out evidence: bigger eval files (well over 50, best a
  site's own), not more training. To ship one: pass held out on the checkpoint the
  service runs, set thresholds, bump its version, move it to `config/system1/questions/`.
- **S1/S2 exits** (system1.md §14.7, §15.11): Pi 5 p95 latency ≤ 1.5 s within 2 GiB,
  native arm64 CI, gates on ≥ 50 real-site examples per set and language.
- **Image release:** first `system1-v0.1.0` tag; until then `SYSTEM1_IMAGE=p4n4-system1:dev`.
- **Not done on purpose:** `english` isn't fine-tuned; per-language temperatures.

## Tooling

- **Training:** `make system1-train` from `stacks/ai` (about 2 h on 8 CPU cores, 11 GiB
  limit, peak about 7 GiB). It writes `system1/train/out/corpus.jsonl` and `out/checkpoint/`.
- **Judge:** `PYTHONPATH=<repo>/lib python3 system1/train/compare.py OUT.json --url http://127.0.0.1:<port> --key-file KEY SETS_DIR`.
  Run it against a service that has every set, drafts included, mounted at `/config/questions`.
  Compare two runs with `compare.py --show A.json B.json`.
- **Card:** `python3 system1/train/card.py out/checkpoint out/base.json out/tuned.json`.
- **Tests:**
  - training tools: from `stacks/ai`, run `<repo>/cli/.venv/bin/python -m pytest -q system1/train`;
  - cli: from `cli`, run `.venv/bin/python -m pytest -q`;
  - lib: from `lib`, run `<repo>/cli/.venv/bin/python -m pytest -q`.

  The host python has no pytest.
- **Serving a local checkpoint offline** for judging: fake the Hugging Face cache under
  `models--<org>--<name>/` with all three pieces, or `huggingface_hub` tries the network
  and the wrapper refuses to start:
  - `snapshots/<rev>/multilingual/`;
  - `refs/<rev>`;
  - `trees/<rev>.json` (format 1: `{path: {size, blob_id}}`).

  Also give the wrapper a pins file for that repo and revision through `SYSTEM1_PINS`.

## Constraints to keep

- **Git and publishing:**
  - Never `git add/rm/mv/commit/submodule add`. Edit files and give commands; commit
    messages are one line under 60 characters with a conventional prefix.
  - Never push to Hugging Face or any external service.
- **Thresholds:**
  - Never lower thresholds or the accuracy and confident-wrong gates to make a set pass.
    Only `min_coverage` may be lowered; raising a threshold is allowed.
  - Any change to wording, options, thresholds, checkpoint, revision, laya or calibration
    bumps the set's version (rule 10).
- **Host and data:**
  - The host runs other `p4n4-*` containers that must not be touched. Use isolated names
    for test containers and volumes.
  - Don't put personal names in example data, and keep client projects out of the main repos.
