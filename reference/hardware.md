# Hardware

`p4n4-hw` (`tools/hw/`) contains KiCad PCB designs and Raspberry Pi 5 GPIO scripts for the p4n4 platform reference hardware.

## Repository Structure

```
hw/
├── dev-board/               # KiCad project: main dev board (schematic + PCB)
├── prototypes/
│   └── leds-indicator/      # KiCad project: LED indicator prototype board
└── scripts/
    └── rpi5/
        ├── p4n4_common.py         # Shared GPIO helpers, service catalogue, TCP probe
        ├── p4n4_boot_sim.py       # Boot sequence simulator
        ├── p4n4_shutdown_sim.py   # Graceful shutdown simulator
        ├── p4n4_health_monitor.py # Service health monitor (TCP probe + GPIO LED)
        ├── p4n4_button_handler.py # Physical button interface
        └── p4n4_mqtt_indicator.py # MQTT traffic activity indicator
```

## PCB Designs

All designs use [KiCad](https://www.kicad.org/) v8+. They are work in progress: the board and
PCB files are still empty placeholders.

| Design | Location | Description |
|--------|----------|-------------|
| Dev Board | `dev-board/` | Main development board schematic and PCB layout |
| LEDs Indicator | `prototypes/leds-indicator/` | Prototype indicator board for status LEDs |

## RPi5 Scripts

All scripts run on a **Raspberry Pi 5** and share a common GPIO pin assignment via `p4n4_common.py`:

| Pin | Role |
|-----|------|
| GPIO 17 (BCM) | Status LED (all scripts) |
| GPIO 27 (BCM) | Push button (`p4n4_button_handler.py` only) |

**Requirements:** the `RPi.GPIO` API, and `paho-mqtt` 2.0 or later for the MQTT indicator
only.

On the Raspberry Pi 5, `RPi.GPIO` has to be the [`rpi-lgpio`](https://rpi-lgpio.readthedocs.io/)
drop-in. The original `RPi.GPIO` can't drive the Pi 5's GPIO, which moved to the RP1 chip; it
fails with "Cannot determine SOC peripheral base address". Install the drop-in from apt (it
replaces `python3-rpi.gpio`), then create a virtual environment that can see it, since
Raspberry Pi OS blocks `pip install` outside one:

```bash
sudo apt install python3-rpi-lgpio
python3 -m venv --system-site-packages ~/.venvs/p4n4-hw
~/.venvs/p4n4-hw/bin/pip install "paho-mqtt>=2"
~/.venvs/p4n4-hw/bin/python scripts/rpi5/p4n4_health_monitor.py
```

---

### `p4n4_boot_sim.py`

Simulates the p4n4 platform boot sequence. Each phase is reflected by a distinct LED blink pattern.

| Phase | Pattern |
|-------|---------|
| Power-on self-test | Rapid flicker (12×, 80 ms) |
| Bootloader | Fast blink (6×, 150 ms) |
| Kernel load | Medium blink (5×, 300 ms) |
| System services | 1 blink per service (500 ms) |
| Network bridge | 3× burst of 4 pulses (250 ms) |
| IoT stack (MING) | 1 blink per service (600 ms) |
| GenAI stack | 1 blink per service (600 ms) |
| Edge AI stack | 1 blink per service (600 ms) |
| Boot complete | Triple pulse → LED solid on |

```bash
python3 scripts/rpi5/p4n4_boot_sim.py
```

---

### `p4n4_shutdown_sim.py`

Simulates a graceful platform shutdown in reverse boot order (API → Edge AI → GenAI → IoT → network → kernel → power-off).

```bash
python3 scripts/rpi5/p4n4_shutdown_sim.py
```

---

### `p4n4_health_monitor.py`

Probes all p4n4 services via TCP every 10 seconds and reflects aggregate health on the status LED.

| State | LED pattern |
|-------|-------------|
| All services up | Double heartbeat pulse every 4 s |
| Non-critical service(s) down | Slow blink — one blink per failing service (350 ms) |
| Critical service down (`mosquitto` or `influxdb`) | Rapid 6-pulse alert burst (120 ms) |

The broker and InfluxDB are critical because every other service depends on them. Set
`P4N4_CRITICAL_SERVICES` (comma-separated labels from the health report, e.g.
`mosquitto,influxdb,node-red`) to choose others.

```bash
python3 scripts/rpi5/p4n4_health_monitor.py
```

---

### `p4n4_button_handler.py`

Listens for press events on GPIO 27 and dispatches actions based on press type.

| Press type | Action | LED feedback |
|------------|--------|--------------|
| Single press | Print live health report | 1 short pulse |
| Double press | `docker restart` each p4n4 container (`p4n4-mqtt`, `p4n4-influxdb`, …) | 3-pulse burst |
| Long press (3 s) | Graceful system shutdown (`sudo -n shutdown -h now`) | Fade-out |

A restart or shutdown that fails ends with a rapid 6-pulse burst, and the log says why.
Containers that aren't on this host are skipped. The user running the script needs Docker
access (the `docker` group) for restarts, and passwordless sudo for `shutdown`; a narrow
sudoers rule is enough:

```bash
echo "$USER ALL=(root) NOPASSWD: /usr/sbin/shutdown" | sudo tee /etc/sudoers.d/p4n4-shutdown
sudo chmod 0440 /etc/sudoers.d/p4n4-shutdown
```

```bash
python3 scripts/rpi5/p4n4_button_handler.py
```

---

### `p4n4_mqtt_indicator.py`

Subscribes to the topics the platform publishes on (`sensors/#`, `inference/#`, `devices/#`, `alerts/#`) and pulses the LED for each arriving message. Alerts (`alerts/<device-id>/critical`, `alerts/escalated`) trigger a faster burst.

| Event | LED pattern |
|-------|-------------|
| Normal message | Single pulse (50 ms on/off) |
| Alert message | 5-pulse burst (80 ms on / 50 ms off) |
| Idle (no traffic for 8 s) | Single heartbeat pulse |

Default broker: `localhost:1883`. Override with `--host` / `--port`. When the broker requires
a login, set `MQTT_USER` and `MQTT_PASSWORD` in the environment (`--username` / `--password`
also work, but other users can see options in the process list).

```bash
MQTT_USER=indicator MQTT_PASSWORD=... \
  ~/.venvs/p4n4-hw/bin/python scripts/rpi5/p4n4_mqtt_indicator.py [--host HOST] [--port PORT]
```

---

## Workstation development

The GPIO scripts require physical RPi hardware. For workstation development, `p4n4_emu.hw.gpio_stub` is a drop-in `RPi.GPIO` replacement — see the [Emulator reference](emulator.md). The scripts run under it unchanged, including `p4n4_button_handler.py`'s edge detection on GPIO 27. Press the button from a test or a REPL with `GPIO.set_input(27, GPIO.LOW)` and release it with `GPIO.set_input(27, GPIO.HIGH)`. Two presses within 0.4 s count as a double press.
