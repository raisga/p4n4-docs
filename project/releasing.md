# Releasing

Each repository keeps its GitHub release bodies in its own `release-notes/` folder, one file
per release named `v<version>.md`, passed to `gh release create --notes-file`. This page holds
what spans repositories: the order of a release set and how to write a body.

## v0.4.1 release set (pending)

A fix release: `p4n4 up`, `p4n4 ai system1 enable` and the n8n credential sync create
`p4n4-net` on the project's `P4N4_SUBNET`. Until now they always used `172.20.0.0/16` and
failed wherever another network held that range
([Troubleshooting](../guides/troubleshooting.md)). The fix is in both repos' `[Unreleased]`
changelog entries; neither release is cut yet.

| # | Release | Repo | Ships as | Notes |
|---|---|---|---|---|
| 1 | p4n4-lib **0.4.1** | `lib` | PyPI `p4n4-lib` | `layout.network_subnet()`; to write: `lib/release-notes/v0.4.1.md` |
| 2 | p4n4 (CLI) **0.4.1** | `cli` | PyPI `p4n4` | Requires p4n4-lib 0.4.1; to write: `cli/release-notes/v0.4.1.md` |

p4n4-api and p4n4-emu don't call `ensure_network`, so they stay on p4n4-lib 0.4.0 and get no
release.

### Before publishing

- **Versions:** bump `pyproject.toml` **and** the package's `__init__.py` in each repo, date
  the `[Unreleased]` changelog entry, and in the CLI require `p4n4-lib>=0.4.1`.
- **Lock:** once p4n4-lib 0.4.1 is on PyPI, `uv lock --upgrade-package p4n4-lib` in `cli`.
- **Smoke test** on a host with no other p4n4 project running. It's also the first full
  `p4n4 up` of iot + ai + runner since the edge layer merged into ai, which wasn't run for
  0.4.0:

  ```bash
  p4n4 init demo --layer all && cd demo && p4n4 up
  curl -s localhost:11434/api/version                     # Ollama
  curl -s localhost:8080/health                           # runner: "influxdb_ok": true
  mosquitto_sub -h localhost -t 'inference/+/result' -C 1 -W 10 &
  mosquitto_pub -h localhost -t sensors/smoke-01/raw -m '{"values":[1.2,3.4,5.6]}'
  ```

  Check that the result reaches the `ai_events` bucket. Where another network holds
  `172.20.0.0/16`, the first `up` should stop and name `P4N4_SUBNET`; set it in `iot/.env`
  and check that `docker network inspect p4n4-net` shows that range. Then run
  `p4n4 migrate edge` on a real project that still has an edge layer.

From the monorepo root. Publishing p4n4-lib's release runs its PyPI workflow; wait for it
before the lock above:

```bash
gh release create v0.4.1 --repo raisga/p4n4-lib --title v0.4.1 --notes-file lib/release-notes/v0.4.1.md
gh release create v0.4.1 --repo raisga/p4n4-cli --title v0.4.1 --notes-file cli/release-notes/v0.4.1.md
```

## Past release sets

Each was published in the table's order: p4n4-lib first, then the two images together, then
the CLI. The stacks, templates, emu, hw and docs keep their own version lines and got no
release; the root repo records their commits.

| Set | p4n4-lib | p4n4-api | p4n4-dashboard | p4n4 (CLI) | Theme |
|---|---|---|---|---|---|
| v0.4.0 (2026-10-16) | [0.4.0](https://github.com/raisga/p4n4-lib/blob/main/release-notes/v0.4.0.md) | [0.3.0](https://github.com/raisga/p4n4-api/blob/main/release-notes/v0.3.0.md) | [1.3.0](https://github.com/raisga/p4n4-dashboard/blob/main/release-notes/v1.3.0.md) | [0.4.0](https://github.com/raisga/p4n4-cli/blob/main/release-notes/v0.4.0.md) | Edge merged into ai, extras layer, `p4n4 migrate edge`; kiosk mode, view settings, vision overlay |
| v0.3.0 (2026-10-14) | [0.3.0](https://github.com/raisga/p4n4-lib/blob/main/release-notes/v0.3.0.md) | [0.2.0](https://github.com/raisga/p4n4-api/blob/main/release-notes/v0.2.0.md) | [1.2.0](https://github.com/raisga/p4n4-dashboard/blob/main/release-notes/v1.2.0.md) | [0.3.0](https://github.com/raisga/p4n4-cli/blob/main/release-notes/v0.3.0.md) | System 1 (with p4n4-system1 0.1.0, tag `system1-v0.1.0` in `stacks/ai`); assistant, views, Feed, Home |
| v0.2.0 (2026-10-03) | [0.2.0](https://github.com/raisga/p4n4-lib/blob/main/release-notes/v0.2.0.md) | [0.1.0](https://github.com/raisga/p4n4-api/blob/main/release-notes/v0.1.0.md) | [1.1.0](https://github.com/raisga/p4n4-dashboard/blob/main/release-notes/v1.1.0.md) | [0.2.0](https://github.com/raisga/p4n4-cli/blob/main/release-notes/v0.2.0.md) | Multi-layer projects, p4n4-lib on PyPI, sign-in |

p4n4 (CLI) 0.2.1 (2026-10-12) was a release of the CLI alone. p4n4-lib 0.2.1 and
p4n4-dashboard 1.1.1 were never published: their changes shipped in p4n4-lib 0.3.0 and
p4n4-dashboard 1.2.0.

### How the current versions fit together

| | needs | used by |
|---|---|---|
| p4n4-lib 0.4.0 | — | api 0.3.0, CLI 0.4.0, p4n4-emu (`p4n4-lib>=0.4.0`) |
| p4n4-api 0.3.0 | p4n4-lib 0.4.0 | dashboard 1.3.0 (view settings, banner, account tools, chats, vision) |
| p4n4-dashboard 1.3.0 | p4n4-api 0.3.0 | the dashboard layer `p4n4 init` scaffolds (`DASHBOARD_VERSION=1.3.0`) |
| p4n4 (CLI) 0.4.0 | p4n4-lib 0.4.0, p4n4-system1 0.1.0 | — |

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

Put new files in the repo's `release-notes/` as `v<version>.md`, and add the set to this
page.

## Where CI gets p4n4-lib

The CI of p4n4-cli and p4n4-api installs p4n4-lib from git `main`
(`pip install "p4n4-lib @ git+https://github.com/raisga/p4n4-lib.git"`), so a change to the
library breaks them on the next run, before it ships. The API image (`P4N4_LIB` build
argument, `p4n4-lib>=0.4.0,<0.5`) and p4n4-emu install it from PyPI, which is what users get.
This is why p4n4-lib goes first in every set: until it's published, the image build and
p4n4-emu's CI fail on a version requirement the CLI's CI already meets.

Still open: whether cli and api CI should switch to PyPI, testing exactly what users install
at the cost of finding library breakage only after a release.
