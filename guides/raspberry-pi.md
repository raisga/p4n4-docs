# Raspberry Pi 5 Node

This guide sets up a Raspberry Pi 5 as a p4n4 node: the stacks running next to the sensors,
with a status LED and a button driven by the [hardware daemon](../reference/hardware.md).
When you finish, the Pi starts the stacks and the daemon on boot, devices publish to it with
a login, and its health shows on the LED and on MQTT.

Rehearse on your workstation first. `p4n4 up --emu rpi5` runs the same project under the
[emulator](../reference/emulator.md)'s Pi 5 profile (4 cores, 7 GB, arm64 under QEMU), so
you find out what fits in 8 GB before you flash anything.

---

## 1. What you need

| Item | Notes |
|------|-------|
| Raspberry Pi 5, 8 GB | 4 GB runs the IoT stack alone; anything with an LLM needs 8 GB |
| Storage | An NVMe SSD on an M.2 HAT is strongly recommended: InfluxDB and Docker layers are slow and wear a microSD card. A 64 GB A2 card works for trying things out |
| Power and cooling | The official 27 W USB-C supply and the Active Cooler. Under load, the Pi 5 throttles without one |
| LED and button (optional) | An LED with a 330 Ω resistor, a momentary push button, jumper wires |

## 2. Operating system

