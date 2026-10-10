# Emulator

`p4n4-emu` (`tools/emu/`) is a workstation hardware emulator. It applies a board's CPU,
memory and disk limits, and optionally its architecture through QEMU, as a Compose overlay
on top of the p4n4 stacks, so a developer's workstation behaves like the target edge
hardware without changing any project file.

## How it works

`p4n4-emu` generates a `*.emu.yml` overlay per stack and passes it after the stack's own
compose files (`compose.yaml` / `docker-compose.yml`, any `docker-compose.override.yml`,
or `COMPOSE_FILE`):

```
docker compose -f docker-compose.yml [-f docker-compose.override.yml] -f <stack>.emu.yml up -d
```

The overlay covers the services the stack's compose config actually defines and sets
`deploy.resources.limits` (CPU, memory), `memswap_limit` and, when Docker's data root is on
a block device, `blkio_config` (disk bandwidth) for each.

All enabled stacks of a project share one emulated device: when their combined CPU or
memory shares exceed the profile, every service is scaled down so the total fits.

Overlays live in `~/.p4n4-emu/overlays/<stack-dir>-<hash>/<stack>.emu.yml`, one folder per
stack directory, so two projects never share one. Each overlay records its profile in an
`x-p4n4-emu` block (Compose ignores `x-` keys), so `down`, `status`, `logs` and
`profile switch` find it on their own. `down` deletes the overlay.

The shared network (`p4n4-net`) and the MQTT broker (`p4n4-mqtt`) are read from the
stacks' compose config, so a project that renames them still works. `up` creates the
networks a stack names before Compose starts it, labelled as Compose labels its own.
An existing unlabelled network is recreated only while no container uses it.

## Hardware profiles

| Profile | CPU | Memory | Disk R/W | Arch |
|---------|-----|--------|----------|------|
| `rpi4` | 4 cores | 3.5 GB | 50 MB/s | arm64 |
| `rpi5` | 4 cores | 7 GB | 100 MB/s | arm64 |
| `mcu-class` | 1 core | 256 MB | 10 MB/s | x86_64 |
| `nuc` | 4 cores | 14 GB | 200 MB/s | x86_64 |

