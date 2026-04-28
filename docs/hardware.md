# Hardware

`p4n4-hw` (`shared/hw/`) contains KiCad PCB designs and Raspberry Pi 5 GPIO scripts for the p4n4 platform reference hardware.

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

All designs use [KiCad](https://www.kicad.org/) v8+.

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

**Requirements:**

```bash
pip install RPi.GPIO paho-mqtt
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
| Critical service down (`p4n4-api`) | Rapid 6-pulse alert burst (120 ms) |

```bash
python3 scripts/rpi5/p4n4_health_monitor.py
```

---

### `p4n4_button_handler.py`

Listens for press events on GPIO 27 and dispatches actions based on press type.

| Press type | Action | LED feedback |
|------------|--------|--------------|
| Single press | Print live health report | 1 short pulse |
| Double press | `docker restart` all non-critical services | 3-pulse burst |
| Long press (3 s) | Graceful system shutdown | Fade-out |

```bash
python3 scripts/rpi5/p4n4_button_handler.py
```

---

### `p4n4_mqtt_indicator.py`

Subscribes to key MQTT topics and pulses the LED for each arriving message. Alert topics trigger a faster burst.

| Event | LED pattern |
|-------|-------------|
| Normal message | Single pulse (50 ms on/off) |
| Alert / error message | 5-pulse burst (80 ms on / 50 ms off) |
| Idle (no traffic for 8 s) | Single heartbeat pulse |

Default broker: `localhost:1883`. Override with `--host` / `--port`.

```bash
python3 scripts/rpi5/p4n4_mqtt_indicator.py [--host HOST] [--port PORT]
```

---

## Workstation development

The GPIO scripts require physical RPi hardware. For workstation development, `p4n4_emu.hw.gpio_stub` is a drop-in `RPi.GPIO` replacement — see the [Emulator reference](emulator.md).
