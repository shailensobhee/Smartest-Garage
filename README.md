# Smartest-Garage
Smartest Garage is a sensor-driven automation project designed to monitor and control a garage. Centered on an ESP32-C6, the controller integrates seamlessly with Home Assistant via the ESPHome native API.

The system currently manages car presence detection, door state monitoring, and light automation, alongside RF-based control (433 MHz via CC1101). An mmWave sensor (HLK-LD2410) - a specialized micro movement sensor - supplements the ultrasonic sensor for enhanced presence tracking.

`Plan` : The controller interacts with a local/Private AI (local LLM gateway) for smart decisions: example, the car is on the way back home -> trigger opening the garage. The AI plugs into Google Maps for location and movement intent information. The garage can also be controlled by voice. 

---

## Architecture

```
                          ┌─────────────────────────────────────────────────┐
                          │              Home Assistant (WiFi6)             │
                          │                                                 │
                          │  • Garage Door cover  (open / close / state)    │
                          │  • Garage Door Position  (0–100%)               │
                          │  • Car Present  (yes / no)                      │
                          │  • Garage Light switch                          │
                          │  • 10 pm open-door notification → phone         │
                          └────────────────────┬────────────────────────────┘
                                               │  ESPHome Native Encrypted API
                                               │  (WiFi6)
                          ┌────────────────────▼────────────────────────────┐
                          │     ESP32-C6 DevKitC-1 (WiFi6/Thread/Matter)    │
                          │                  (ESPHome)                      │
                          │                                                 │
  SENSORS ────────────────┤                                                 │
                          │  GPIO4  TRIG ──────────────────────────────────►│
                          │  GPIO5  ECHO ◄──────────────────────────────────┤─── HC-SR04 #1
                          │                                                 │    Door Position (0–100%)
                          │  GPIO22 TRIG ──────────────────────────────────►│
                          │  GPIO23 ECHO ◄──────────────────────────────────┤─── HC-SR04 #2
                          │                                                 │    Car Presence
                          │                                                 │    (ceiling, pointing down)
                          │  GPIO6  ◄───────────────────────────────────────┤─── Reed Switch + Magnet
                          │                                                 │    Door open / closed
                          │                                                 │    (door frame)
                          │  GPIO16 RX ◄────────────────────────────────────┤─── HLK-LD2410  [coming soon]
                          │  GPIO17 TX ─────────────────────────────────────┤    mmWave movement sensor
                          │                                          (UART) │
                          │                                                 │
  OUTPUTS ────────────────┤                                                 │
                          │              ┌──────────────────────────────────┤
                          │  GPIO8 ──────► 3V Optocoupler Relay             │─── Garage Light
                          │              │  (HIGH trigger, NO contact)      │    AC 250V / 10A
                          │              └──────────────────────────────────┤
                          │              ┌──────────────────────────────────┤
                          │  GPIO7 ──────► 3V Optocoupler Relay             │─── Garage Door Opener
                          │              │  (HIGH trigger, NO contact)      │    wall button terminals
                          │              │  [hardwired backup]              │    (backup trigger)
                          │              └──────────────────────────────────┤
                          │                                                 │
  RF ──────────────────── ┤                                                 │
                          │  GPIO10 MOSI ───────────────────────────────────┤
                          │  GPIO11 MISO ◄──────────────────────────────────┤─── CC1101 433 MHz
                          │  GPIO12 CLK  ───────────────────────────────────┤    RF Transceiver
                          │  GPIO13 CS   ───────────────────────────────────┤    (SPI)
                          │  GPIO20 GDO0 (TX) ─────────────────────────────►│
                          │  GPIO21 GDO2 (RX) ◄─────────────────────────────┤─── 433 MHz antenna
                          │                                                 │    (learn + transmit
                          │                                                 │    garage door code)
                          └─────────────────────────────────────────────────┘
```

### Component Summary

| Component | GPIO | Role |
|---|---|---|
| HC-SR04 #1 | GPIO4 / GPIO5 | Door position sensor (0–100%) |
| HC-SR04 #2 | GPIO22 / GPIO23 | Car presence — ceiling mount, pointing down |
| Reed switch + magnet | GPIO6 | Door open/closed state — door frame |
| 3V optocoupler relay (light) | GPIO8 | Garage light — HIGH trigger, NO contact |
| 3V optocoupler relay (door) | GPIO7 | Garage opener backup trigger — wall button terminals |
| CC1101 (SPI) | GPIO10–13 | 433 MHz RF transceiver bus |
| CC1101 GDO0 | GPIO20 | RF transmit — replays captured door code |
| CC1101 GDO2 | GPIO21 | RF receive — learn mode, dumps code to ESPHome logs |
| HLK-LD2410 (UART) | GPIO16 / GPIO17 | mmWave movement detection *(coming soon)* |

### Relay Wiring (3V Optocoupler Module)

```
ESP32-C6 GPIO8 / GPIO7  ──► IN   (signal — HIGH = relay fires)
ESP32-C6 3.3V           ──► VCC
ESP32-C6 GND            ──► GND

Load:  COM ──► live wire in
       NO  ──► live wire out  (circuit closes when relay fires)

Jumper: leave at default TTL position for ESP32-C6
```

### Light Automation Logic

```
Door opens                ──► Light ON  (immediate)
LD2410 detects movement   ──► Light ON  (immediate)   [when sensor arrives]

Door closes  AND
  no LD2410 movement      ──► 2-minute timer ──► Light OFF
```

### CC1101 Learn Workflow

```
1. Flash device
2. Open ESPHome logs
3. Press physical remote  ──► remote_receiver dumps binary code + protocol to logs
4. Copy code → esphome/secrets.yaml (garage_rf_code)
5. Update protocol block in smartest-garage.yaml
6. OTA reflash
7. Test via Home Assistant Garage Door cover entity
```

---

## Quick Start

```bash
# 1. Install dependencies
pip install esphome littlefs-python esptool

# 2. Fill in your credentials
cp esphome/secrets.yaml.example esphome/secrets.yaml   # edit with your WiFi, AP, API key, OTA password
# (secrets.yaml is gitignored — never commit it)

# 3. First flash (USB)
esphome run esphome/smartest-garage.yaml

# 4. Accept the device in Home Assistant → Settings → Devices & Services
```

> **Troubleshooting:**
> - `No module named 'littlefs'` → `pip install littlefs-python`
> - `No module named 'esptool'` → `pip install esptool`
> - `No module named 'fatfs'` or `cannot import name 'create_extended_partition' from 'fatfs'` → do **not** `pip install fatfs`; PlatformIO bundles its own version. Run `platformio pkg update --global` to refresh it, or if the PyPI version is installed, remove it with `pip uninstall fatfs`.

See [`docs/ha-automations/garage-door-notification.yaml`](docs/ha-automations/garage-door-notification.yaml) for the 10 pm open-door notification automation.
