# Development

The umbrella repo's `scripts/` run a p4n4 project on the code in your checkout instead of
`pip install p4n4`. `lib`, `cli` and `api` are installed editable into one venv (`.venv` at
the repo root), so an edit applies on the next command, or on save for the API. The
dashboard runs with `flutter run` (hot reload) instead of its image, and `p4n4 init`
scaffolds from the `stacks/` and `dashboard/` checkouts.

## Prerequisites

| Tool | Needed for |
|------|------------|
| `git`, `curl`, `docker` with Compose v2 | Required |
| [`uv`](https://docs.astral.sh/uv/) | `scripts/dev setup` (the venv) |
| `flutter`, `dart` | The dashboard with hot reload and its themes (optional) |
| `gh` | Releases and CI runs (optional) |

Check the checkout and the machine first:

```bash
git submodule update --init --recursive
scripts/doctor
```

`scripts/doctor` is read-only: it checks the tools, that every submodule is checked out,
that the venv's `p4n4_lib`, `p4n4` and `p4n4_api` come from this checkout, which host ports
the stacks need are free, and whether `p4n4-net` exists. It exits 1 when a check fails
(warnings don't count), so scripts can gate on it.

## `scripts/dev`

```bash
scripts/dev setup                                       # .venv with lib, cli, api editable; flutter pub get
scripts/dev new mqtt-influx-grafana-ollama /tmp/demo    # copy a template, each layer's .env from its example
scripts/dev up /tmp/demo                                # stack + API + hot-reload dashboard
```

| Command | What it does |
|---------|--------------|
| `setup` | Creates `.venv` (Python ≥ 3.11) with `lib`, `cli` and `api` editable, and runs `flutter pub get` in `dashboard/` |
| `p4n4 <args…>` | The CLI from `cli/`. `init` gets `--source-iot/ai/edge/dashboard` pointing at the local checkouts unless you pass your own |
| `new <template> [dir]` | Copies a template from `tools/templates/projects` and creates each layer's `.env` from its `.env.example` |
| `api [project]` | p4n4-api for the project, reloading on changes in `api/` and `lib/` |
| `dashboard [project] [--theme]` | The dashboard with hot reload on `:8088` (needs the API) |
| `up [project] [--theme] [--no-stack]` | `p4n4 up` for every layer except `dashboard`, the API in the background (log in `<project>/.p4n4-api/dev-api.log`), then the dashboard in front. Ctrl-C or `q` stops the API and dashboard; the stack keeps running (`scripts/dev p4n4 down`) |
| `which` | Which projects own the `p4n4-*` containers on this host |
| `status [project]` | The project's API, dashboard and stack containers, and other projects holding containers |
| `shell` | A shell with `.venv` active, so `p4n4` and `p4n4-api` come from this checkout |

`[project]` is a p4n4 project directory; the default is the nearest directory up from the
current one with a `.p4n4.json`. To use the script from any project, symlink it onto your
`PATH`: `ln -s "$PWD/scripts/dev" ~/.local/bin/p4n4-dev`.

`setup` installs with `uv pip install -e` rather than `uv run` or `uv sync` in a package
directory, which would swap the editable `lib` for the PyPI release.

### API and dashboard in development

The API gets `P4N4_PROJECT_DIR=<project>` and keeps its data in `<project>/.p4n4-api`
(`P4N4_API_DATA_DIR`), so each project has its own user database. `P4N4_API_DEV_USERS`
defaults to `true`, which creates the demo accounts `admin`, `power` and `normie` with the
password `p4n4`. Never use those outside development.

`scripts/dev_api.py` wraps p4n4-api with auto-reload and moves the bearer token from
`X-Upstream-Authorization` back to `Authorization`, as the dashboard container's nginx does.
`flutter run`'s dev proxy (`dashboard/web_dev_config.yaml`) doesn't, so without it every API
call from a hot-reload dashboard is a 401.

`--theme` applies the project's dashboard theme (`dashboard.theme` in `.p4n4.json`). It
rewrites tracked files in `dashboard/` (`assets/brand`, `web/`); undo it with
`cd dashboard && dart tool/brand.dart apply p4n4 --web-only`.

| Variable | Default | Description |
|----------|---------|-------------|
| `P4N4_API_HOST` / `P4N4_API_PORT` | `127.0.0.1` / `8000` | Where the API listens. The dev proxy expects `:8000`. For the dashboard *container*, use the `docker0` address, e.g. `P4N4_API_HOST=172.17.0.1` |
| `P4N4_API_DEV_USERS` | `true` | Demo accounts `admin`/`power`/`normie`, password `p4n4` |
| `WEB_PORT` | `8088` | Dashboard port |

### One project at a time

Stacks use fixed container names and host ports, so only one project runs on a host at a
time. `scripts/dev which` lists the projects holding `p4n4-*` containers, and
`scripts/dev p4n4 down --all` stops them all.

## `scripts/repos`

Every submodule at a glance, without changing any working tree or index:

```bash
scripts/repos               # branch, uncommitted changes, ahead/behind upstream, commits since tag, pointer
scripts/repos fetch         # git fetch every submodule, then status
scripts/repos changes lib   # commits since the submodule's last tag (release notes material)
```

The **pointer** column compares the commit the umbrella repo records with the submodule's
`HEAD`. See [Releasing](../project/releasing.md) for the release order.