What each limit does and doesn't reproduce, and where the numbers come from, is under
[How faithful the emulation is](#how-faithful-the-emulation-is).

## Requirements

- Docker Engine >= 24 and Docker Compose >= 2.17
- Python >= 3.11
- cgroup v2 for disk limits on buffered writes (`docker info -f '{{.CgroupVersion}}'`);
  CPU and memory limits work on v1 too
- QEMU binfmt_misc, only to emulate another architecture (ARM profiles on an x86 host).
  Docker Desktop ships it.

## Installation

```bash
cd tools/emu
uv tool install --editable .     # puts p4n4-emu on your PATH; or: pip install -e .
```

Once it's on PyPI: `uv tool install p4n4-emu`. The sensor simulator runs from
`ghcr.io/raisga/p4n4-sensor-sim:<version>` (amd64 and arm64), pulled the first time it
starts, or built from the installed package when that fails.

## Usage

### From the p4n4 CLI

```bash
p4n4 up --emu rpi5        # the project's stacks under the rpi5 profile
p4n4 down                 # stacks started that way stop through p4n4-emu down
```

`p4n4 up --emu` runs `p4n4-emu up --profile <profile>` for the stacks, so `p4n4-emu` must
be on `PATH`. `p4n4 down` recognises stacks whose containers were created with an emu
overlay and hands them to `p4n4-emu down`, which also removes the overlay and the
simulator.

### Preflight check

```bash
p4n4-emu setup --check-only
```

All items should show **OK**. The check reads Docker's cgroup version and driver from
`docker info`, so it's right on Docker Desktop too, where the engine runs in a VM.

### Enable ARM64 emulation (optional)

Required to run the ARM profiles (`rpi4`/`rpi5`) on an x86 Linux host. `up` runs their
arm64 images by default and refuses to start until QEMU is registered; pass `--native` to
apply only the resource limits with host-architecture images:

```bash
p4n4-emu setup --arch arm64
```

It runs `tonistiigi/binfmt --install arm64` once per host, with the image pinned to
`qemu-v10.2.3` by digest because it runs privileged. Docker Desktop needs no setup.

### Start a stack

```bash
# Inside a p4n4 project: all enabled stacks
p4n4-emu up --profile rpi5
p4n4-emu up --profile rpi5 --stack ai     # one stack only

# Outside a project: point at a compose directory (or a parent with per-stack subdirectories)
p4n4-emu up --stack-dir ~/p4n4/stacks/iot --profile rpi5

# Synthetic sensor data from 3 devices
p4n4-emu up --profile rpi5 --sim --sim-devices 3
```

`--dry-run` prints the overlay without starting anything. When a later stack fails to
start, the stacks this run started are stopped again; stacks already running stay up.

### Status, switching profile, stopping

```bash
p4n4-emu status                    # the profile up used; usage against limits
p4n4-emu status --json             # the same, for scripts and CI
p4n4-emu profile switch rpi4       # new CPU / memory limits, without restarting
p4n4-emu down                      # stop containers
p4n4-emu down --volumes            # stop and remove data volumes
```

For each service, `status` shows CPU and memory in use against the container's limits
(highlighted at 90%), and whether the container actually has the limits its overlay asks
for. **stale** means it doesn't: it was created before the overlay changed, or without
it; `p4n4-emu up` recreates it.

`profile switch` applies the new profile's CPU and memory limits to running containers
with `docker update`. Disk limits can't change that way, so `status` shows them stale until
the next `up`, and a switch that changes the architecture is refused.

## Command reference

```
p4n4-emu setup [--arch arm64|armv7] [--check-only]
p4n4-emu up    [--profile PROFILE] [--stack iot|ai|edge|dashboard|iot,ai|all]
               [--stack-dir PATH] [--arch arm64|armv7|x86_64 | --native]
               [--sim] [--sim-interval 2.0] [--sim-devices 1] [--build] [--pull]
               [--dry-run]
p4n4-emu down  [--stack ...] [--stack-dir PATH] [--volumes] [--yes]
p4n4-emu status [--profile PROFILE] [--stack ...] [--json]
p4n4-emu logs  [SERVICE] [--stack ...] [--stack-dir PATH] [--tail 100] [--no-follow]
p4n4-emu profile list [--json]
p4n4-emu profile show <name> [--json]
p4n4-emu profile switch <name> [--stack ...] [--dry-run]
p4n4-emu sim start [--interval 2.0] [--devices 1] [--mqtt-host HOST] [--network NAME]
                   [--rebuild]
p4n4-emu sim stop
p4n4-emu sim status
```

Without `--stack`, commands target the enabled stacks of the surrounding p4n4 project
(`.p4n4.json` is found by walking up from the current directory), falling back to `iot`.
Stack directories resolve in this order: `--stack-dir` (its `<stack>/` subdirectory
first), then the project layout (flat root or `<project>/<stack>/`), then a `<stack>/` or
compose file next to the current directory.

`--arch` overrides the profile's architecture; without it, ARM profiles emulate arm64 and
x86 profiles run natively. `--native` never forces a platform.

## Sensor simulator

The simulator (`--sim`, or `p4n4-emu sim start`) publishes synthetic MQTT payloads on the
topics the p4n4 stack consumes, `sensors/<device-id>/<measurement>`:

```
sensors/emu-sensor-0/temperature   {"value": 23.4, "unit": "C"}
sensors/emu-sensor-0/humidity      {"value": 58.2, "unit": "%"}
sensors/emu-sensor-0/pressure      {"value": 1012.7, "unit": "hPa"}
sensors/emu-sensor-0/raw           {"values": [0.01, -0.02, 1.00], "cpu_pct": 42.3}
```

Each device follows the same waveforms, shifted by a phase derived from its id, so
`emu-sensor-0` and `emu-sensor-1` report different values at the same moment. If the
broker isn't up yet, or restarts, the simulator keeps retrying (1 s, doubling up to 30 s)
and drops the readings it takes while disconnected. It joins the iot broker's network and
connects to it by the name the iot stack gives it; `--mqtt-host` and `--network` override
both.

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

It covers the `RPi.GPIO` calls the p4n4 scripts make: `setmode`, `setup` (with
`pull_up_down=` and `initial=`, one channel or a list), `output`, `input`, the edge
functions (`add_event_detect`, `add_event_callback`, `remove_event_detect`,
`event_detected`, with `bouncetime`) and `cleanup`. `PWM` and `wait_for_edge` aren't
stubbed yet.

Nothing outside the script drives an input pin, so press a button with `set_input`, a
stub-only call. A level change is an edge and fires the pin's callbacks:

```python
GPIO.set_input(27, GPIO.LOW)    # press the button on GPIO 27 (pulled up)
GPIO.set_input(27, GPIO.HIGH)   # release it
```

Set `logging.basicConfig(level=logging.DEBUG)` to see pin state transitions in your terminal.

## How faithful the emulation is

p4n4-emu reproduces a board's *resource ceilings* and *instruction set*, not its speed. Use
it to find what doesn't fit (a service killed for memory, a stack that can't start, a
dashboard that stalls on disk), not to predict latency or throughput on the real device.

