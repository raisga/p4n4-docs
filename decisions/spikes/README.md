# Rust rewrite: P0 spikes

**Date:** 2026-11  
**Status:** In progress

The P0 spikes of the [Rust rewrite plan](https://github.com/raisga/p4n4/blob/main/RUST_REWRITE_PLAN.md#91-spikes-p0) (§9.1). Each spike answers its questions with a decision, in a note here, and the ADRs written in P1 ([plan §15](https://github.com/raisga/p4n4/blob/main/RUST_REWRITE_PLAN.md#15-adr-changes)) link to the notes. The spikes' code is in the superproject's `spikes/<id>/` until P1 deletes it.

| Spike | Decides | Needs | Status |
|---|---|---|---|
| [S1: Compose](s1-compose.md) | The minimum Compose version, and the recorded fixtures for `FakeDriver` | Docker | Partial: desk research and static checks point to 2.21.0; the engine checks are left |
| [S2: Proxy and Grafana](s2-proxy.md) | The proxy implementation, the settings each upstream needs, and Grafana role mapping | Docker | Partial: desk research confirms Grafana's role header and leans to `axum-reverse-proxy`; the hands-on checks are left |
| S3: Inference packaging | Backends per target, link mode and base images | x86_64, the Pi 5 and the armv7 board | Not started |
| [S4: System 1 parity](s4-system1.md) | Go or no-go for Rust System 1 in 1.0 | x86_64 and the Pi 5 | Partial: Q13 answered (yes) by desk research |
| S5: ELM327 | Driver structure, and the transcript fixtures | The car and its ELM327 adapter | Not started |
| S6: GPIO and shutdown | The GPIO crate, and the shutdown path | The Pi 5 and the armv7 board | Not started |
| [S7: YAML](s7-yaml.md) | The YAML crate (plan Q1) | Any machine | Partial: desk research leans to `serde-saphyr` 2.x; the hands-on checks are left |
| S8: Cross builds and Windows | The target list, the glibc floor and the CI matrix | Linux with zig, and Windows | Not started |
| S9: Docker Desktop | Supported server modes for each platform | Docker Desktop on Windows and macOS | Not started |

A spike's row links to its note once the note exists.

## Writing a note

One file per spike, named like its code directory: `s1-compose.md`, `s7-yaml.md`, and so on.

```markdown
# Spike S7: YAML

**Date:** 2026-11  
**Status:** Done  
**Plan:** [§9.1](https://github.com/raisga/p4n4/blob/main/RUST_REWRITE_PLAN.md#91-spikes-p0)  
**Code:** `spikes/s7-yaml/` in the superproject, until P1

## Questions

The spike's questions, copied from the plan.

## Setup

Machines, operating systems, and the versions of Docker, Compose, Rust and the crates involved.

## Results

One subsection per question: what was tried and what happened, with commands and output
trimmed to what matters.

## Decision

The answer, and what it decides in the plan.

## Follow-ups

Changes to the plan, open questions, and work for P1.
```
