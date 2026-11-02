# Spike S7: YAML

**Date:** 2026-11  
**Status:** Partial: desk research done; the hands-on checks wait for the BIOS update  
**Plan:** [§9.1](https://github.com/raisga/p4n4/blob/main/RUST_REWRITE_PLAN.md#91-spikes-p0)  
**Code:** `spikes/s7-yaml/` in the superproject, until P1; it doesn't exist yet

## Questions

From the plan's S7 row:

> Candidates: `serde-saphyr` and `serde_norway`. Each needs source spans for diagnostics, anchors and merge keys, typed deserialization of layer Compose files, and serialization of `compose.yaml` and the emulator overlay.

S7 decides [Q1](https://github.com/raisga/p4n4/blob/main/RUST_REWRITE_PLAN.md#13-open-questions), "Which YAML crate: `serde-saphyr` or `serde_norway`?", whose proposed default is "The one that passes spike S7". This note is the desk half. A later run executes the cases it lists.

## Setup

Nothing was compiled and no Docker command ran, because compiles crash this machine's CPU (Windows 11 with WSL Ubuntu 24.04) until a BIOS update.

- **Crates:** `serde-saphyr` 2.0.0 and `serde_norway` 0.9.42, the latest releases, read from their `.crate` files. References such as `de.rs` L12 are to each crate's `src/`, which docs.rs also serves ([serde-saphyr](https://docs.rs/crate/serde-saphyr/2.0.0/source/src/), [serde_norway](https://docs.rs/crate/serde_norway/0.9.42/source/src/)). Metadata comes from the crates.io and GitHub APIs.
- **Workspace:** Rust 1.99.1, `rust-version = "1.99"`, edition 2024.
- **Inputs:** v0.4.2, inventoried with PyYAML 6.0.3's `yaml.parse` and `yaml.compose` under Python 3.10, plus one overlay rendered by v0.4.2's `render_overlay` in WSL.
- **Compose's YAML reader**, read from source but not run:

| Compose | compose-go | go-yaml |
|---|---|---|
| v2.21.0, the minimum [S1](s1-compose.md) leans to | v1.18.3 | `gopkg.in/yaml.v3` v3.0.1 |
| v2.39.2 | v2.8.1 | `go.yaml.in/yaml/v3` v3.0.4 |
| v5.6.1 (2026-10-21), the latest | v2.16.1 | `go.yaml.in/yaml/v4` v4.0.0-rc.6 ([go.mod](https://github.com/docker/compose/blob/v5.6.1/go.mod)) |

go-yaml's rules were read in v3.0.4 and v4.0.0-rc.6. v3.0.1 is assumed to match v3.0.4.

## Results

### The inputs

Compose files are read at run time. The rest go once through `xtask convert` ([§4.1](https://github.com/raisga/p4n4/blob/main/RUST_REWRITE_PLAN.md#41-repository-layout)) and are TOML afterwards. The inventory covers 42 files and 4,025 lines, one document per file, with no `---` and no directives:

- 21 Compose files: 7 in `stacks/` (with obd2's `override.example.yml` and `tests/compose.smoke.yml`), 13 in `tools/templates/projects/`, and `cli/test-project/docker-compose.yml`;
- 10 emulator and simulator files: the 7 profiles, `tools/emu/examples/sim-edge.yml` and `sim-scenario.yml`, and obd2's scenario `warmup.yaml`;
- 10 metadata files: `extra.yaml` ×2, `template.yaml` ×7 and `stacks/extras/schema/licenses.yaml`;
- the overlay that v0.4.2 renders for the `jetson-orin-nano` profile and the `ai` stack.

Left out: CI workflows, Grafana provisioning and `loki.yml` (copied as they are), `lib/p4n4_lib/sources.yaml` (dropped), `stack.emu.yml.j2` (replaced by typed structs) and the dashboard's own Compose files.

| Feature | Count | Where |
|---|--:|---|
| Anchors / aliases | 1 / 8 | `x-logging: &logging` (`stacks/iot/docker-compose.yml` L12), used by 8 services |
| Merge keys (`<<`) | 0 | |
| Top-level `x-` keys | 2 | `x-logging`, and `x-p4n4-emu` in the overlay |
| Tags | 1 | `ports: !reset []` (`obd2/tests/compose.smoke.yml` L6) |
| Block scalars | 10 | All `>-`: `description` in the 9 metadata files other than `licenses.yaml`, and `payload` (`obd2/extra.yaml` L47) |
| Duplicate keys | 0 | |
| Non-string keys | 0 | 2 under YAML 1.1: the `off` keys below |
| Empty values / `~` or `null` | 65 / 0 | 64 named volumes, and `raw:` (`sim-scenario.yml` L22) |
| Plain scalars that YAML 1.1 and 1.2 resolve differently | 2 | The `off` keys below |

**The one YAML 1.1 hazard** is at L12–13 of `stacks/extras/obd2/simulator/scenarios/warmup.yaml`:

```yaml
voltage: {running: 14.1, off: 12.4}
off: []                       # [[from, to], ...]: seconds with the engine off
```

PyYAML, a YAML 1.1 parser, reads both keys as `False`. YAML 1.2 and go-yaml read "off", as do both crates for a struct field or a `String` key. So v0.4.2's `elm327_sim.py`, which loads the file with `yaml.safe_load` (L98) and looks up `"off"` (L88–89), ignores both values. They equal the built-in defaults, so nothing shows today. After `xtask convert`, Rust applies them, and the two simulators diverge as soon as someone edits them.

Nothing else differs: there are no plain yes/no/on/off values, and no `0644`, `1e3`, `1_000`, hex or timestamps. All 50 `ports` entries are quoted, and `1883:1883` wouldn't be sexagesimal anyway, because that needs 0–59 after each colon.

**50 strings must stay quoted,** or they change type: 31 `environment` values (`'true'` ×12, `'false'` ×9, `'1'` ×7, `''` ×2 and `'0'`), three `deploy` limits (`'0.5'` ×2 and `'2.0'`), `restart: "no"` (L120 of the letta template's `ai/docker-compose.yml`), and in the overlay five `cpus` such as `"1.04"` and ten blkio `rate: "524288000"`.

**Interpolation.** 589 lines of the Compose files hold `${…}`, all in string positions. None is in a typed field such as `retries` yet, but Compose interpolates before it checks types, so any field may hold one.

**How Compose reads them.** compose-go decodes each file untyped, interpolates, then casts known fields. go-yaml resolves plain scalars as YAML 1.2 does, except that:

- `0644` is legacy octal, 420, and `_` is dropped from numbers (v3.0.4 [`resolve.go` L234](https://github.com/yaml/go-yaml/blob/v3.0.4/resolve.go#L234); v4 [`resolver.go` L160–161](https://github.com/yaml/go-yaml/blob/v4.0.0-rc.6/internal/libyaml/resolver.go#L160-L161));
- a typed Go `bool` accepts y/yes/on and n/no/off (v3.0.4 [`decode.go` L684–692](https://github.com/yaml/go-yaml/blob/v3.0.4/decode.go#L684-L692); v4 [`constructor.go` L184–193](https://github.com/yaml/go-yaml/blob/v4.0.0-rc.6/internal/libyaml/constructor.go#L184-L193)), while an untyped `off` stays "off";
- base 60 isn't supported ([v4 `resolver.go` L134–136](https://github.com/yaml/go-yaml/blob/v4.0.0-rc.6/internal/libyaml/resolver.go#L134-L136)).

compose-go's `toBoolean` then turns the YAML 1.1 words into booleans, with a warning, for a fixed list of fields ([v2.16.1 `interpolate.go` L133–148](https://github.com/compose-spec/compose-go/blob/v2.16.1/loader/interpolate.go#L133-L148)). It rejects duplicate keys ([`reset.go` L193–194](https://github.com/compose-spec/compose-go/blob/v2.16.1/loader/reset.go#L193-L194)) and non-string keys ([`loader.go` L984](https://github.com/compose-spec/compose-go/blob/v2.16.1/loader/loader.go#L984)).

<details>
<summary>Per-file inventory</summary>

`templates/` is `tools/templates/projects/`, `profiles/` is `tools/emu/p4n4_emu/profiles/definitions/`, and `emu/examples/` is `tools/emu/examples/`. "Must stay quoted" counts quoted strings that would change type if written plain; "1.1 ≠ 1.2" counts plain scalars that YAML 1.1 and 1.2 resolve differently.

| File | Lines | Anchors / aliases | `x-` keys | Tags | Block scalars | Empty values | Must stay quoted | 1.1 ≠ 1.2 |
|---|--:|---|---|---|---|--:|--:|--:|
| `stacks/ai/docker-compose.yml` | 346 | — | — | — | — | 6 | 8 | — |
| `stacks/extras/_template/docker-compose.yml` | 53 | — | — | — | — | 1 | — | — |
| `stacks/extras/docker-compose.yml` | 26 | — | — | — | — | — | — | — |
| `stacks/extras/obd2/docker-compose.yml` | 85 | — | — | — | — | — | — | — |
| `stacks/extras/obd2/override.example.yml` | 14 | — | — | — | — | — | — | — |
| `stacks/extras/obd2/tests/compose.smoke.yml` | 28 | — | — | `!reset` | — | — | — | — |
| `stacks/iot/docker-compose.yml` | 342 | 1 / 8 | `x-logging` | — | — | 7 | 1 | — |
| `templates/mqtt-influx-grafana-n8n/ai/docker-compose.yml` | 125 | — | — | — | — | 1 | 5 | — |
| `templates/mqtt-influx-grafana-n8n/iot/docker-compose.yml` | 192 | — | — | — | — | 5 | 1 | — |
| `templates/mqtt-influx-grafana-ollama-go2rtc/ai/docker-compose.yml` | 91 | — | — | — | — | 1 | 1 | — |
| `templates/mqtt-influx-grafana-ollama-go2rtc/extras/docker-compose.yml` | 87 | — | — | — | — | 1 | 1 | — |
| `templates/mqtt-influx-grafana-ollama-go2rtc/iot/docker-compose.yml` | 207 | — | — | — | — | 5 | 7 | — |
| `templates/mqtt-influx-grafana-ollama-letta/ai/docker-compose.yml` | 158 | — | — | — | — | 2 | 5 | — |
| `templates/mqtt-influx-grafana-ollama-letta/iot/docker-compose.yml` | 195 | — | — | — | — | 5 | 1 | — |
| `templates/mqtt-influx-grafana-ollama-pidurable/ai/docker-compose.yml` | 118 | — | — | — | — | 2 | 1 | — |
| `templates/mqtt-influx-grafana-ollama-pidurable/iot/docker-compose.yml` | 186 | — | — | — | — | 5 | — | — |
| `templates/mqtt-influx-grafana-ollama/ai/docker-compose.yml` | 87 | — | — | — | — | 1 | 1 | — |
| `templates/mqtt-influx-grafana-ollama/iot/docker-compose.yml` | 187 | — | — | — | — | 5 | 1 | — |
| `templates/mqtt-influx-grafana/docker-compose.yml` | 187 | — | — | — | — | 5 | 1 | — |
| `templates/mqtt-nodered-influx-grafana/docker-compose.yml` | 234 | — | — | — | — | 6 | 1 | — |
| `cli/test-project/docker-compose.yml` | 148 | — | — | — | — | 6 | — | — |
| Overlay, rendered (`jetson-orin-nano`, `ai`) | 117 | — | `x-p4n4-emu` | — | — | — | 15 | — |
| `profiles/jetson-orin-nano.yml` | 15 | — | — | — | — | — | — | — |
| `profiles/mcu-class.yml` | 13 | — | — | — | — | — | — | — |
| `profiles/nuc.yml` | 9 | — | — | — | — | — | — | — |
| `profiles/rpi-zero2w.yml` | 13 | — | — | — | — | — | — | — |
| `profiles/rpi3.yml` | 15 | — | — | — | — | — | — | — |
| `profiles/rpi4.yml` | 12 | — | — | — | — | — | — | — |
| `profiles/rpi5.yml` | 12 | — | — | — | — | — | — | — |
| `emu/examples/sim-edge.yml` | 37 | — | — | — | — | — | — | — |
| `emu/examples/sim-scenario.yml` | 29 | — | — | — | — | 1 | — | — |
| `stacks/extras/obd2/simulator/scenarios/warmup.yaml` | 29 | — | — | — | — | — | — | 2 |
| `stacks/extras/_template/extra.yaml` | 39 | — | — | — | 1 `>-` | — | — | — |
| `stacks/extras/obd2/extra.yaml` | 82 | — | — | — | 2 `>-` | — | — | — |
| `templates/mqtt-influx-grafana-n8n/template.yaml` | 77 | — | — | — | 1 `>-` | — | — | — |
| `templates/mqtt-influx-grafana-ollama-go2rtc/template.yaml` | 79 | — | — | — | 1 `>-` | — | — | — |
| `templates/mqtt-influx-grafana-ollama-letta/template.yaml` | 81 | — | — | — | 1 `>-` | — | — | — |
| `templates/mqtt-influx-grafana-ollama-pidurable/template.yaml` | 72 | — | — | — | 1 `>-` | — | — | — |
| `templates/mqtt-influx-grafana-ollama/template.yaml` | 64 | — | — | — | 1 `>-` | — | — | — |
| `templates/mqtt-influx-grafana/template.yaml` | 47 | — | — | — | 1 `>-` | — | — | — |
| `templates/mqtt-nodered-influx-grafana/template.yaml` | 59 | — | — | — | 1 `>-` | — | — | — |
| `stacks/extras/schema/licenses.yaml` | 28 | — | — | — | — | — | — | — |

</details>

**Left for the hands-on run:** load all 42 files with both crates, as the test plan below describes.

### The candidates

| | `serde-saphyr` | `serde_norway` |
|---|---|---|
| Version examined | 2.0.0, 2026-10-13 | 0.9.42, 2024-12-21 |
| Releases in the last 12 months | 30: 1.0.0 on 2026-07-31, then 2.0.0 ten weeks later | None |
| Licence | MIT OR Apache-2.0 | MIT OR Apache-2.0; its `unsafe-libyaml-norway` is MIT |
| `rust-version` | 1.89 | 1.71.1 |
| Parser | `granit-parser` 2.0.0, its author's fork of `saphyr-parser`, with `#![forbid(unsafe_code)]` | `unsafe-libyaml-norway` 0.2.15: libyaml transpiled to unsafe Rust |
| Dependencies with default features, serde excluded | About 20; about 64 with `miette` | About 9 |
| Downloads, all-time (last 90 days) | 15.0M (11.4M) | 18.3M (9.7M) |
| Repository | [bourumir-wyngs/serde-saphyr](https://github.com/bourumir-wyngs/serde-saphyr): last commit 2026-11-01; 1 open issue, 3 open PRs; 98 issues closed in 12 months | [cafkafk/serde-norway](https://github.com/cafkafk/serde-norway): last commit 2024-12-21; 8 open issues, 4 open PRs; none closed in 12 months |
| RustSec advisories | None | None |

The dependency counts are estimates from crates.io's dependency data, because `cargo tree` couldn't run.

Not candidates:

- `serde_yaml` is deprecated by its author, and Appendix C already excludes it.
- `serde_yaml_ng` 0.10.0 is MIT only and built on `unsafe-libyaml`. RustSec lists that as unmaintained and suggests `saphyr` or `serde-saphyr` instead ([RUSTSEC-2024-0449](https://rustsec.org/advisories/RUSTSEC-2024-0449.html)).
- `serde_yml` is unsound and archived ([RUSTSEC-2025-0068](https://rustsec.org/advisories/RUSTSEC-2025-0068.html); [RUSTSEC-2025-0067](https://rustsec.org/advisories/RUSTSEC-2025-0067.html) for its `libyml`).
- `saphyr` 0.3.3 is a YAML 1.2 node tree with source markers, but no serde. It matters only as a second parser for spans next to `serde_norway`.

**Left for the hands-on run:** `cargo tree -e normal`, `cargo deny check`, and the clean build time and release binary size of a small loader built with each crate.

### Source spans for diagnostics

§5.5 gives each diagnostic "a code, a severity and a file span (via miette)", and V003–V006 fire on YAML that parsed fine: `ports`, cross-layer `depends_on`, `container_name` and `${VAR}`. So p4n4 needs the spans of valid values, not only of errors.

**serde-saphyr** has them:

- `Spanned<T>` gives a value's `referenced` and `defined` locations (`spanned.rs` L162–166). Through an alias, they are the alias and the anchor (L58–70).
- A `Location` is a 1-based line and column, in characters (`location.rs` L8 and L45–53). Its `Span` has the `byte_offset()` and `byte_len()` that miette needs (`span.rs` L277–304), but only for `&str` input.
- Errors have `location()` (`de/error.rs` L1749) and `render()`, which prints a snippet (L1471). The `miette` feature adds `to_miette_report` (`de/miette.rs` L47) and about 44 crates; converting a `Span` by hand takes a few lines.
- **The catch:** under `flatten`, and in untagged or internally tagged enums, `Spanned` reports 0:0 (`spanned.rs` L72–80). Compose's short and long forms, such as `depends_on`, need hand-written `Deserialize` impls or a `Spanned` around the whole enum (L82).

**serde_norway** has locations only in errors: a byte `index()`, `line()` and `column()` (`error.rs` L50–74 and L102). There's no `Spanned`, and its event layer is private (`lib.rs` L168). So V003–V006 would need a second parser, such as `saphyr`, and a map from paths to spans, which breaks with aliases and merges.

**Left for the hands-on run:**

- `Spanned` on `container_name`, a `ports` entry, `depends_on.<service>.required` and an `environment` value in `stacks/iot/docker-compose.yml`: the line, column and byte slice must be right.
- `logging` through `*logging`: `referenced` at the alias, `defined` at L12.
- The same file with CRLF, with a UTF-8 BOM, and with a non-ASCII comment above a value: the byte slices must still be right, and one miette report must render.
- `Spanned` in an untagged enum, under `flatten`, and around a whole enum.
- `serde_norway`'s error locations for a type error, a duplicate key and a syntax error.

### Anchors and merge keys

The inputs have one anchor and no merges. But Compose users write `<<: *defaults`, and the `_template` extra invites hand-written layer files.

**serde-saphyr** replays aliases into any target, and applies merge keys into structs as well as maps. The key can be `<<` or `!!merge`, and the value a mapping or a list of them (`de/key_nodes.rs` L555–578 and L689–768; on by default, `de/options.rs` L620). Its default budget (`de/budget.rs` L275–301: 1M events, 50,000 aliases, depth 64, 250,000 nodes, 10,000 merge keys) and alias limits (1M replayed events; `de/options.rs` L137–156) stop a "billion laughs" file with an error.

**serde_norway** caps alias replay at 100 times the document's events (`de.rs` L477–479), and depth at 128 (L112). It merges only in `Value::apply_merge` (`value/mod.rs` L626–667), before `from_value`, which loses the locations. Straight into a struct, `<<` is an unknown key, and the merged fields are lost.

**Left for the hands-on run:**

- The 8 `logging` values in `stacks/iot/docker-compose.yml` must equal the anchor's.
- Against `docker compose config --format json`: `<<: *base`; `<<: [*a, *b]` with a shared key; an explicit key over a merged one; a merge inside `environment`; and a merge of a merge. With `serde_norway`, both through `apply_merge` and straight into structs.
- A billion-laughs file (9 levels of 9 aliases) and a 10,000-deep nesting must fail fast with a located error. Record the time and peak memory.

### Typed deserialization of layer Compose files

From serde-saphyr's `parse_scalars.rs` (L27–44, L172–269) and `de/deserializer.rs` (L880–926, L964–984), and serde_norway's `de.rs` (L1062–1130, L1264). Bold marks a result that differs from go-yaml's.

| Plain scalar | YAML 1.1 (PyYAML) | YAML 1.2 core | go-yaml (Compose) | `serde-saphyr`, defaults | `serde-saphyr`, Compose options¹ | `serde_norway` |
|---|---|---|---|---|---|---|
| `off`, untyped | false | "off" | "off" | **false** | "off" | "off" |
| `off` into a bool | false | error | false² | false | **error** | **error** |
| `0644`, untyped | 420 | 644 | 420 | **644.0** | 420 | **"0644"** |
| `0644` into an integer | 420 | 644 | 420 | **error** | 420 | **error** |
| `08`, untyped | "08" | 8 | 8.0 | 8.0 | 8.0 | **"08"** |
| `1_000`, untyped | 1000 | "1_000" | 1000 | 1000 | 1000 | **"1_000"** |
| `1e3`, untyped | "1e3" | 1000.0 | 1000.0 | 1000.0 | 1000.0 | 1000.0 |
| `22:22`, untyped | 1342 | "22:22" | "22:22" | "22:22" | "22:22" | "22:22" |
| empty, `~`, `null` | null | null | null | null | null | null |

¹ `strict_booleans: true` and `legacy_octal_numbers: true`. Both default to `false` (`de/options.rs` L622–623).  
² Into a Go `bool`. Compose decodes untyped and gets "off"; `toBoolean` then makes it false, with a warning, for the fields on its list. Other boolean fields probably reject it (to confirm).

- **serde-saphyr's defaults aren't Compose's.** Untyped, it reads `off` and `yes` as booleans, and `0644` as 644.0. That matters because `serde_json::Value`, untagged enums and `flatten` all read untyped: an `environment` value typed as an untagged enum of string, number and bool would turn `FOO: off` into `false`. The two options fix every row except typed booleans: with `strict_booleans`, `read_only: off` fails where Compose only warns. A small `ComposeBool` that also takes compose-go's words, with a warning, matches Compose.
- **Strings.** Both crates give a `String` the scalar as written (serde-saphyr: `de/deserializer.rs` L1124 and L1214; serde_norway: `de.rs` L1470–1491). Compose resolves first and stringifies after ([`mapping.go` L115–126](https://github.com/compose-spec/compose-go/blob/v2.16.1/types/mapping.go#L115-L126)), so `environment: {FOO: 0644}` is "0644" to p4n4 but "420" in the container.
- **Nulls.** serde-saphyr refuses an empty value, `~` or `null` for a `String` (`de/deserializer.rs` L1208–1225); use `Option<String>`. Both crates read an empty value as an empty map, struct or list, so the 64 named volumes load (serde-saphyr: L1511–1525 and L1722–1747, also for `~` and `null`; serde_norway: `de.rs` L1610–1698).
- **Tags.** serde-saphyr ignores unknown tags such as `!reset` on sequences and structs, captures them with `Tagged<T>` (`wrappers.rs` L277), or rejects them by option (`de/options.rs` L318–333). serde_norway ignores them on sequences (`de.rs` L1610–1636). Both break untagged enums, because a tagged node becomes a one-entry map (`de/deserializer.rs` L796–808) or an enum (`de.rs` L1206–1257).
- **Duplicate keys.** serde-saphyr rejects them by default (`de/options.rs` L619), as compose-go does. serde_norway rejects them in its `Mapping` (`mapping.rs` L817) and in struct fields, but probably not in an `IndexMap`.
- **Interpolation.** serde-saphyr's optional `properties` feature (no extra dependencies; `Cargo.toml` L83) interpolates while deserializing, and its `PropertySyntax::DockerCompose` mode claims Compose's rules (`de/options.rs` L87–89; README, "Properties"). So `retries: ${N}` could load into an integer, with `${N}`, not its value, in errors. Unset variables become "" silently, so V006 still scans the raw strings. With serde_norway, interpolate the strings first, then parse.

**Left for the hands-on run:**

- The 21 Compose files and S1's sample layers (`spikes/s1-compose/project/{core,iot,ai}/compose.yaml`) into the partial Compose model, with both crates, checked against the Compose oracle in the test plan.
- Each row of the table, with each crate and both serde-saphyr option sets, into `serde_json::Value`, `bool`, `i64`, `f64`, `String` and an untagged enum.
- `compose.smoke.yml`'s `!reset` into a `Vec`, an untagged enum and `Tagged<T>`, and `!override` on a mapping.
- A duplicate service name, and a duplicate key inside a service, into an `IndexMap` and into a struct.
- Empty, `~` and `null` into `Option<T>`, a struct with defaults, and a map.
- `FOO: 0644` in `environment`, `read_only: off`, `read_only: ${RO:-false}` and `retries: ${N}`, with each crate and with the latest Compose.
- With `properties` in `DockerCompose` mode and the values in `stacks/iot/.env.example`: `stacks/iot/docker-compose.yml` must equal `docker compose config`, `Spanned` values must point at their `${…}`, and an unset variable must give "", as in Compose.
- `warmup.yaml`, whose two `off` keys must stay "off", and the 10 `>-` scalars, which must equal PyYAML's values.

### Serialization of `compose.yaml` and the emulator overlay

p4n4 writes `compose.yaml` and `.p4n4/ports/<layer>.yaml` (§5.3) and `.p4n4/emu/compose.emu.yaml` (§6.6), and §10 snapshots them. v0.4.2's `render_overlay` (`tools/emu/p4n4_emu/overlays/generator.py` L225–295) fills a Jinja2 template that quotes values by hand (`cpus: "{{ svc.cpus }}"`, `stack.emu.yml.j2` L37). From typed structs, the serializer must quote. No layer file is rewritten, so comments needn't survive.

**serde-saphyr** (`ser/options.rs` L70–168; defaults at L194–213):

- **Quoting.** By default (`yaml_12: false`), it quotes strings that a YAML 1.1 or 1.2 reader would take for something else: yes/no/on/off/y/n, numbers, sexagesimals, timestamps, nulls, `<<` and `=` (`ser/quoting.rs` L197–214). So `"no"`, `"0.5"`, `"524288000"` and `"22:22"` stay quoted. `127.0.0.1:18101:1883`, `3352m` and `/dev/sda` come out plain (L284–328), which go-yaml reads as strings. `DoubleQuoted<T>` (`wrappers.rs` L36) forces quotes, for example to match S1's sample ports file.
- **Layout.** The indent is 2 spaces. `compact_list_indent` (L94), on by default, puts `- ` level with its key; §5.3 indents lists, so turn it off. `FlowSeq<T>` (`wrappers.rs` L7) writes §5.3's `path: [...]`, and `Commented<T>` (L196) its trailing comment, with the default `comments` feature (L195).
- **Block scalars.** `prefer_block_scalars` (L130), on by default, folds strings over 80 characters. Turn it off.
- **Order and anchors.** Keys come out in struct and map order. Anchors appear only for `RcAnchor` and `ArcAnchor` values (`anchors.rs` L82 and L107).

**serde_norway** has no options. `serialize_str` (`ser.rs` L308–364) quotes only what its YAML 1.2 rules would mistype, plus digit strings such as `0644`. So `no`, `yes`, `on`, `off` and `22:22` come out plain ([issue #30](https://github.com/cafkafk/serde-norway/issues/30); the fix, [PR #31](https://github.com/cafkafk/serde-norway/pull/31), has been open since 2025-04). go-yaml reads them back as strings, but PyYAML doesn't. The layout is libyaml's: lists level with their key, no flow sequences, no comments, unlimited width (`libyaml/emitter.rs` L72–73). CRLF inside strings comes out escaped ([issue #28](https://github.com/cafkafk/serde-norway/issues/28)). It never writes anchors.

**Left for the hands-on run:**

- §5.3's `compose.yaml`, serialized from structs and diffed with the plan and with `spikes/s1-compose/project/compose.yaml`. The same for a ports file and for S1's `.p4n4/emu/compose.emu.yaml`.
- The jetson overlay, serialized from structs: `docker compose config` must give the same result for it as for v0.4.2's.
- Every hazard string, written, then read back unchanged by the same crate, by go-yaml (through `docker compose config`) and by PyYAML: `no`, `yes`, `on`, `off`, `y`, `n`, `true`, `~`, `null`, the empty string, `0644`, `08`, `1_000`, `1e3`, `.inf`, `0x1F`, `22:22`, `2001-12-14`, `1.04`, `524288000`, `<<`, `=`, `- a`, `#x`, `a: b`, ` lead`, `trail `, a 100-character line, two lines, CRLF and non-ASCII text.
- Serializing twice must give the same bytes, with LF line endings on Windows too.

### Which crate (Q1): requirements against crates

| Requirement | `serde-saphyr` 2.0.0 | `serde_norway` 0.9.42 |
|---|---|---|
| Locations in errors | **Yes:** line, column and a snippet; optional miette | **Yes:** byte index, line and column |
| Spans of valid values (V003–V006) | **Yes:** `Spanned<T>`, with byte ranges for `&str` input; 0:0 under `flatten` and untagged enums | **No:** they need a second parser |
| Aliases | **Yes,** with the alias's and the anchor's locations | **Yes** |
| Merge keys into typed structs | **Yes,** on by default | **Partial:** only through `Value::apply_merge`, losing the locations |
| Limits against alias bombs | **Yes:** a budget and alias limits | **Yes:** replay capped at 100 times the events |
| Rejecting duplicate keys | **Yes,** by default | **Partial:** only in `Mapping` and struct fields |
| Resolving scalars like Compose's go-yaml | **Yes** with two options; **No** with the defaults | **Partial:** YAML 1.2, and `0644` stays a string |
| Compose's tags (`!reset`, `!override`) | **Yes:** ignored, captured with `Tagged<T>`, or rejected | **Partial:** ignored on sequences, but they break `deserialize_any` |
| `${VAR}` in typed fields | **Yes:** the optional `properties` feature, in Compose mode | **No:** interpolate the strings first |
| Loading the typed Compose model | Not tested | Not tested |
| Quoting ambiguous strings on output | **Yes:** YAML 1.1 and 1.2 ambiguities | **Partial:** `no`, `yes`, `on` and `off` come out plain |
| Layout for §5.3 | **Yes:** flow lists, comments, indent options, forced quotes | **Partial:** a fixed style, with no flow lists or comments |
| Anchors on output (not needed) | Yes, opt-in | No |
| Maintained, safe Rust, no advisories | **Yes** | **Partial:** no advisory, but no commit since 2024-12, and an unsafe libyaml port |

**Left for the hands-on run:** the "Not tested" row, and confirming each Yes and Partial with the cases above.

### Hands-on test plan

The cases are in the "Left for the hands-on run" lists above. They share this setup:

- **Crates:** `serde-saphyr = "2.0.0"` with its default features (`serialize`, `deserialize` and `comments`) plus `properties`, `serde_norway = "0.9.42"`, `serde_json`, `indexmap` with `serde`, and `miette`. Build serde-saphyr a second time with its `miette` feature, for the size and time comparison. Note any newer versions.
- **Models,** the same for both crates: a partial Compose model with the fields that §5.5's rules and the overlay need, ignoring unknown fields, and the metadata, profile and scenario models that `xtask convert` needs.
- **Inputs:** the 41 input files in the per-file table, plus the overlay, rendered as shown below.
- **Oracles:** `docker compose config --format json` on the minimum (from S1) and the latest Compose for Compose files, and PyYAML for the rest, with the two `off` keys as the known difference.
- **Measurements:** `cargo tree -e normal`, `cargo deny check`, the clean build time and the release binary size.

**What passing means:** every file loads and matches its oracle; spans are right with LF, CRLF and a BOM; merges match Compose's; the alias bomb fails fast; the output passes `docker compose config`, and go-yaml and PyYAML read every hazard string back unchanged; and `cargo deny` is clean.

<details>
<summary>Rendering the overlay with v0.4.2</summary>

Run this in WSL with the golden venv, `~/.venvs/p4n4-golden-v0.4.2`, which has `tools/emu` installed. Set `PYTHONDONTWRITEBYTECODE=1`, and pass two arguments: the repository root and the output file.

```python
import sys
from p4n4_emu.profiles.loader import load_profile_file
from p4n4_emu.overlays.generator import render_overlay, service_shares, budget_scale

root = sys.argv[1]
prof = load_profile_file(root + "/tools/emu/p4n4_emu/profiles/definitions/jetson-orin-nano.yml")
svcs = ["ollama", "n8n", "ei-runner", "system1", "agent"]
scale = budget_scale([service_shares("ai", svcs, None, prof)])
out = render_overlay(prof, "ai", "/dev/sda", services=svcs, scale=scale, gpu="nvidia",
                     cpu_factor=0.5, cpuset="0-5", cpuset_auto=True)
open(sys.argv[2], "w", encoding="utf-8", newline="\n").write(out)
```

</details>

**Left for the hands-on run:** running this plan, and recording each result in the subsections above.

## Decision

Not decided yet.

**Leaning:** `serde-saphyr` 2.x with its default features, without `miette`, behind a small `yaml` module in `p4n4-core` that sets the options in one place. Its `properties` feature is worth adding if the hands-on run shows that it interpolates as Compose does. Reasons:

- It is the only candidate with spans on valid values, which V003–V006 need, and with merge keys in typed structs.
- With two options, it resolves plain scalars as Compose does, and it offers Compose-style interpolation. `serde_norway` can't read `0644` as Compose does.
- It quotes whatever a YAML 1.1 or 1.2 reader would misread, and has the layout controls that §5.3 uses.
- It is maintained safe Rust with no advisories. `serde_norway` hasn't had a commit in 23 months, and its parser descends from `unsafe-libyaml`, which RustSec lists as unmaintained.

Risks:

- **Churn.** 2.0 came ten weeks after 1.0. Pin the version and update deliberately.
- **Defaults unlike Compose's:** YAML 1.1 booleans, `0644` as a float, folded strings and compact lists. The wrapper sets them once, and tests pin them.
- **Spans** vanish under `flatten` and untagged enums, which shapes the Compose model.
- **Dependencies:** about twice as many as `serde_norway`'s, to be measured.

The fallback is `serde_norway` with `Value::apply_merge`, plus `saphyr` for spans. It would work, but every file would be parsed twice, by two YAML stacks.

The leaning becomes the decision if serde-saphyr passes the hands-on run.

## Follow-ups

These are proposed plan changes; this spike doesn't edit the plan.

- **§13 Q1 and Appendix C:** after the hands-on run, answer Q1, and put the crate, its features and its options in the YAML row. Add `serde_yaml_ng` and `serde_yml` to "Not used", with their advisories.
- **§5.5:** read layer files with `strict_booleans` and `legacy_octal_numbers` on, duplicate keys as errors, and merges on. Let booleans take compose-go's YAML 1.1 words with a warning, and avoid `flatten` and untagged enums where a rule needs a span. Choose how to interpolate: keep fields as strings until p4n4 interpolates them, or use serde-saphyr's `properties`. V006 scans the raw `${…}` either way. Consider a warning rule for plain scalars that YAML 1.1 and 1.2 read differently (`off`, `yes`, `0644`, `22:22`).
- **§4.1, §6.6 and §6.10:** `xtask convert` should report the YAML 1.1/1.2 differences it finds. The first is `warmup.yaml`'s `off` keys, which v0.4.2 ignores and Rust will apply. That is harmless at today's values, and quoting the keys in v0.4.x would make both agree. TOML has no null, so the scenario schema needs a spelling for `raw:`, which must be null today (`sim-scenario.yml` L22; `tools/emu/p4n4_emu/sim/scenario.py` L440–444). `raw = {}` is one option.
- **§5.3 and §6.6:** record the serializer settings (`prefer_block_scalars` and `compact_list_indent` off, the default quoting, the header prepended as text), and whether port mappings are quoted. S1's sample quotes them, and serde-saphyr needs `DoubleQuoted` to match.
- **§9.1 (S1) and §10:** Compose v5.6.1 reads YAML with go-yaml v4, still a release candidate, whose rules match v3's on a desk reading. compose-go's list of booleans that take YAML 1.1 words also grows between versions: `depends_on.*.required` joined it after v2.8.1. Run S7's hazard file on both the minimum and the latest Compose.
