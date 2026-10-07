# Releasing

Each repository keeps its GitHub release bodies in its own `release-notes/` folder, one file
per release named `v<version>.md`, passed to `gh release create --notes-file`. This page holds
what spans repositories: the order of a release set and how to write a body.

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
