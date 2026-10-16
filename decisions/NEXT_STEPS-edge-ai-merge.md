# Edge → AI merge: next steps (handoff, 2026-10-14)

State at the end of the session that merged the edge stack into the AI stack and turned
p4n4-edge into p4n4-extras. Decision: [ADR-005](adr/ADR-005.md). User-facing guide:
[Migrating from the edge stack](../guides/migrating-from-edge.md).

## Where things stand

- **Everything is edited on disk, nothing is staged or committed.** Twelve repositories
  changed (see "Commit" below).
- **The edge runner is the AI stack's `ei-runner` service and profile**, off by default (on
  with `p4n4 init --layer all` or `--ei-runner`). Container `p4n4-ei-runner`, port 8080 on
  `127.0.0.1`, topics and variables unchanged.
- **p4n4-extras is the repository that was p4n4-edge**: an opt-in `extras` layer, never in
  `--layer all`, nothing enabled by default. Each extra is its own directory, pulled into
  the layer's compose file with `include:`; `_template/` is the skeleton to copy. It ships
  no extras yet.
- **`edge` is a legacy layer**: old projects keep running, with a reminder. `p4n4 migrate
  edge` moves it into `ai` (or into `extras` when it isn't the runner, as in the go2rtc
  template), keeping what it replaces in `edge.pre-migrate/`.
- **p4n4-lib is bumped to 0.4.0**; the CLI, p4n4-api (and its Dockerfile) and p4n4-emu
  require `>=0.4.0`. The CLI, API and dashboard versions are **not** bumped.
- **The submodule is still at `stacks/edge`** on disk: the move to `stacks/extras` needs
  `git mv` (below).
- **`p4n4 init` now writes a project `.gitignore`** (every layer's `.env`, generated
  credentials, model files, p4n4-api's data, `edge.pre-migrate/`); it never overwrites one.
  Projects scaffolded before have none: copy it from a fresh `p4n4 init`.
- **Also fixed:** the AI stack's Ollama pin (`ollama/ollama:0.35` doesn't exist; now
  `0.35.1`), two errors in the p4n4-ai README, and the design docs (`design.md` 0.2,
  `specs.md` 0.1-draft8 with a status pass, `architecture.md`, ADR-001 to ADR-003).

## Verified

| Check | Result |
|---|---|
| `docker compose config`: iot, ai (every profile and combination), extras, `_template/`, the go2rtc template's layers | Pass, no warnings; yamllint clean |
| Runner built from `stacks/ai`, run in an isolated Compose project | `/health`, `/api/v1/info`, `/api/v1/infer`, MQTT `sensors/<id>/raw` → `inference/<id>/result` |
| `p4n4 migrate edge` on a project made from the committed (pre-merge) stacks | Runner spliced into `ai/`, models and keys moved, `p4n4 validate` passes |
| `p4n4 init --layer iot,extras`, then `p4n4 up extras` | Skipped, no containers, validates |
| go2rtc template `tests/smoke.sh` | PASS, extras layer included |
| Tests | lib 196, cli 117, api 408, emu 617 (+1 skipped), runner 93, dashboard 246 + `flutter analyze`, templates `validate.py`; ruff clean |

**Not verified:** a full `p4n4 up` of iot + ai + runner (a project was running on the host
with the fixed names and ports), so the runner's InfluxDB write and Grafana → InfluxDB at
runtime; GitHub CI; GPU, Raspberry Pi and Jetson hardware.

## 1. Review

`docs`, the root `README.md` and `tools/hw/README.md` already had staged work: the session's
edits sit unstaged on top. Review them with `git diff` (not `--cached`). `web/blog` has
changes that aren't from this session.

## 2. Rename the repository and move the submodule

```bash
gh repo rename p4n4-extras --repo raisga/p4n4-edge --yes
cd ~/Desktop/p4n4
git mv stacks/edge stacks/extras
git config -f .gitmodules submodule.stacks/edge.url git@github.com:raisga/p4n4-extras.git
git submodule sync stacks/extras
git -C stacks/extras remote -v      # should show p4n4-extras
```

The submodule's name stays `stacks/edge`, as other submodules keep their old names.

## 3. Commit

One commit per repository, then the root, which records the new submodule commits:

