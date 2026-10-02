# Emulator

`p4n4-emu` (`tools/emu/`) is a workstation hardware emulator. It applies Docker resource constraints (CPU, RAM, disk I/O) and optional QEMU ARM64 emulation as a Compose overlay on top of the existing p4n4 stacks — making a developer's workstation behave like target edge hardware without modifying any production files.

## How it works

`p4n4-emu` generates a `*.emu.yml` overlay per stack and passes it after the stack's own compose files (`compose.yaml` / `docker-compose.yml`, any `docker-compose.override.yml`, or `COMPOSE_FILE`):

```
docker compose -f docker-compose.yml [-f docker-compose.override.yml] -f <stack>.emu.yml up -d
```

The overlay covers the services the stack's compose config actually defines and injects `deploy.resources.limits` (CPU, memory), `memswap_limit` and optional `blkio_config` (disk I/O) per service. Production files are never modified.

All enabled stacks of a project share one emulated device: when their combined CPU or memory shares exceed the profile, every service is scaled down so the total fits.

## Hardware profiles

| Profile | CPU | Memory | Disk R/W | Arch |
|---------|-----|--------|----------|------|
| `rpi4` | 4 cores | 3.5 GB | 50 MB/s | arm64 |
| `rpi5` | 4 cores | 7 GB | 100 MB/s | arm64 |
| `mcu-class` | 1 core | 256 MB | 10 MB/s | x86_64 |
| `nuc` | 4 cores | 14 GB | 200 MB/s | x86_64 |

## Requirements

- Docker Engine >= 24
- Docker Compose >= 2.17
- Python >= 3.11
- cgroup v2 — check with `cat /sys/fs/cgroup/cgroup.controllers`
- QEMU binfmt_misc — only to emulate another architecture, e.g. ARM profiles on an x86 host

## Installation

```bash
cd tools/emu
uv sync          # or: pip install -e .
```

Verify:

```bash
uv run p4n4-emu --help
```

## Usage

### Preflight check

```bash
uv run p4n4-emu setup --check-only
```

All items should show **OK**. A `cgroup v2` warning means CPU/memory limits won't be enforced.

### Enable ARM64 emulation (optional)

Required to run the ARM profiles (`rpi4`/`rpi5`) on an x86 host. `up` runs their arm64 images by default and refuses to start until QEMU is registered; pass `--native` to apply only the resource limits with host-architecture images:

```bash
uv run p4n4-emu setup --arch arm64
```

### Start a stack

```bash
# Inside a p4n4 project: all enabled stacks
uv run p4n4-emu up --profile rpi5
uv run p4n4-emu up --profile rpi5 --stack ai     # one stack only

# Outside a project: point at a compose directory (or a parent with per-stack subdirectories)
uv run p4n4-emu up --stack-dir ~/p4n4/stacks/iot --profile rpi5

# All stacks + synthetic sensor data from 3 devices
uv run p4n4-emu up --stack all --stack-dir ~/p4n4/stacks --profile rpi5 --sim --sim-devices 3
```

Use `--dry-run` to inspect the generated overlay before any containers start:

```bash
uv run p4n4-emu up --stack-dir ~/p4n4/stacks/iot --profile rpi5 --dry-run
```

### Status and stop

```bash
uv run p4n4-emu status --profile rpi5

uv run p4n4-emu down --profile rpi5            # stop containers
uv run p4n4-emu down --profile rpi5 --volumes  # stop and remove data volumes
```

## Command reference

```
p4n4-emu setup [--arch arm64|armv7] [--check-only]
p4n4-emu up    [--profile PROFILE] [--stack iot|ai|edge|iot,ai|all]
               [--stack-dir PATH] [--arch arm64|armv7|x86_64 | --native]
               [--sim] [--sim-interval 2.0] [--sim-devices 1] [--dry-run]
p4n4-emu down  [--profile PROFILE] [--stack iot|ai|edge|iot,ai|all]
               [--stack-dir PATH] [--volumes]
p4n4-emu status [--profile PROFILE] [--stack iot|ai|edge|iot,ai|all]
p4n4-emu profile list
p4n4-emu profile show <name>
p4n4-emu sim start [--interval 2.0] [--devices 1] [--mqtt-host p4n4-mqtt]
p4n4-emu sim stop
p4n4-emu sim status
```

Without `--stack`, `up`/`down`/`status` target the enabled stacks of the surrounding p4n4 project (`.p4n4.json` is found by walking up from the current directory), falling back to `iot`. Stack directories resolve in this order: `--stack-dir` (its `<stack>/` subdirectory first), then the project layout (flat root or `<project>/<stack>/`), then a `<stack>/` or compose file next to the current directory.

`--arch` overrides the profile's architecture; without it, ARM profiles emulate arm64 and x86 profiles run natively. `--native` never forces a platform.

## Sensor simulator

The built-in simulator (`--sim`) publishes synthetic MQTT payloads on the topics the p4n4 stack consumes, `sensors/<device-id>/<measurement>`:

```
sensors/emu-sensor-0/temperature   {"value": 23.4, "unit": "C"}
sensors/emu-sensor-0/humidity      {"value": 58.2, "unit": "%"}
sensors/emu-sensor-0/pressure      {"value": 1012.7, "unit": "hPa"}
sensors/emu-sensor-0/raw           {"values": [0.01, -0.02, 1.00], "cpu_pct": 42.3}
```

Run standalone against a local Mosquitto:

```bash
MQTT_HOST=localhost uv run python -m p4n4_emu.sim.sensor_sim
```

## GPIO stub

`p4n4_emu.hw.gpio_stub` is a drop-in `RPi.GPIO` replacement for running [RPi5 hardware scripts](hardware.md) on a workstation:

```python
import sys
import p4n4_emu.hw.gpio_stub as GPIO
sys.modules["RPi"] = type(sys)("RPi")
sys.modules["RPi.GPIO"] = GPIO
```

Set `logging.basicConfig(level=logging.DEBUG)` to see pin state transitions in your terminal.

## Known limitations

- **cgroup v2 required** — CPU/memory limits are not enforced on cgroup v1 hosts.
- **blkio_config** is skipped when Docker's data root is on a non-block device (tmpfs, NFS).
- **QEMU overhead** ~3–10x on ARM64; Ollama LLM inference under QEMU is impractical.
- **Network I/O throttling** is not supported (requires host-level `tc netem`).
