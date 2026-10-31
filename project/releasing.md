# Releasing

Each repository keeps its GitHub release bodies in its own `release-notes/` folder, one file
per release named `v<version>.md`, passed to `gh release create --notes-file`. This page holds
what spans repositories: the order of a release set and how to write a body.

## Past release sets

Each was published in the table's order: p4n4-lib first, then the two images together, then
the CLI. The stacks, templates, emu, hw and docs keep their own version lines and got no
release; the root repo records their commits.

| Set | p4n4-lib | p4n4-api | p4n4-dashboard | p4n4 (CLI) | Theme |
|---|---|---|---|---|---|
| v0.4.2 (2026-10-29) | [0.4.2](https://github.com/raisga/p4n4-lib/blob/main/release-notes/v0.4.2.md) | [0.4.0](https://github.com/raisga/p4n4-api/blob/main/release-notes/v0.4.0.md) | [1.5.0](https://github.com/raisga/p4n4-dashboard/blob/main/release-notes/v1.5.0.md) | [0.4.2](https://github.com/raisga/p4n4-cli/blob/main/release-notes/v0.4.2.md) | Incident investigator's settings (with p4n4-agent 0.1.0, tag `agent-v0.1.0` in `stacks/ai`); services inside the dashboard, the Feed's review page, pictures and links |
| v0.4.0 (2026-10-16) | [0.4.0](https://github.com/raisga/p4n4-lib/blob/main/release-notes/v0.4.0.md) | [0.3.0](https://github.com/raisga/p4n4-api/blob/main/release-notes/v0.3.0.md) | [1.3.0](https://github.com/raisga/p4n4-dashboard/blob/main/release-notes/v1.3.0.md) | [0.4.0](https://github.com/raisga/p4n4-cli/blob/main/release-notes/v0.4.0.md) | Edge merged into ai, extras layer, `p4n4 migrate edge`; kiosk mode, view settings, vision overlay |
| v0.3.0 (2026-10-14) | [0.3.0](https://github.com/raisga/p4n4-lib/blob/main/release-notes/v0.3.0.md) | [0.2.0](https://github.com/raisga/p4n4-api/blob/main/release-notes/v0.2.0.md) | [1.2.0](https://github.com/raisga/p4n4-dashboard/blob/main/release-notes/v1.2.0.md) | [0.3.0](https://github.com/raisga/p4n4-cli/blob/main/release-notes/v0.3.0.md) | System 1 (with p4n4-system1 0.1.0, tag `system1-v0.1.0` in `stacks/ai`); assistant, views, Feed, Home |
| v0.2.0 (2026-10-03) | [0.2.0](https://github.com/raisga/p4n4-lib/blob/main/release-notes/v0.2.0.md) | [0.1.0](https://github.com/raisga/p4n4-api/blob/main/release-notes/v0.1.0.md) | [1.1.0](https://github.com/raisga/p4n4-dashboard/blob/main/release-notes/v1.1.0.md) | [0.2.0](https://github.com/raisga/p4n4-cli/blob/main/release-notes/v0.2.0.md) | Multi-layer projects, p4n4-lib on PyPI, sign-in |

p4n4-lib and p4n4 (CLI) 0.4.1 (2026-10-17) were a fix release of the two: `p4n4-net` on the
project's `P4N4_SUBNET`. p4n4 (CLI) 0.2.1 (2026-10-12) was a release of the CLI alone.
p4n4-lib 0.2.1, p4n4-dashboard 1.1.1 and 1.4.0 were never published: their changes shipped
in p4n4-lib 0.3.0, p4n4-dashboard 1.2.0 and 1.5.0.

### How the current versions fit together

| | needs | used by |
|---|---|---|
| p4n4-lib 0.4.2 | — | api 0.4.0 and p4n4-emu (`p4n4-lib>=0.4.0`), CLI 0.4.2 (`>=0.4.2`) |
| p4n4-api 0.4.0 | p4n4-lib 0.4.0 or later | dashboard 1.5.0 (Feed pictures, links and `lang`; 1.3.0 also works) |
| p4n4-dashboard 1.5.0 | p4n4-api 0.4.0 | the dashboard layer `p4n4 init` scaffolds (`DASHBOARD_VERSION=1.5.0`) |
| p4n4 (CLI) 0.4.2 | p4n4-lib 0.4.2, p4n4-system1 0.1.0 | — |

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