| Repository | Commit message |
|---|---|
| `stacks/ai` | `feat: add the ei-runner profile, pin ollama 0.35.1` |
| `stacks/extras` (was `stacks/edge`) | `feat!: turn p4n4-edge into p4n4-extras` |
| `stacks/iot` | `docs: point inference results at p4n4-ai's runner` |
| `lib` | `feat!: extras layer, legacy edge layer, migrate_edge` |
| `cli` | `feat!: add p4n4 migrate edge and the extras layer` |

The project `.gitignore` (`lib/p4n4_lib/scaffold.py`, `cli/p4n4/commands/init.py`) goes
in the `lib` and `cli` commits above.
| `api` | `feat: find the runner in the ai layer's ei-runner` |
| `dashboard` | `feat: list the EI runner under the AI stack` |
| `tools/emu` | `feat: move ei-runner limits to ai, add extras stack` |
| `tools/templates` | `feat!: rename the go2rtc template's edge layer to extras` |
| `tools/hw` | `docs: relabel the edge phase as the AI stack's runner` |
| `docs` | `docs: ADR-005, design docs and specs status pass` |
| root (`p4n4`) | `chore: merge edge into ai, move stacks/edge to extras` |

## 4. Push and publish, in this order

1. Rename the repository (step 2), so the CLI's CI can clone `p4n4-extras`.
2. Push `stacks/ai`, `stacks/extras` and `stacks/iot`: `p4n4 init` clones their `main`, and
   the new p4n4-lib expects the runner files in p4n4-ai.
3. Push `lib`, then publish **p4n4-lib 0.4.0** to PyPI. The CLI's CI installs lib from git
   `main`; p4n4-emu's CI and the API image install it from PyPI.
4. Push `cli`, `api`, `dashboard`, `tools/templates`, `tools/hw` and `docs`.
5. In `tools/emu`: `uv lock --upgrade-package p4n4-lib` (its lock still pins 0.2.0), then
   push. Its CI fails until step 3 is done.
6. Push the root.

## 5. Release

- **p4n4-lib 0.4.0:** move its `[Unreleased]` CHANGELOG entry under `0.4.0`, write
  `release-notes/v0.4.0.md`.
- **p4n4 (CLI):** bump to 0.4.0 in `pyproject.toml` **and** `p4n4/__init__.py`; move its
  `[Unreleased]` entry; write `release-notes/v0.4.0.md` (breaking: `--layer edge` and
  `--source-edge` removed; new: `migrate edge`, `--layer extras`, `--ei-runner`, the
  project `.gitignore`).
- **p4n4-api, p4n4-dashboard:** bump and write release notes (the API now needs p4n4-lib
  0.4.0; the dashboard lists the EI runner under the AI stack).
- **`docs/project/releasing.md`:** add the v0.4.0 release set, with the order above.

## 6. Verify on a host with nothing else running

```bash
p4n4 init demo --layer all && cd demo && p4n4 up
curl -s localhost:11434/api/version                       # Ollama
curl -s localhost:8080/health                             # runner (ei-runner is on with --layer all)
mosquitto_sub -h localhost -t 'inference/+/result' -C 1 -W 10 &
mosquitto_pub -h localhost -t sensors/smoke-01/raw -m '{"values":[1.2,3.4,5.6]}'
curl -su admin:<GRAFANA_PASSWORD> localhost:3000/api/datasources   # then /api/datasources/uid/<uid>/health
```

Check that `/health` reports `"influxdb_ok": true` and that results reach the `ai_events`
bucket. Then try `p4n4 migrate edge` on a real project that still has an edge layer.

## 7. Follow-ups

- **`p4n4 status`** has no image column (F-0.1.4); stack names have no shell completion
  (F-0.3.7).
- **The CLI's `jinja2` dependency is unused**: drop it.
- **Extras catalog** ([AI harness design](ai-harness.md#5-extras-community-stacks) §5): a
  manifest per extra, a validator, `p4n4 extra list|add|remove`.
- **p4n4-dashboard's Services tab** has a fixed catalog: it can't show extras yet.
- **The go2rtc template's extras layer** is one compose file; it could follow the
  per-extra directory layout of p4n4-extras.
- **Old runner volumes** in migrated projects (`<project>-edge_ei-runner-data`) hold
  nothing and can be removed.
