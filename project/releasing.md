# Releasing

Each repository keeps its GitHub release bodies in its own `release-notes/` folder, one file
per release named `v<version>.md`, passed to `gh release create --notes-file`. This page holds
what spans repositories: the order of a release set and how to write a body.

## v0.4.0 release set (2026-10)

Publish in this order. Each step needs the one before it:

| # | Release | Repo | Ships as | Notes |
|---|---|---|---|---|
| 1 | p4n4-lib **0.4.0** | `lib` | PyPI `p4n4-lib` | Not written yet (`lib/release-notes/v0.4.0.md`) |
| 2 | p4n4-api **0.3.0** | `api` | `ghcr.io/raisga/p4n4-api` (installs p4n4-lib from PyPI) | [api/release-notes/v0.3.0.md](https://github.com/raisga/p4n4-api/blob/main/release-notes/v0.3.0.md) |
| 2 | p4n4-dashboard **1.3.0** | `dashboard` | `ghcr.io/raisga/p4n4-dashboard` | [dashboard/release-notes/v1.3.0.md](https://github.com/raisga/p4n4-dashboard/blob/main/release-notes/v1.3.0.md) |
| 3 | p4n4 (CLI) **0.4.0** | `cli` | PyPI `p4n4` | Not written yet (`cli/release-notes/v0.4.0.md`) |

Release the two images together, after p4n4-lib 0.4.0 is on PyPI and `api/uv.lock` locks it.
The CLI goes last: its `[Unreleased]` changes need p4n4-lib 0.4.0.

### Before publishing

- **p4n4-lib 0.4.0:** move its `[Unreleased]` CHANGELOG entry under `0.4.0` and write
  `release-notes/v0.4.0.md` (the edge layer merged into ai, the extras layer, `migrate_edge`,
  `layout.runner_layer`).
- **p4n4 (CLI) 0.4.0:** bump the version in `pyproject.toml` **and** `p4n4/__init__.py`
  (both still say 0.3.0), move its `[Unreleased]` entry, and write `release-notes/v0.4.0.md`.
  Breaking: `--layer edge` and `--source-edge` are gone. New: `p4n4 migrate edge`,
  `--layer extras`, `--ei-runner` and the project `.gitignore`.
- **p4n4-emu:** `uv lock --upgrade-package p4n4-lib` once 0.4.0 is on PyPI, so its lock
  matches its `p4n4-lib>=0.4.0` requirement. Its CI fails until then.
- **Smoke test** on a host with no other p4n4 project running:

  ```bash
  p4n4 init demo --layer all && cd demo && p4n4 up
  curl -s localhost:11434/api/version                     # Ollama
  curl -s localhost:8080/health                           # runner: "influxdb_ok": true
  mosquitto_sub -h localhost -t 'inference/+/result' -C 1 -W 10 &
  mosquitto_pub -h localhost -t sensors/smoke-01/raw -m '{"values":[1.2,3.4,5.6]}'
  ```

  Check that the result reaches the `ai_events` bucket, then run `p4n4 migrate edge` on a
  real project that still has an edge layer. This full `p4n4 up` of iot + ai + runner wasn't
  run when the merge was made.

From the monorepo root:

```bash
gh release create v0.3.0 --repo raisga/p4n4-api       --title v0.3.0 --notes-file api/release-notes/v0.3.0.md
gh release create v1.3.0 --repo raisga/p4n4-dashboard --title v1.3.0 --notes-file dashboard/release-notes/v1.3.0.md
gh release create v0.4.0 --repo raisga/p4n4-cli       --title v0.4.0 --notes-file cli/release-notes/v0.4.0.md
```

### How they fit together

| | needs | used by |
|---|---|---|
| p4n4-lib 0.4.0 | — | api 0.3.0, CLI 0.4.0, p4n4-emu (`p4n4-lib>=0.4.0`) |
| p4n4-api 0.3.0 | p4n4-lib 0.4.0 | dashboard 1.3.0 (view settings, banner, account tools, chats, vision) |
| p4n4-dashboard 1.3.0 | p4n4-api 0.3.0 | — |
| p4n4 (CLI) 0.4.0 | p4n4-lib 0.4.0 | — |

## v0.3.0 release set (2026-10)

Publish in this order. Each step needs the one before it:

| # | Release | Repo | Ships as | Notes |
|---|---|---|---|---|
| 1 | p4n4-system1 **0.1.0** | `stacks/ai` (tag `system1-v0.1.0`) | `ghcr.io/raisga/p4n4-system1` | The image `SYSTEM1_IMAGE` pins; no GitHub release |
| 1 | p4n4-lib **0.3.0** | `lib` | PyPI `p4n4-lib` | [lib/release-notes/v0.3.0.md](https://github.com/raisga/p4n4-lib/blob/main/release-notes/v0.3.0.md) |
| 2 | p4n4-api **0.2.0** | `api` | `ghcr.io/raisga/p4n4-api` (installs p4n4-lib from PyPI) | [api/release-notes/v0.2.0.md](https://github.com/raisga/p4n4-api/blob/main/release-notes/v0.2.0.md) |
| 2 | p4n4-dashboard **1.2.0** | `dashboard` | `ghcr.io/raisga/p4n4-dashboard` | [dashboard/release-notes/v1.2.0.md](https://github.com/raisga/p4n4-dashboard/blob/main/release-notes/v1.2.0.md) |
| 3 | p4n4 (CLI) **0.3.0** | `cli` | PyPI `p4n4` | [cli/release-notes/v0.3.0.md](https://github.com/raisga/p4n4-cli/blob/main/release-notes/v0.3.0.md) |

p4n4-lib 0.2.1 and p4n4-dashboard 1.1.1 were never published: their changes ship in
p4n4-lib 0.3.0 and p4n4-dashboard 1.2.0. Release the two images together. The stacks,
templates, emu, hw and docs keep their own version lines; the root repo records their
commits.

From the monorepo root:

```bash
gh release create v0.3.0 --repo raisga/p4n4-lib       --title v0.3.0 --notes-file lib/release-notes/v0.3.0.md
gh release create v0.2.0 --repo raisga/p4n4-api       --title v0.2.0 --notes-file api/release-notes/v0.2.0.md
gh release create v1.2.0 --repo raisga/p4n4-dashboard --title v1.2.0 --notes-file dashboard/release-notes/v1.2.0.md
gh release create v0.3.0 --repo raisga/p4n4-cli       --title v0.3.0 --notes-file cli/release-notes/v0.3.0.md
```

### How they fit together

| | needs | used by |
|---|---|---|
| p4n4-lib 0.3.0 | — | CLI 0.3.0, api 0.2.0 (`p4n4-lib>=0.3.0`) |
| p4n4-api 0.2.0 | p4n4-lib 0.3.0 | dashboard 1.2.0 (assistant, views, Feed, Home) |
| p4n4-dashboard 1.2.0 | p4n4-api 0.2.0 | CLI 0.3.0 (`--layer dashboard` pins it) |
| p4n4 (CLI) 0.3.0 | p4n4-lib 0.3.0, p4n4-system1 0.1.0 | — |

## v0.2.0 release set (2026-10)

Publish in this order. Each step needs the one before it:

| # | Release | Repo | Ships as | Notes |
|---|---|---|---|---|
| 1 | p4n4-lib **0.2.0** | `lib` | PyPI `p4n4-lib` | [lib/release-notes/v0.2.0.md](https://github.com/raisga/p4n4-lib/blob/main/release-notes/v0.2.0.md) |
| 2 | p4n4-api **0.1.0** | `api` | `ghcr.io/raisga/p4n4-api` (installs p4n4-lib from PyPI) | [api/release-notes/v0.1.0.md](https://github.com/raisga/p4n4-api/blob/main/release-notes/v0.1.0.md) |
| 2 | p4n4-dashboard **1.1.0** | `dashboard` | `ghcr.io/raisga/p4n4-dashboard` | [dashboard/release-notes/v1.1.0.md](https://github.com/raisga/p4n4-dashboard/blob/main/release-notes/v1.1.0.md) |
| 3 | p4n4 (CLI) **0.2.0** | `cli` | PyPI `p4n4` | [cli/release-notes/v0.2.0.md](https://github.com/raisga/p4n4-cli/blob/main/release-notes/v0.2.0.md) |

Release the two images together. The other repos (stacks, templates, emu, hw, docs) keep
their own version lines and get no release in this set.

From the monorepo root:

```bash
gh release create v0.2.0 --repo raisga/p4n4-lib       --title v0.2.0 --notes-file lib/release-notes/v0.2.0.md
gh release create v0.1.0 --repo raisga/p4n4-api       --title v0.1.0 --notes-file api/release-notes/v0.1.0.md
gh release create v1.1.0 --repo raisga/p4n4-dashboard --title v1.1.0 --notes-file dashboard/release-notes/v1.1.0.md
gh release create v0.2.0 --repo raisga/p4n4-cli       --title v0.2.0 --notes-file cli/release-notes/v0.2.0.md
```

### How they fit together

| | needs | used by |
|---|---|---|
| p4n4-lib 0.2.0 | — | CLI 0.2.0, api 0.1.0 (`p4n4-lib>=0.2.0`) |
| p4n4-api 0.1.0 | p4n4-lib 0.2.0 | dashboard 1.1.0 (sign-in, `normie` role) |
| p4n4-dashboard 1.1.0 | p4n4-api 0.1.0 | CLI 0.2.0 (`--layer dashboard` pins it) |
| p4n4 (CLI) 0.2.0 | p4n4-lib 0.2.0 | — |

## Writing a release body

Every file follows the same structure, so the set reads as one release:

1. **One sentence**: what the release is (first release) or what it changes. Don't start
   with a heading: `gh` shows `--title` above the body.
2. **Install**: one code block (`pip`/`pipx` or `docker pull`, with Python or platforms).
3. **Trust callout** (`> **Trusted networks only.** …`) for anything that runs services.
4. **`## What's in it`** for a first release, **`## Highlights`** for later ones.
5. **`## Compatibility`**: the versions it needs and pairs with, from the table above.
6. **`## Upgrading from x.y`** and/or **`## Limitations`**, when there are any.
7. **`Full list: [CHANGELOG.md](…)`** for repos that keep a changelog (lib, CLI).

Put new files in the repo's `release-notes/` as `v<version>.md`, and add a row to the set
table above.

## Where CI gets p4n4-lib

The CI of p4n4-cli and p4n4-api installs p4n4-lib from git `main`
(`pip install "p4n4-lib @ git+https://github.com/raisga/p4n4-lib.git"`), so a change to the
library breaks them on the next run, before it ships. The API image (`P4N4_LIB` build argument, `p4n4-lib>=0.4.0,<0.5`)
and p4n4-emu install it from PyPI, which is what users get. This is why p4n4-lib goes first in every set: until
it's published, the image build and p4n4-emu's CI fail on a version requirement the CLI's
CI already meets.

Still open: whether cli and api CI should switch to PyPI, testing exactly what users install
at the cost of finding library breakage only after a release.

