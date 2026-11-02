# Spike S1: Compose

**Date:** 2026-11  
**Status:** Partial: desk research and static checks done; the engine checks wait for the BIOS update  
**Plan:** [§9.1](https://github.com/raisga/p4n4/blob/main/RUST_REWRITE_PLAN.md#91-spikes-p0)  
**Code:** `spikes/s1-compose/` in the superproject, until P1

## Questions

From the plan's S1 row:

1. The oldest Compose v2 that supports everything the design uses. Expected: 2.20, which added `include:`, or later. The features are:
   - `include:` with `env_file`, and with a list of paths (the port files);
   - `depends_on.required: false`;
   - profiles from `COMPOSE_PROFILES` in `.env`;
   - `--format json` for `config`, `ps` and `version`;
   - `up --wait`.
2. The root-owned `default` network named `p4n4-net`, with `ip_range` and a fixed `ipv4_address`.
3. Whether `down` removes containers of a profile that is not active, such as `sim` after `p4n4 sim start`. If not, p4n4 passes `--profile` for every service it ever started.

The spike decides the minimum Compose version (exit code 4 and `doctor`), and the recorded fixtures for `FakeDriver`.

This part covers what needs no Docker engine:
- the release notes, docs and code;
- how each version loads a project shaped like §5.3.

Anything that creates a container waits for the hands-on run.

## Setup

**Environment.**
- Windows 11, with WSL Ubuntu 24.04.2 (kernel 5.15.167.4) and python3. Nothing was compiled.
- Static checks only, with `DOCKER_HOST` pointing at a nonexistent socket (`unix:///nonexistent/docker.sock`):
  - `DOCKER_CONTEXT` and every `COMPOSE_*` variable unset;
  - an empty `DOCKER_CONFIG`;
  - each command under `timeout 60`.

  They ran on 2026-11-01.
- Docker Desktop 28.4.0 with Compose v2.39.2 is installed, but not used yet.

**Compose binaries.** 14 upstream releases, as the Linux x86_64 asset `docker-compose-linux-x86_64`. Each was checked against its `.sha256`. They are in WSL under `~/.cache/p4n4-spikes/s1-compose/bin/v<version>/docker-compose`, 793 MB in all.

| Version | Why it's in the set |
|---|---|
| 2.17.3 | A control, from before `include:` |
| 2.19.1 | The last release without `include:` |
| 2.20.0 | Adds `include:` |
| 2.20.2 | `depends_on.required` takes effect |
| 2.20.3 | An include's `env_file` takes effect |
| 2.21.0 | Handles included services whose profile is inactive; `ps` JSON becomes one object per line |
| 2.23.3 | `down SERVICE` enables the service's profile (from 2.23.0); the last release on compose-go v1 |
| 2.24.0 | Moves to compose-go v2 |
| 2.24.6 | The version that Ubuntu 24.04 shipped with |
| 2.24.7 | An include's `env_file` is resolved from the including file |
| 2.25.0 | A `config --format json` regression |
| 2.27.1 | A `.env` regression |
| 2.39.2 | Docker Desktop 28.4.0's version, as on this machine |
| 5.6.1 | The latest (2026-10-21). After 2.40.3 (2025-10-30), Compose jumped to [5.0.0][v5.0.0] (2025-12-02) |

**Ubuntu's own builds.** `docker-compose-v2` 2.24.6+ds1-0ubuntu2 (noble) and 2.40.3+ds1-0ubuntu1~24.04.1 (noble-updates).
- They are built from repacked source against Debian's Go packages, so they could differ from upstream.
- They were unpacked from their [packages][ubuntu-pkg] without installing them, and run through the same checks.
- They are in `~/.cache/p4n4-spikes/s1-compose/ubuntu/`, with their results in `ubuntu/static-matrix.txt`.

**Sources.**
- docker/compose: the release notes, and the PRs, issues and code at each tag;
- compose-spec/compose-go: the release notes and PRs;
- the Compose docs on docs.docker.com.

The links are at the end.

**The test project** is `spikes/s1-compose/project/`:
- a root `compose.yaml` with `name:`;
- three `include` entries, each with `env_file: .env`. Two of them list a layer file and then its ports file;
- `server`, with a healthcheck and a fixed address;
- `sim`, `mqtt` and `agent`, each in its own profile, with cross-layer `depends_on` that use `required: false`;
- `COMPOSE_PROFILES=mqtt,agent` in `.env`;
- the `default` network, named `p4n4-net`.

Six probes each isolate one question. `static-checks.sh` runs everything for each binary, from the project directory, the way p4n4 should: `docker compose --project-directory <dir> -f compose.yaml …`.

## Results

### Q1: The oldest version

**The features the plan uses.** "First version" means the first release where the feature works for a project like §5.3's. That's sometimes later than the release that introduced it.

| Feature | Plan | First version | Source | Confidence |
|---|---|---|---|---|
| `include:` with a list of paths | §5.3 | **2.20.0**. 2.19.1 accepts the key, but loads no services and exits with 0 | compose-go [#416][cg416]; the [reference][include-ref]; static run | High |
| An include's `env_file` | §5.3 | **2.20.3**. 2.20.0 and 2.20.2 accept it, then ignore it | compose-go [#441][cg441]; the [include how-to][include-howto] says 2.20.3; static run | High |
| An included service whose profile is inactive (`sim`) | §5.3, §6.6 | **2.21.0**. Before it:<br>• `config sim` fails with "no such service: sim";<br>• `--profile sim` fails with "depends on undefined service mqtt";<br>• `config --profiles` leaves out `sim` | compose-go [#448][cg448]; static run | High |
| `depends_on.required: false` | §5.5 (V004) | Accepted in 2.20.0; takes effect in **2.20.2**. It covers a dependency that its profile disables, never an undefined one | compose-go [#428][cg428], [#432][cg432]; [#10792]; the [docs][depends-on]; static run | High |
| `COMPOSE_PROFILES` in `.env` | D2, §5.3 | Before 2.17. 2.27.1 ignores it when the cwd isn't the project directory | [#11856], fixed by [#11861] in 2.27.2; static run | High |
| `--profile` replaces `COMPOSE_PROFILES` rather than adding to it | §9.1 (the S1 row's fallback) | 2.13.0 | [#9906]; static run | High |
| Naming a service selects it even when its profile is inactive | §6.3, §6.6 | • `config`, `up`, `logs` and the like: 2.14.0 ([#10025]); with `include:`, 2.21.0;<br>• `restart`: 2.22.0 ([#10952]);<br>• `down`: 2.23.0 ([#11108]) | Release notes; static run for `config` | High for `config`; medium for the others |
| `config --format json` | §6.1 | Before 2.17. 2.25.0 prints YAML instead | [#11627], fixed by [#11630] in 2.26.0; static run | High |
| `version --format json` | §6.1 | Before 2.17 | Static run | High |
| `ps --format json` | §6.1 | Before 2.17.<br>• A JSON array up to 2.20.3, and one object per line from 2.21.0.<br>• 2.21.0 to 2.23.x leave out `Project`, as reported for 2.21.0 and 2.23.0 | [#10918], [#10958]; [#11222], fixed by [#11223] in 2.24.0 | High; the output isn't seen yet |
| `images --format json` | §6.1 | Before 2.17. 2.37.0 alone prints a map keyed by image name, rather than a list | [#12916], fixed by [#12917] in 2.37.1 | High; the output isn't seen yet |
| `up --wait` | §6.3, §10 | 2.1.1 | [#8777]; `--help` of all 14 | High |
| Top-level `name:` | §5.3 | Before 2.17 | Static run | High |
| The `default` network: `name`, `ipam.config` with `subnet` and `ip_range`, and a fixed `ipv4_address` | §5.3 | Before 2.17 | `config` output from 2.20.0; `create.go` passes `IPRange` at [2.17.3][create-2.17.3], [2.21.0][create-2.21.0] and [2.39.2][create-2.39.2] | High for the model; the engine side is pending |
| The emulator overlay as a second `-f` over included services | §6.1, §6.6 | 2.20.0, except 2.24.0 to 2.24.5: "services.server conflicts with imported resource" | compose-go [#559][cg559], in 2.24.6; static run of 2.24.0 | High for 2.24.0; medium for 2.24.1 to 2.24.5, which weren't tested |
| A root file that redefines an included service | §5.3 | Rejected by every 2.x; merged from 5.0.0 | compose-go [#835][cg835] (v2.10.0, in 5.0.0); static run of 5.6.1 | High |
| `up --pull`, `down --volumes`, `logs --follow`, `--since` and `--tail`, `exec -T`, `run --rm`, `config --quiet` | §5.5 (V012), §6.3 | Before 2.17 | `--help` of all 14 | High |

**Not used by the plan,** for completeness:
- **YAML tags** ([merge reference][merge-ref]):
  - `!reset`: no version in the docs; it came in compose-go 1.13, so Compose 2.17 or 2.18 (medium-low confidence);
  - `!override`: 2.24.4.
- **`env_file` options** ([services reference][services-ref]): `required` from 2.24.0, and `format` from 2.30.0.
- **include `project_directory`:** in the schema from 2.20.0; its path resolution was fixed by compose-go [#614][cg614], which shipped in 2.27.0.
- **`config` flags:**
  - `--variables`: 2.26.0 according to the docs. Of the versions tested, 2.27.1 is the first whose `--help` lists it; 2.26.x wasn't tested;
  - `--hash` and `--no-interpolate`: before 2.17. `--no-interpolate` was restored in 2.24.7 ([#11556]).
- **`ls --format json`:** before 2.17.
- **`up --wait-timeout`:** 2.17.0 ([#10276]). p4n4 times its calls itself (§6.1).
- **Healthcheck `start_interval`:** 2.20.2 ([services reference][services-ref]). It also needs Engine 25.0, according to the [Dockerfile reference][dockerfile-hc].

**The static matrix.** Rows that are `ok` in every version are left out:
- `version --format json`;
- the `--help` checks for `up --wait` and `up --wait-timeout`;
- `--format` on `ps`, `images` and `ls`;
- `config --profiles`, `--no-interpolate` and `--hash`;
- the flags in the table's last row.

Every upstream binary reports `v<version>`.

```text
                  2.17.3 2.19.1 2.20.0 2.20.2 2.20.3 2.21.0 2.23.3 2.24.0 2.24.6 2.24.7 2.25.0 2.27.1 2.39.2 5.6.1
config-variables  n/a    n/a    n/a    n/a    n/a    n/a    n/a    n/a    n/a    n/a    n/a    ok     ok     ok
cfg-yaml          REJ    ok     ok     ok     ok     ok     ok     ok     ok     ok     ok     ok     ok     ok
cfg-json          REJ    ok     ok     ok     ok     ok     ok     ok     ok     ok     BAD    ok     ok     ok
json-ports        skp    BAD    ok     ok     ok     ok     ok     ok     ok     ok     skp    ok     ok     ok
json-network      skp    BAD    ok     ok     ok     ok     ok     ok     ok     ok     skp    ok     ok     ok
json-env          skp    BAD    ok     ok     ok     ok     ok     ok     ok     ok     skp    ok     ok     ok
svc-profiles      REJ    BAD    ok     ok     ok     ok     ok     ok     ok     ok     ok     ok     ok     ok
svc-no-profiles   REJ    BAD    ok     ok     ok     ok     ok     ok     ok     ok     ok     ok     ok     ok
svc-profile-star  REJ    BAD    ok     ok     ok     ok     ok     ok     ok     ok     ok     ok     ok     ok
svc-profile-sim   REJ    BAD    REJ    REJ    REJ    ok     ok     ok     ok     ok     ok     ok     ok     ok
target-sim        REJ    REJ    REJ    REJ    REJ    ok     ok     ok     ok     ok     ok     ok     ok     ok
target-sim-prof   REJ    REJ    REJ    REJ    REJ    ok     ok     ok     ok     ok     ok     ok     ok     ok
profiles-list     REJ    BAD    BAD    BAD    BAD    ok     ok     ok     ok     ok     ok     ok     ok     ok
overlay-f         REJ    REJ    ok     ok     ok     ok     ok     REJ    ok     ok     ok     ok     ok     ok
ps-json           REJ    eng    eng    eng    eng    eng    eng    eng    eng    eng    eng    eng    eng    eng
cwd-root-abs-f    REJ    BAD    REJ    REJ    REJ    REJ    REJ    REJ    REJ    ok     ok     BAD    ok     ok
cwd-root-rel-f    REJ    REJ    REJ    REJ    REJ    REJ    REJ    REJ    REJ    REJ    REJ    REJ    REJ    REJ
inc-env-file      REJ    BAD    BAD    BAD    ok     ok     ok     ok     ok     ok     ok     ok     ok     ok
dep-disabled-opt  REJ    REJ    REJ    ok     ok     ok     ok     ok     ok     ok     ok     ok     ok     ok
dep-disabled-req  REJ    REJ    REJ    REJ    REJ    REJ    REJ    REJ    REJ    REJ    REJ    REJ    REJ    REJ
dep-undefined-opt REJ    REJ    REJ    REJ    REJ    REJ    REJ    REJ    REJ    REJ    REJ    REJ    REJ    REJ
dep-generated     REJ    BAD    ok     ok     ok     ok     ok     ok     ok     ok     ok     ok     ok     ok
root-redefines    REJ    REJ    REJ    REJ    REJ    REJ    REJ    REJ    REJ    REJ    REJ    REJ    REJ    ok
```

The codes:
- `ok`: accepted. For the rows that check the output, it's also as expected. `cfg-yaml` checks only the exit status;
- `REJ`: rejected;
- `BAD`: accepted, with a wrong result;
- `eng`: needs the engine;
- `n/a`: the flag isn't in `--help`;
- `skp`: a prerequisite failed.

The rows:
- **`cfg-yaml` and `cfg-json`.** `config`, and `config --format json`.
- **`json-ports`, `json-network` and `json-env`.** Whether that JSON holds, in order:
  - the ports from the ports files;
  - the network's `name`, `subnet` and `ip_range`, with `server`'s address;
  - the values from `.env`.
- **`svc-profiles` and `svc-no-profiles`.** `config --services`, with and without `COMPOSE_PROFILES` in `.env`.
- **`svc-profile-star` and `svc-profile-sim`.** `config --services`:
  - with `--profile '*'`, expecting all four services;
  - with `--profile sim`, expecting only `server` and `sim`.
- **`target-sim` and `target-sim-prof`.** `config sim`, without and with `--profile sim`.
- **`profiles-list`.** `config --profiles`.
- **`overlay-f`.** The emulator overlay as a second `-f`.
- **`ps-json`.** `ps --format json`.
- **`cwd-root-abs-f` and `cwd-root-rel-f`.** `config --services` run from the cwd `/`, first with an absolute `-f`, then with a relative one.
- **The probes:**
  - `inc-env-file`: an include's `env_file` other than the root `.env`;
  - `dep-disabled-opt` and `dep-disabled-req`: a dependency that its profile disables, with and without `required: false`;
  - `dep-undefined-opt`: `required: false` on an undefined service;
  - `dep-generated`: the V004 fix below;
  - `root-redefines`: the root file adds ports to an included service.

**What the matrix shows.**

- **The version gate must be p4n4's own.** 2.17.3 rejects `include:`, but 2.19.1 accepts the key and loads an empty project with exit code 0. Its `config` "passes" while it ignores every layer, so Compose's own errors can't stand in for the version check.
- **2.21.0 is the oldest version that passes every check that runs the main project from its own directory,** apart from `ps-json`, which needs the engine. 2.20.x drops an included service whose profile is inactive instead of disabling it. So `sim` can't be named, `required: false` can't cover it, and `config --profiles` doesn't list it. `p4n4 sim start` needs all three.
- **Three later regressions.**
  - 2.24.0 rejects the emulator overlay. Going by the fix's date, so do 2.24.1 to 2.24.5.
  - 2.25.0 prints YAML for `config --format json`.
  - 2.27.1 loses `COMPOSE_PROFILES` from `.env` when the cwd is elsewhere. That doesn't matter if p4n4 runs Compose from the project directory.
- **The cwd matters.**
  - `-f` is resolved against the cwd, not against `--project-directory`, in every version (`cwd-root-rel-f`).
  - An include's relative `env_file` is resolved against the cwd until 2.24.6, and against the including file from 2.24.7 (compose-go [#582][cg582]). Run from the cwd `/`, 2.20.0 to 2.24.6 stop with "Couldn't find env file: /.env".

  So §6.1's `--project-directory <dir> -f compose.yaml` works only if Compose also runs with `<dir>` as its cwd.
- **V004 can't work as written.** From 2.20.2, `required: false` covers a dependency that its profile disables (`dep-disabled-opt`). It never covers a service that no file defines, and that is what an absent layer leaves behind. `dep-undefined-opt` fails in all 14 versions, 5.6.1 included, with "service agent depends on undefined service mqtt".

  The fix works from 2.20.0 (`dep-generated`):
  - the layer files declare no cross-layer `depends_on`;
  - `sync` writes them, with `required: false`, into a generated file in the dependent layer's include entry;
  - it does so only while the other layer is present.
- **A root file can't redefine an included service in 2.x.** Every 2.x rejects it, as §5.3 says; 5.x merges it. The ports files are still needed.

**Version strings.** The upstream binaries print `v<version>`, for example `{"version":"v2.39.2"}`. The parser must also accept:
- Docker Desktop's `v2.39.2-desktop.1`, as quoted in [#13399]. Its JSON form isn't seen yet;
- Ubuntu's `2.24.6+ds1-0ubuntu2` and `2.40.3+ds1-0ubuntu1~24.04.1`, which have no `v` (static run);
- 2.32.3's commit hash, which it prints in place of the version ([release notes][v2.32.3]; fixed in 2.32.4). An unparsable version should therefore warn rather than stop;
- major version 5.

Ubuntu 24.04's two builds give the same matrix as upstream 2.24.6 and 2.39.2, so the distribution's packaging changes nothing here.

**Fixtures.** `fixtures/v2.39.2/` and `fixtures/v2.21.0/` hold `config.json` and `version.json`. The script would replace any work path with `/srv/p4n4s1`, but this project's output has none. A parser must allow for these details:
- 2.21.0 writes `"external": false` under `networks.default`; 2.39.2 leaves it out.
- `published` is a string and `target` a number. `entrypoint` and `networks.default` can be `null`.
- Strings use Go's HTML escapes (`\u0026`, `\u003e`).
- Services of inactive profiles, such as `sim`, are missing. The full model needs `--profile '*'`.
- In `ps` JSON, `CreatedAt` uses Go's time format, not RFC 3339, and that won't change ([#12520], closed as not planned).

**Left for the hands-on run:**
- **`up --wait`** on 2.21.0, 2.39.2 and 5.6.1:
  - `agent` waits for `server`'s healthcheck;
  - the exit status when a healthcheck fails.
- **`ps --format json` fixtures:**
  - NDJSON on 2.21.0;
  - `Project` present from 2.24.0;
  - the output when nothing is running.
- **`images --format json` fixtures.**
- **Docker Desktop's `docker compose version --format json`.**

### Q2: The `p4n4-net` network

- **The model holds up.** From 2.20.0, apart from 2.25.0, `config --format json` keeps the network's `name`, `subnet` and `ip_range`, and `server`'s `ipv4_address` (`json-network`). `pkg/compose/create.go` passes `IPRange` to the Engine at every tag read (see the table).
- **A leftover network makes `up` fail.** Since 2.19.0 ([#10612], [#10639]), when a network with that name exists, Compose compares its `com.docker.compose.network` label with the network's key, `default`. On a mismatch it stops with: "network p4n4-net was found but has incorrect label com.docker.compose.network set to "p4n4-net"". See `create.go` [L1097 at 2.21.0][label-2.21.0] and [L1293 at 2.39.2][label-2.39.2].
  - v0.4's CLI creates `p4n4-net` with that label set to `p4n4-net` (`lib/p4n4_lib/compose.py`, L121–176).
  - A plain `docker network create` sets no label.

  `up` fails in both cases. A network with the right label but another project's, such as a second p4n4 project's (K8), only gets a warning, and Compose joins it.
- **The plan's leftover checks miss v0.4's network.** §6.3's `doctor` looks for "an external `p4n4-net` network without Compose labels", and K9 and §14 item 3 speak of an "unlabeled" one. v0.4's network has the network label, set to another value, and has no project label.
- **What a subnet change does depends on the version.**
  - Before 2.31.0, Compose uses an existing network as it is. A new subnet or `ip_range` in `p4n4.toml` has no effect until the network is removed.
  - From 2.31.0, Compose detects the change and recreates the network, stopping the containers attached to it ([#12267]).
  - 2.37.0 fixed that recreation for the connected containers ([#12899]).
- **Nothing else changed.** No release note from 2.17.0 to 5.6.1 mentions another change to how the `default` network handles `name`, `ip_range` or `ipv4_address`. 5.5.1 passes `ipam.options` to the Engine ([#14094]), which the plan doesn't use.

**Left for the hands-on run:**
- **`docker network inspect` after `up`.** Check:
  - the name;
  - `IPAM.Config`, including `IPRange`;
  - the labels: `com.docker.compose.network=default` and `com.docker.compose.project=p4n4s1-main`;
  - `server` at 10.201.1.2, and the other containers inside 10.201.1.128/25.
- **The leftover case,** on a throwaway network with v0.4's labels (never the real `p4n4-net`), and the exact error.
- **A subnet change** between two `up`s, on 2.21.0 and 2.39.2.
- **The error for an overlapping subnet,** for the driver's error mapping.

### Q3: `down` and inactive profiles

- **The docs say no.** According to the [profiles how-to][profiles], `down` stops the services of the active profiles and the services without a profile. In its example, "Running `docker compose down` only stops `backend` and `db`", and not the profiled `phpmyadmin`.
- **v0.4 already relies on this.**
  - v0.4.2's `down` runs `--profile '*' down`. Its comment says that "`down` only stops the ones in active profiles" (`lib/p4n4_lib/compose.py`, L81–89).
  - Its `down_project` (L403–420) stops a project by name. It runs `-p NAME down` from an empty directory, because "By name alone Compose finds the project's containers, including those of every profile, through their labels".
- **What that means for 1.0.** §6.6 says the `sim` service "shares the project's network, logs and `down`". Suppose `p4n4 sim start` runs `up -d sim`, and a plain `p4n4 down` follows. Then `down` should leave `p4n4-sim` running, and fail to remove `p4n4-net` while `sim` is attached. That is expected, but not observed yet.
- **The release notes.** Naming a service selects it even when its profile is inactive:
  - for `up`, `config` and the like, since 2.14.0 ([#10025]);
  - for `restart SERVICE`, since 2.22.0 ([#10952]);
  - for `down SERVICE`, since 2.23.0 ([#11108]).

  No release note from 2.17.0 to 5.6.1 mentions a change to what a plain `down` does with services of inactive profiles.
- **`--profile '*'` activates every profile.** It works with `include:` from 2.20.0 (`svc-profile-star`). But `--profile` replaces `COMPOSE_PROFILES` rather than adding to it ([#9906]): `svc-profile-sim` gives only `server` and `sim`. So passing `--profile sim` alone would drop every profile from `.env`.
- **Proposal.**
  - p4n4 passes `--profile '*'` to the commands that act on existing containers: `down`, `stop`, `restart`, `logs`, `ps` and `exec`.
  - It never passes it to `up` or `pull`, which would start or pull everything.
  - `p4n4 sim start` names the service (`up -d sim`), with no `--profile`.

  This replaces "passes `--profile` for every service it ever started", and leaves no list to keep. `--profile '*'` reaches only services that are still defined. The containers of a layer that `p4n4 remove` dropped from `compose.yaml` are orphans, and only `down --remove-orphans` removes them.

**Left for the hands-on run,** on 2.21.0, 2.23.3 and 2.39.2:
- **`ps`.** After `up -d` and `up -d sim`, does `ps --format json` list `p4n4s1-sim` without `--profile`? 2.24.0 added `ps --orphans` ([#11220]).
- **A plain `down`.** Is `p4n4s1-sim` left running? Is the network error fatal, or a warning?
- **`--profile '*' down`.** Everything should go, network included.
- **Named services.** Run `restart sim`, `logs sim` and `down sim` on 2.21.0. If they fail before 2.22.0 and 2.23.0, `--profile '*'` must cover them, or the minimum moves.
- **Orphans.** After removing an include entry, check what these do with that layer's containers:
  - `down`;
  - `down --remove-orphans`;
  - `up`;
  - `-p NAME down` from an empty directory, as v0.4's `down_project` runs it. Check the network too.

## Decision

Not decided yet: the engine checks may move the minimum.

**Leaning:** a minimum of **2.21.0** (2023-08-30). It's set by the handling of included services whose profile is inactive (compose-go [#448][cg448]), which `p4n4 sim start` relies on. With that minimum, `ps` JSON is always NDJSON.

The exit-code-4 check and `doctor` would also refuse:
- 2.24.0 to 2.24.5 (the emulator overlay);
- 2.25.0 (`config --format json`);
- 2.37.0, unless the `images` parser accepts its map.

Ubuntu 24.04's 2.24.6 and 2.40.3, whose static results match upstream's, and Docker Desktop's 2.39.2 all clear this gate.

## Follow-ups

**Proposed plan changes:**
- **Wording: §2.4, §9.1 and §12.2.** The major version after 2.40.3 was 5.0.0. Write "Compose 2.21 or later" rather than "Compose v2". The version parser and the e2e "latest" must accept 5.x.
- **§6.1, errors.**
  - Parse versions with and without the `v`, with `-desktop.N` suffixes and with Debian `+ds1…` suffixes.
  - Refuse versions below the minimum, and the denylist, with code 4.
  - Warn on an unparsable version.
- **§6.1, addressing.** Run Compose with the project directory as its cwd, as well as passing `--project-directory <dir> -f compose.yaml`.
- **§6.1, output parsing.**
  - With 2.21 as the minimum, `ps --format json` is NDJSON only, and may be empty. The "JSON array or NDJSON" in §6.1 can go.
  - `config` leaves out services of inactive profiles unless it gets `--profile '*'`.
- **§5.5 (V004) and §5.3.** Layer files declare no cross-layer `depends_on`. `sync` writes them, with `required: false`, into a generated file in the dependent layer's include entry, while the other layer is present. `.p4n4/ports/<layer>.yaml` can hold them, since `sync` already lists it in that entry. Its name would then undersell it.
- **§5.3.** Qualify "Compose rejects a root file that redefines an included service" with "before 5.0". The ports files stay.
- **§6.3, §6.6 and the S1 row in §9.1.**
  - Pass `--profile '*'` to `down`, `stop`, `restart`, `logs`, `ps` and `exec`, as v0.4's `down` already does.
  - `sim start` names its service.
  - Without this, §6.6's "shares the project's … logs and `down`" doesn't hold.

  The details wait for the hands-on run.
- **§6.3 (`doctor`, and `up`'s leftover checks), K9 in §11, and §14 item 3.** Flag any `p4n4-net` that this project doesn't own, by its `com.docker.compose.project` and `com.docker.compose.network=default` labels, rather than only an unlabeled or external one. v0.4's network makes `up` fail. §12.1's upgrade steps already remove it.
- **§5.2, subnet changes.** Decide what p4n4 does when the subnet or `server_ip` changes while the network exists. Compose ignores the change before 2.31.0, and recreates the network from 2.31.0. For example, `up` could stop with a hint to run `down` first.
- **§8 and §10, e2e.** Run on 2.21.0 and on the latest 5.x.
- **§11, K2.** Cite the regressions found (2.24.0 to 2.24.5, 2.25.0, 2.27.1, 2.32.3 and 2.37.0) as the reason for per-version fixtures and a denylist.

**For the hands-on run:**
- **Check the engine first, read-only.** The owner's v0.4 `p4n4-net` and containers may be there.
  - If `p4n4-net` exists, set `P4N4S1_NETWORK=p4n4s1-net`.
  - Never touch `p4n4-net`, or anything not named `p4n4s1-*`.
  - Check that 10.201.1.0/24 and ports 18100 to 18110 are free.
- **Record the fixtures.** Put the `ps` and `images` fixtures next to the `config` ones. Add a `--profile '*' config --format json` fixture for the full model.
- **Finish the note.** Fill in each "Left for the hands-on run" list, then record the decision and set the status to Done.

[v5.0.0]: https://github.com/docker/compose/releases/tag/v5.0.0
[v2.32.3]: https://github.com/docker/compose/releases/tag/v2.32.3
[ubuntu-pkg]: https://launchpad.net/ubuntu/+source/docker-compose-v2
[include-ref]: https://docs.docker.com/reference/compose-file/include/
[include-howto]: https://docs.docker.com/compose/how-tos/multiple-compose-files/include/
[depends-on]: https://docs.docker.com/reference/compose-file/services/#depends_on
[services-ref]: https://docs.docker.com/reference/compose-file/services/
[merge-ref]: https://docs.docker.com/reference/compose-file/merge/
[dockerfile-hc]: https://docs.docker.com/reference/dockerfile/#healthcheck
[profiles]: https://docs.docker.com/compose/how-tos/profiles/#stop-application-and-services-with-specific-profiles
[cg416]: https://github.com/compose-spec/compose-go/pull/416
[cg428]: https://github.com/compose-spec/compose-go/pull/428
[cg432]: https://github.com/compose-spec/compose-go/pull/432
[cg441]: https://github.com/compose-spec/compose-go/pull/441
[cg448]: https://github.com/compose-spec/compose-go/pull/448
[cg559]: https://github.com/compose-spec/compose-go/pull/559
[cg582]: https://github.com/compose-spec/compose-go/pull/582
[cg614]: https://github.com/compose-spec/compose-go/pull/614
[cg835]: https://github.com/compose-spec/compose-go/pull/835
[#8777]: https://github.com/docker/compose/pull/8777
[#9906]: https://github.com/docker/compose/pull/9906
[#10025]: https://github.com/docker/compose/pull/10025
[#10276]: https://github.com/docker/compose/pull/10276
[#10612]: https://github.com/docker/compose/pull/10612
[#10639]: https://github.com/docker/compose/pull/10639
[#10792]: https://github.com/docker/compose/pull/10792
[#10918]: https://github.com/docker/compose/pull/10918
[#10952]: https://github.com/docker/compose/pull/10952
[#10958]: https://github.com/docker/compose/issues/10958
[#11108]: https://github.com/docker/compose/pull/11108
[#11220]: https://github.com/docker/compose/pull/11220
[#11222]: https://github.com/docker/compose/issues/11222
[#11223]: https://github.com/docker/compose/pull/11223
[#11556]: https://github.com/docker/compose/pull/11556
[#11627]: https://github.com/docker/compose/issues/11627
[#11630]: https://github.com/docker/compose/pull/11630
[#11856]: https://github.com/docker/compose/issues/11856
[#11861]: https://github.com/docker/compose/pull/11861
[#12267]: https://github.com/docker/compose/pull/12267
[#12520]: https://github.com/docker/compose/issues/12520
[#12899]: https://github.com/docker/compose/pull/12899
[#12916]: https://github.com/docker/compose/issues/12916
[#12917]: https://github.com/docker/compose/pull/12917
[#13399]: https://github.com/docker/compose/issues/13399
[#14094]: https://github.com/docker/compose/pull/14094
[create-2.17.3]: https://github.com/docker/compose/blob/v2.17.3/pkg/compose/create.go#L1149
[create-2.21.0]: https://github.com/docker/compose/blob/v2.21.0/pkg/compose/create.go#L1172
[create-2.39.2]: https://github.com/docker/compose/blob/v2.39.2/pkg/compose/create.go#L1390
[label-2.21.0]: https://github.com/docker/compose/blob/v2.21.0/pkg/compose/create.go#L1097
[label-2.39.2]: https://github.com/docker/compose/blob/v2.39.2/pkg/compose/create.go#L1293
