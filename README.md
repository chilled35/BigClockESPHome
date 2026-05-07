# Big Clock — ESPHome

ESPHome firmware for the Big NTP Clock, replacing the original native Arduino/PlatformIO firmware. Same hardware, same features — but now managed from the ESPHome dashboard alongside all your other ESPHome devices.

**Original native firmware:** [chilled35/BigClock](https://github.com/chilled35/BigClock)

## Hardware

Identical to the original — no changes needed.

| Component | Detail |
|---|---|
| Microcontroller | Seeed Studio XIAO ESP32-C3 |
| Clock digit strip | 252 LEDs, WS2812 GRB, GPIO5 (D3) |
| Downlight strip | 14 LEDs, WS2812 GRB, GPIO4 (D2) |
| Light sensor | LDR on GPIO2 (A0, ADC1) |
| PSU | Meanwell 60W 5V |

## What changed vs the native firmware

| Feature | Native firmware | This ESPHome config |
|---|---|---|
| Toolchain | VS Code + PlatformIO | ESPHome dashboard |
| OTA updates | ArduinoOTA (~75 s) | ESPHome OTA (faster, integrated) |
| HA integration | MQTT discovery only | Native ESPHome API + MQTT discovery |
| BST/GMT | Manual last-Sunday calculation | `timezone: "Europe/London"` — automatic |
| Colour control | Custom web UI at `/` and `/fx` | HA colour pickers (RGB light entities) |
| Credentials | Hardcoded in `main.cpp` | `secrets.yaml` (gitignored) |
| Config | Recompile to change anything | Edit YAML, flash OTA |

## Home Assistant entities

After adding the device, HA will automatically show these under **Settings → Devices → Big Clock**:

| Entity | Type | Description |
|---|---|---|
| Hour Colour | Light (RGB) | Colour of the two hour digits |
| Minute Colour | Light (RGB) | Colour of the two minute digits |
| Downlight | Light (RGB) | Decorative downlight strip |
| Rainbow Mode | Switch | Toggle rainbow wave animation |
| Ambient Light | Sensor | Raw LDR voltage (for diagnostics) |
| Randomise Colours | Button | Pick random hour + minute colours |

## Setup

### 1. Prerequisites

- ESPHome installed (as a Home Assistant add-on, or standalone via pip)
- Mosquitto MQTT broker running (HA add-on or standalone)

### 2. Configure secrets

```bash
cp secrets.yaml.example secrets.yaml
```

Edit `secrets.yaml` with your WiFi credentials, MQTT broker details, and an ESPHome API key. The API key is a 32-byte base64 string — the ESPHome dashboard generates one automatically when you add a new device, or generate it manually:

```bash
python3 -c "import base64, os; print(base64.b64encode(os.urandom(32)).decode())"
```

### 3. Add to ESPHome dashboard

If using the HA add-on: open ESPHome → **+ New device** → **Skip** (since you have the YAML already) → paste or upload `bigclock-esphome.yaml`. Make sure `secrets.yaml` is in the same directory as the config.

### 4. First flash (USB)

The first flash must be over USB — OTA isn't available until the firmware is running.

```bash
esphome run bigclock-esphome.yaml
```

Connect the XIAO via USB-C. ESPHome will compile, detect the port, and flash automatically.

### 5. Subsequent updates (OTA)

Once the clock is on your network, all future updates go over WiFi:

```bash
esphome run bigclock-esphome.yaml
```

Or click **Install → Wirelessly** in the ESPHome dashboard.

## MQTT — displaying external sensor values

The clock subscribes to `bigclock/display`. Publish any numeric string to it and the clock will display that value (as `XX °C`) for 5 seconds every 10-second cycle.

**Example HA automation** (hot tub temperature):

```yaml
alias: Push hot tub temp to BigClock
triggers:
  - trigger: state
    entity_id: sensor.hottub_temperature
actions:
  - action: mqtt.publish
    data:
      topic: bigclock/display
      payload: "{{ states('sensor.hottub_temperature') }}"
```

The value is displayed floored to an integer (e.g. `22.75` → `22`), clamped to 0–99.

To clear the external value and return to time-only display, publish an empty string or a negative number:

```yaml
- action: mqtt.publish
  data:
    topic: bigclock/display
    payload: "-1"
```

## MQTT topics

| Topic | Direction | Description |
|---|---|---|
| `bigclock/availability` | publish | `online` / `offline` (retained) |
| `bigclock/display` | subscribe | Float string to display (e.g. `"22.5"`) |
| `bigclock/light/hour_colour/command` | subscribe | HA light command (auto-managed) |
| `bigclock/light/minute_colour/command` | subscribe | HA light command (auto-managed) |
| `bigclock/light/downlight/command` | subscribe | HA light command (auto-managed) |
| `bigclock/switch/rainbow_mode/command` | subscribe | `ON` / `OFF` |
| `bigclock/button/randomise_colours/command` | subscribe | `PRESS` |

All entity topics follow ESPHome's standard MQTT topic structure and are published via MQTT discovery — you don't need to configure them manually in HA.

## Segment layout reference

```
 ─────      pixels  9–17  (top)
│     │     pixels 18–26  (left-top)     pixels  0–8  (right-top)
 ─────      pixels 27–35  (middle)
│     │     pixels 45–53  (left-bottom)  pixels 36–44 (right-bottom)
 ─────      pixels 54–62  (bottom)
```

Digit pixel bases: `hours-tens=189`, `hours-units=126`, `mins-tens=63`, `mins-units=0`

## Colour persistence

ESPHome restores the last known state of the Hour Colour and Minute Colour light entities on reboot (`restore_mode: RESTORE_DEFAULT_ON`). Colours survive power cycles without any extra flash storage code.

## Extending

**Add more display modes:** Add extra `addressable_lambda` effects to `clock_strip` and switch between them using automations or a `select` entity.

**Display non-temperature values:** The `bigclock/display` topic accepts any integer 0–99. The `°C` suffix is hardcoded in the rendering lambda — edit `draw(10 ...)` / `draw(11 ...)` to change or remove it if you want to display other things.

**Presets:** Create HA scenes that set Hour Colour and Minute Colour together, or use the Randomise button for instant variety.