1. Flash **Raspberry Pi OS Lite (64-bit)** with Raspberry Pi Imager. In its settings, set
   the hostname (it becomes the node's id in MQTT topics, e.g. `pi-greenhouse`), a user,
   your Wi-Fi if needed, and enable SSH with your public key.
2. Boot, SSH in, and update:

   ```bash
   sudo apt update && sudo apt full-upgrade -y
   sudo reboot
   ```

## 3. Docker and the CLI

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker "$USER"          # log out and back in for it to apply

sudo apt install -y pipx git
pipx ensurepath                           # then open a new shell
pipx install p4n4
p4n4 --version
```

Raspberry Pi OS blocks `pip install` outside a virtual environment, which is why the CLI goes
in with `pipx`. Its Python (3.11 on bookworm, 3.13 on trixie) meets the CLI's 3.11 minimum.

## 4. Choose what runs

The Pi 5 has 8 GB for everything: the OS, the stacks and any model. Pick one row:

| Node | `p4n4 init` | AI `COMPOSE_PROFILES` | Memory (rough) |
|------|-------------|-----------------------|--------|
| Telemetry only | `p4n4 init node` | — | About 1 GB |
| Telemetry + edge inference | `p4n4 init node --layer iot,ai --ei-runner` | `ei-runner` (remove `ollama`) | About 1.5 GB plus the model |
| Telemetry + a local LLM | `p4n4 init node --layer iot,ai` | `ollama` | 3–5 GB with a model of 2–4 GB |
| Telemetry + System 1 | `p4n4 init node --layer iot,ai`, then `p4n4 ai system1 enable` | `ollama,system1` or `system1` | Plan 2 GiB for System 1 |

- **Edge inference without an LLM** is the usual sensor-side node
  ([ADR-005](../decisions/adr/ADR-005.md)): set `COMPOSE_PROFILES=ei-runner` in
  `node/ai/.env`. Download the **AARCH64** `.eim` build from Edge Impulse Studio, or use an
  ONNX model; see [Edge Inference](../stacks/edge-inference.md).
- **LLMs** run on the CPU: keep to small models (`phi3:mini`, `gemma4:e2b`, `qwen2.5:3b`) and
  `OLLAMA_MAX_LOADED_MODELS=1`. Expect answers in tens of seconds, not seconds.
- **System 1** hasn't been measured natively on a Pi 5 yet (an S1 exit criterion in
  [System 1](../decisions/system1.md)); treat its latency as unknown.
- **The dashboard** is light (nginx serving static files): add `dashboard` to `--layer` to
  serve it from the node ([Dashboard service](../stacks/dashboard.md)).

Then start it:

```bash
cd node
p4n4 up
p4n4 status
```

`RESTART_POLICY=unless-stopped` (the default) brings every container back after a reboot,
and Docker starts on boot, so the stacks need nothing else to survive a power cut.

## 5. Require a login from devices

Out of the box, devices on your LAN can publish anonymously. Before connecting real ones,
set a device login in `node/iot/.env` (or `node/.env` for a single-layer project) and
restart the broker:

```bash
MQTT_USER=devices
MQTT_PASSWORD=<a long random value>
```

```bash
cd iot && docker compose up -d mqtt && cd ..      # recreates the broker with the new .env
docker logs p4n4-mqtt 2>&1 | grep 'p4n4 mqtt'    # "... need a login: 1 account(s) ..."
```

Give each device and the hardware daemon an account of its own with
`make mqtt-user NAME=<name>` in the iot layer, so you can revoke one without touching the
others. The stack's own services use an internal listener and need nothing. See
[Mosquitto authentication](security.md#mosquitto-authentication).

## 6. LED and button

Wire them to the 40-pin header (BCM numbering):

| Part | Pin | Header pins |
|------|-----|-------------|
| LED | GPIO 17, through the 330 Ω resistor, LED's cathode to GND | 11 (GPIO 17), 9 (GND) |
| Button | GPIO 27 to GND; the daemon enables the internal pull-up | 13 (GPIO 27), 14 (GND) |

Then install the daemon from a checkout of [p4n4-hw](https://github.com/raisga/p4n4-hw):

```bash
git clone https://github.com/raisga/p4n4-hw.git ~/p4n4-hw
~/p4n4-hw/scripts/rpi5/install-hwd.sh --shutdown
```

It installs `rpi-lgpio` (the Pi 5's `RPi.GPIO`), a virtual environment with paho-mqtt, adds
you to the `gpio` and `docker` groups, and starts `p4n4-hwd.service`. `--shutdown` lets a
3-second press power the Pi off.

Edit `/etc/p4n4/hwd.env` to match the node, then `sudo systemctl restart p4n4-hwd`:

```bash
P4N4_SERVICES=mosquitto,influxdb,node-red,grafana    # only what this node runs
MQTT_USER=hwd                                        # its account from step 5
MQTT_PASSWORD=<its password>
```

Leave out the services this node doesn't run, or the LED reports them as down.

## 7. Check it

```bash
sudo apt install -y mosquitto-clients
journalctl -u p4n4-hwd -f                       # the daemon's log
mosquitto_sub -h localhost -u devices -P '<password>' -t 'status/#' -v
```

The LED stays dark until the first check, then gives a double heartbeat every 4 seconds when
everything is up. `status/<hostname>/online` and `status/<hostname>/health` are retained, so
any client sees the node's state as soon as it subscribes. A single press logs a health report
and checks again right away; see the [LED patterns and presses](../reference/hardware.md#p4n4_hwdpy-hardware-daemon).

To reach Grafana and Node-RED from other machines without opening ports, use
[Tailscale](https://tailscale.com/) as in the [Jetson guide](jetson.md); the steps are the same
on Raspberry Pi OS.

## Troubleshooting

| Symptom | Cause and fix |
|---------|---------------|
| `Cannot determine SOC peripheral base address` | The original `RPi.GPIO`, which can't drive the Pi 5. Install `python3-rpi-lgpio` (`install-hwd.sh` does) |
| The LED blinks slowly although everything works | A service in `P4N4_SERVICES` that this node doesn't run. Narrow the list |
| Containers restart or get killed under load | Out of memory: `docker stats`, then a smaller model or fewer AI services. `docker inspect <container> --format '{{.State.OOMKilled}}'` confirms it |
| A double press does nothing | The daemon's user isn't in the `docker` group yet: re-log or reboot after `install-hwd.sh` |
| Devices get "not authorised" | `MQTT_PASSWORD` is set: they need `MQTT_USER` or an account in `config/mosquitto/passwd` |

More in [Troubleshooting](troubleshooting.md).