| What | How it's applied | Enforced | Approximated, or not at all |
|------|------------------|----------|-----------------------------|
| CPU | `deploy.resources.limits.cpus` → CFS quota (`cpu.max`) per container | Yes, cgroup v1 or v2 | It caps CPU *time*, not core *speed*: four host cores are much faster than four Cortex-A72 / A76 cores (planned: a per-profile performance factor). Each service gets a share and can't borrow what the others leave idle, as it could on a real board (planned: one shared cgroup slice). |
| Memory | `limits.memory` → `memory.max`; over it, the kernel OOM-kills the container | Yes, cgroup v1 or v2 | Page cache counts towards the limit, as on the board. The OS's own share is left out of the profile (see below), not simulated. |
| Swap | `memswap_limit` = memory: no swap | Yes on cgroup v2; on v1 only with swap accounting (`swapaccount=1`) | A board configured with swap has some; the profiles assume none. |
| Disk bandwidth | `blkio_config` → `io.max` read/write bytes per second, on the whole disk that holds Docker's data root | Yes for direct and buffered I/O on cgroup v2; direct I/O only on v1 | Bandwidth only: no IOPS or latency limit, so small random writes are far faster than on an SD card (planned). Reads served from page cache aren't throttled. Volumes or bind mounts on another disk aren't limited. Skipped when no block device is found (tmpfs, NFS, some btrfs / LVM setups, Docker Desktop). `profile switch` can't change it on running containers. |
| Disk priority | `blkio_config.weight` → `io.bfq.weight` | Only with the BFQ I/O scheduler | Has no effect on hosts using `mq-deadline` or `none`, the usual default for NVMe. |
| Architecture | `platform: linux/arm64` with QEMU user-mode emulation | Yes: the arm64 images and binaries run | Speed: QEMU is roughly 3–10x slower, so timings mean nothing and LLM inference (Ollama) is impractical. Kernel features come from the host kernel, not the board's. |
| Storage size, network, GPIO and other peripherals, accelerators, temperature | — | No | No capacity limit, no network shaping (planned: `tc netem`), no thermal throttling. GPIO is a stub (`p4n4_emu.hw.gpio_stub`); sensors come from the simulator. |

On Docker Desktop the limits apply inside its Linux VM, and the VM's own CPU and memory
settings cap everything on top of them.

### How the profile numbers were chosen

| Profile | Board | CPU | Memory | Disk (read = write) |
|---------|-------|-----|--------|------|
| `rpi4` | Raspberry Pi 4, 4 GB | 4 cores (Cortex-A72) | 3.5 GB: 4 GB minus 512 MB for the OS | 50 MB/s, an SD card |
| `rpi5` | Raspberry Pi 5, 8 GB | 4 cores (Cortex-A76) | 7 GB: 8 GB minus 1 GB for the OS | 100 MB/s, an SD card on the Pi 5's faster slot |
| `nuc` | Intel NUC, 16 GB | 4 cores | 14 GB: 16 GB minus 2 GB for the OS | 200 MB/s, a conservative SSD |
| `mcu-class` | none: a floor for very small Linux devices | 1 core | 256 MB | 10 MB/s |

These are round estimates for each class of board, not measurements; the repo holds no
benchmark behind them. Two known biases: real SD cards write much slower than they read,
while the profiles use one rate for both; and the CPU count says nothing about per-core
speed. `mcu-class` is a small x86 Linux container, not a microcontroller: with 256 MB, the
iot stack's Node-RED gets 38 MiB and InfluxDB 89 MiB (InfluxDB alone used 137 MiB on a
workstation), so expect them to be OOM-killed. To match your own hardware, measure it
(`sysbench cpu`, `fio`, `free -m`) and edit `p4n4_emu/profiles/definitions/*.yml` in `tools/emu`.
