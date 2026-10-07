# Release checklist — p4n4-lib extraction, multi-layer layout, p4n4-api bootstrap

> **Status: historical.** Tasks 1, 3, 5, 6, 7 and 8 are done. Tasks 2 and 4 (publish p4n4-lib, release the
> CLI) were superseded by the v0.2.0 release, which publishes p4n4-lib 0.2.0 (there is no 0.1.0) with
> CLI 0.2.0. The optional tasks 9 (CI lib source) and 10 (p4n4-emu depending on p4n4-lib) are still open.

Generated 2026-07-06. Order matters: **task 1 first** (cli/api CI install p4n4-lib from
its GitHub URL), **task 8 last**. Tasks 6 and 7 can happen anytime. Tasks 4, 9, 10
need the lib on PyPI (task 2).

---

## 1. Commit and push `lib` (p4n4-lib) — FIRST

⚠ The submodule is on a **detached HEAD** (at `133782e`, same commit as `main`).

```bash
cd ~/Desktop/p4n4/lib
git checkout main
git add -A
git commit -m "feat: implement shared library extracted from p4n4-cli

- manifest, env, compose, secrets modules moved from the CLI
- layers: registry of repo URLs, copy paths, required files/env keys
- layout: flat single-layer vs per-layer multi-layer project resolution
- scaffold: fetch stack sources (clone or local path) and copy into projects
- validate: pure project checks returning (passed, errors)
- tests, ruff config, CI and PyPI publish workflows"
git push origin main
gh run watch --repo raisga/p4n4-lib   # confirm CI is green
```

## 2. Publish p4n4-lib v0.1.0 to PyPI — after task 1

One-time trusted-publishing setup:

1. pypi.org → account → *Publishing* → *Add a new pending publisher*:
   project `p4n4-lib`, owner `raisga`, repo `p4n4-lib`,
   workflow `publish.yml`, environment `pypi`.
2. GitHub `raisga/p4n4-lib` → Settings → Environments → create environment `pypi`.

Then release:

```bash
cd ~/Desktop/p4n4/lib
gh release create v0.1.0 --title "v0.1.0" \
  --notes "Initial release: shared library between P4N4 stacks and clients."
# verify in a scratch venv:
pip install p4n4-lib==0.1.0
```

Manual fallback: `uv build && uv publish` with a PyPI API token.

## 3. Commit and push `cli` (p4n4-cli) — after task 1

```bash
cd ~/Desktop/p4n4/cli
git add -A
git commit -m "feat: multi-layer project layout; extract shared code into p4n4-lib

- multi-layer init scaffolds each layer into its own subdirectory
  (iot/, ai/) as separate Compose projects; single-layer stays flat
- up/down honor the stack argument and dependency order; logs gains
  --stack; status prints one table per stack; secret rotate keeps
  shared keys in sync across layer .env files
- p4n4/utils, sources.py, and scaffold logic replaced by p4n4-lib
- tests locate stack sources in CI (sibling clones) or the monorepo
  (stacks/<name>); CI installs p4n4-lib from GitHub"
git push origin main
```

## 4. Release p4n4 (cli) 0.2.0 to PyPI — after tasks 2 and 3

The `p4n4-lib>=0.1.0` dependency must resolve on PyPI first.

1. Bump version to `0.2.0` in **both** `pyproject.toml` and `p4n4/__init__.py`.
2. `CHANGELOG.md`: rename `[Unreleased]` → `[0.2.0] - <date>`.
3. `git commit -m "chore: release 0.2.0"` and push.
4. `gh release create v0.2.0` — the existing publish workflow uploads to PyPI.

## 5. Commit and push `api` (p4n4-api) — after task 1

```bash
cd ~/Desktop/p4n4/api
git add -A
git commit -m "feat: bootstrap read-only FastAPI service on p4n4-lib

- GET /health, /api/v1/project, /api/v1/project/validate,
  /api/v1/stacks, /api/v1/stacks/{stack}
- understands flat and multi-layer project layouts via p4n4_lib.layout
- config via P4N4_PROJECT_DIR / P4N4_API_HOST / P4N4_API_PORT
- state-changing endpoints deferred until auth lands
- tests (synthetic projects, stubbed Compose), CI, README status section"
git push origin main
```

## 6. Commit and push `tools/emu` (p4n4-emu) — anytime

⚠ `README.md` and `guide.md` mix my own earlier uncommitted doc edits with the new
multi-layer additions — commit mine separately first if I want them split.

```bash
cd ~/Desktop/p4n4/tools/emu
git add -A
git commit -m "feat: resolve stacks from p4n4 projects (flat and multi-layer)

- new utils/project.py: finds .p4n4.json, reads enabled layers, and
  resolves each stack dir (flat root or <project>/<stack>/), mirroring
  p4n4_lib.layout; dedupes the three _resolve_stack_dir copies
- --stack defaults to the surrounding project's enabled stacks;
  accepts comma-separated names; 'all' means enabled stacks in a project
- down stops stacks in reverse dependency order
- deliberately no p4n4-lib dependency until it is published"
git push origin main
```

## 7. Commit and push `web/docs` (p4n4-docs) — anytime

```bash
cd ~/Desktop/p4n4/web/docs
git add -A
git commit -m "docs: document multi-layer project layout and p4n4-lib (ADR-002)

- ADR-002: per-layer subdirectories in multi-layer projects
- getting-started: --layer usage and both project layouts
- cli-reference: init flags, logs --stack, per-layer validate/secret
- architecture: real p4n4-cli tree + p4n4_lib modules; per-layer .env
  strategy; removed stale Jinja2/single-.env/p4n4 up --all claims"
git push origin main
```

## 8. Update submodule pointers in root p4n4 repo — LAST (after 1, 3, 5, 6, 7)

```bash
cd ~/Desktop/p4n4
git add lib cli api tools/emu web/docs
git commit -m "chore: update submodules for p4n4-lib extraction and multi-layer layout"
git push
```

Deliberately excludes `web/blog` (dirty state predates this work) and this `TODO.md`.

---

## Optional follow-ups (after task 2)

### 9. Decide the p4n4-lib source for cli/api CI

Keep installing from `git+https://github.com/raisga/p4n4-lib.git` (tests against lib
`main`, catches integration breakage early) **or** drop that install step and resolve
from PyPI (tests what users actually get). Update `.github/workflows/ci.yml` in
p4n4-cli and p4n4-api accordingly.

### 10. Give p4n4-emu a real p4n4-lib dependency

Add `p4n4-lib>=0.1.0` to `tools/emu/pyproject.toml`, run `uv lock` / `uv sync`, and
replace the mirrored logic in `p4n4_emu/utils/project.py` (find_manifest / layout
rules) with `p4n4_lib.manifest` + `p4n4_lib.layout`. Keep `expand_stacks` and the
emu-specific fallbacks (explicit `--stack-dir`, bare checkout candidates).
`tests/test_project.py` should keep passing unchanged.
