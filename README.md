# Smartest-Garage
A SmartHome, sensor-driven project to monitor and control a garage using an ESP32-C6 and Home Assistant.

---

## Architecture

```
                          ┌─────────────────────────────────────────────────┐
                          │              Home Assistant (WiFi)               │
                          │                                                  │
                          │  • Garage Door cover  (open/close/state)         │
                          │  • Garage Door Position  (0–100%)                │
                          │  • Car Present  (yes/no)                         │
                          │  • Garage Light switch                           │
                          │  • 10 pm open-door phone notification            │
                          └────────────────────┬────────────────────────────┘
                                               │  ESPHome Native Encrypted API
                                               │  (WiFi)
                          ┌────────────────────▼────────────────────────────┐
                          │                                                  │
                          │               ESP32-C6 DevKitC-1                │
                          │                  (ESPHome)                       │
                          │                                                  │
       ┌──────────────────┤  GPIO4  TRIG ◄──────────────────────────────────┤──┐
       │                  │  GPIO5  ECHO ──────────────────────────────────►│  │  HC-SR04 #1
       │   Door           │                                                  │  │  Door Position
       │   Position       │                                                 ─┘  │  (0–100%)
       │   (0–100%)       ├──────────────────────────────────────────────────  │
       │                  │  GPIO22 TRIG ◄──────────────────────────────────┐  │
       │                  │  GPIO23 ECHO ──────────────────────────────────►│  │  HC-SR04 #2
       │                  │                                                  │  │  Car Presence
       │                  │                                             Ceiling  │  (ceiling mount)
       │                  │                                                  │──┘
       │                  │
       │                  │  GPIO6  ◄── Reed Switch + Magnet ─────────────── Door frame
       │                  │             (door open/closed)
       │                  │
       │                  │  GPIO7  ──► Door Relay  ──────────────────────── Garage opener
       │                  │             (hardwired backup trigger)            wall button terminals
       │                  │
       │                  │  GPIO8  ──► Light Relay ──────────────────────── Garage light
       │                  │             (on: door open / movement)
       │                  │             (off: 2 min after door closed
       │                  │                   + no movement)
       │                  │
       │                  │  GPIO10 MOSI ─┐
       │                  │  GPIO11 MISO ─┤
       │                  │  GPIO12 CLK  ─┼──────────────────────────────── CC1101
       │                  │  GPIO13 CS   ─┘                                 433 MHz RF
       │                  │  GPIO20 GDO0 (TX) ────────────────────────────► transceiver
       │                  │  GPIO21 GDO2 (RX) ◄──────────────────────────── (learn + transmit
       │                  │                                                   garage door code)
       │                  │
       │                  │  GPIO16 RX ◄──┐
       │                  │  GPIO17 TX ──►│─────────────────────────────── HLK-LD2410
       │                  │               │                                 mmWave sensor
       │                  │          (UART)                                 (movement detection)
       │                  │          [coming soon]                          [coming soon]
       │                  │
       └──────────────────┘
```

### Component Summary

| Component | GPIO | Role |
|---|---|---|
| HC-SR04 #1 | GPIO4 / GPIO5 | Door position sensor (0–100%) |
| HC-SR04 #2 | GPIO22 / GPIO23 | Car presence (ceiling mount, pointing down) |
| Reed switch | GPIO6 | Door open/closed state |
| Door relay | GPIO7 | Hardwired backup door trigger |
| Light relay | GPIO8 | Garage light control |
| CC1101 SPI | GPIO10–13 | 433 MHz RF transceiver (SPI bus) |
| CC1101 GDO0 | GPIO20 | RF transmit (door open/close signal) |
| CC1101 GDO2 | GPIO21 | RF receive (learn mode) |
| HLK-LD2410 | GPIO16 / GPIO17 | mmWave movement detection *(coming soon)* |

### Light Automation Logic

```
Door opens                ──► Light ON  (immediate)
LD2410 detects movement   ──► Light ON  (immediate)

Door closes AND
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
# 1. Install ESPHome
pip install esphome

# 2. Fill in your credentials
cp esphome/secrets.yaml.example esphome/secrets.yaml   # edit with your WiFi + API key
# (secrets.yaml is gitignored — never commit it)

# 3. First flash (USB)
esphome run esphome/smartest-garage.yaml

# 4. Accept the device in Home Assistant → Settings → Devices & Services
```

See [`docs/ha-automations/garage-door-notification.yaml`](docs/ha-automations/garage-door-notification.yaml) for the 10 pm open-door notification automation.
