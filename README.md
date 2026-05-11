# Big Clock — ESPHome

ESPHome firmware for the Big NTP Clock, replacing the original native Arduino/PlatformIO firmware. Same hardware (upgraded MCU), same features — but now managed from the ESPHome dashboard alongside all your other ESPHome devices.

**Original native firmware:** [chilled35/BigClock](https://github.com/chilled35/BigClock)

## Hardware

| Component | Detail |
|---|---|
| Microcontroller | Waveshare ESP32-S3-Zero (clone) |
| Clock digit strip | 252 LEDs, WS2812 GRB, GPIO4 |
| Downlight strip | 14 LEDs, WS2812 GRB, GPIO5 |
| Light sensor | LDR on GPIO2 (A0, ADC1) |
| PSU | Meanwell 60W 5V |

> **Note:** The original build used a Seeed Studio XIAO ESP32-C3. The ESP32-S3-Zero is a drop-in upgrade — same physical pin positions on the board, same GPIO numbers, no rewiring required.

## ESPHome config key settings

| Setting | Value | Reason |
|---|---|---|
| `board` | `esp32-s3-devkitc-1` | Closest known ESPHome board ID |
| `flash_size` | `4MB` | Waveshare ESP32-S3-Zero has 4MB (devkitc-1 defaults to 8MB — mismatch causes boot loop) |
| `framework` | `arduino` | ESP32-S3 is dual-core; WiFi runs on core 0, app on core 1 — no RMT/WiFi conflicts |
| LED method | `neopixelbus / esp32_rmt` | Reliable hardware-timed output |
| RMT channels | clock=2, down=3 | Channels 0+1 are pre-claimed by the devkitc-1 board init |
| `power_save_mode` | `none` | Keeps WiFi radio awake continuously |
| `reboot_timeout` | `0s` | Device retries WiFi indefinitely without rebooting |

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
| Colour presets | None | 5 named presets with colour pickers + apply buttons |

## Home Assistant entities

After adding the device, HA will automatically show these under **Settings → Devices → Big Clock**:

### Display

| Entity | Type | Description |
|---|---|---|
| Hour Colour | Light (RGB) | Colour of the two hour digits |
| Minute Colour | Light (RGB) | Colour of the two minute digits |
| Downlight | Light (RGB) | Decorative downlight strip |
| Rainbow Mode | Switch | Toggle rainbow wave animation |
| Ambient Light | Sensor | Raw LDR voltage (for diagnostics) |
| Randomise Colours | Button | Pick random hour + minute colours |

### Presets

5 named colour presets are built in. Each preset stores independent hour and minute colours.

| Entity | Type | Description |
|---|---|---|
| Preset 1–5 Hour Colour | Light (RGB) | Edit the hour digit colour for this preset |
| Preset 1–5 Min Colour | Light (RGB) | Edit the minute digit colour for this preset |
| Apply Preset 1–5 | Button | Push the preset colours to the active display |
| Preset 1–5 Name | Text | Editable label shown in the HA device page |

**Default preset colours:**

| Preset | Hour | Minute |
|---|---|---|
| 1 — Classic | White | White |
| 2 — Sunset | Orange `#FF6600` | Red `#FF0000` |
| 3 — Ocean | Cyan `#00FFFF` | Blue `#0066FF` |
| 4 — Forest | Green `#00FF00` | Lime `#AAFF00` |
| 5 — Candy | Pink `#FF00AA` | Purple `#AA00FF` |

Preset colours are seeded on first boot only — editing a preset in HA persists across reboots.

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

Open ESPHome → **+ New device** → **Skip** → paste or upload `bigclock-esphome.yaml`. Make sure `secrets.yaml` is in the same directory.

### 4. First flash (USB)

The first flash must be over USB — OTA isn't available until the firmware is running.

Connect the ESP32-S3-Zero via USB-C and flash from the ESPHome dashboard, or:

```bash
esphome run bigclock-esphome.yaml
```

### 5. Subsequent updates (OTA)

Click **Install → Wirelessly** in the ESPHome dashboard, or:

```bash
esphome run bigclock-esphome.yaml
```

---

## MQTT display messages

The clock shows the time by default. Publish a JSON message to `bigclock/display` and the clock immediately switches to showing that value for **5 seconds**, then reverts to the time automatically.

### Message format

```json
{
  "value": 22,
  "format": "temp",
  "color": "#FF6600",
  "suffix_color": "#00CCFF"
}
```

| Field | Required | Description |
|---|---|---|
| `value` | Yes | Integer to display. Send `-1` to cancel immediately. |
| `format` | No | `"temp"` (default) → `XX °C`, clamped 0–99. `"number"` → `XXXX`, clamped 0–9999. |
| `color` | No | Hex colour for the value digits. Defaults to white `#FFFFFF`. |
| `suffix_color` | No | Hex colour for the `°C` suffix in temp mode. Defaults to `color`. |

### Display modes

**`"format": "temp"`** — two digits plus degree and C:
```
[ 2 ][ 2 ][ ° ][ C ]     value: 22
```

**`"format": "number"`** — four digits, full width:
```
[ 4 ][ 2 ][ 5 ][ 0 ]     value: 4250
```

### Sequencing multiple values

An HA automation sends messages one at a time, with a 5-second delay between them. Each message gets its own 5-second slot on the clock face.

```yaml
alias: BigClock — hot tub + solar display
triggers:
  - trigger: time_pattern
    minutes: "/5"   # run every 5 minutes
actions:
  # Slide 1: hot tub temperature (orange digits, cyan °C)
  - action: mqtt.publish
    data:
      topic: bigclock/display
      payload: >
        {"value": {{ states('sensor.hottub_temperature') | int }},
         "format": "temp",
         "color": "#FF6600",
         "suffix_color": "#00CCFF"}

  - delay: "00:00:05"

  # Slide 2: solar generation in watts (yellow, 4-digit)
  - action: mqtt.publish
    data:
      topic: bigclock/display
      payload: >
        {"value": {{ states('sensor.solar_power') | int }},
         "format": "number",
         "color": "#FFFF00"}
```

### Cancel early

Send `{"value": -1}` to immediately clear any active overlay and return to the clock:

```yaml
- action: mqtt.publish
  data:
    topic: bigclock/display
    payload: '{"value": -1}'
```

---

## MQTT topics

| Topic | Direction | Description |
|---|---|---|
| `bigclock/availability` | publish | `online` / `offline` (retained) |
| `bigclock/display` | subscribe | JSON display message (see above) |
| `bigclock/light/hour_colour/command` | subscribe | HA light command (auto-managed) |
| `bigclock/light/minute_colour/command` | subscribe | HA light command (auto-managed) |
| `bigclock/light/downlight/command` | subscribe | HA light command (auto-managed) |
| `bigclock/switch/rainbow_mode/command` | subscribe | `ON` / `OFF` |
| `bigclock/button/randomise_colours/command` | subscribe | `PRESS` |

All entity topics follow ESPHome's standard MQTT topic structure and are created automatically via MQTT discovery.

---

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

ESPHome restores the last known state of all colour light entities on reboot (`restore_mode: RESTORE_DEFAULT_ON`). Hour colour, minute colour, downlight colour, and all preset colours survive power cycles.

## Extending

**Add more display formats:** Extend `g_display_format` with new integer values and add corresponding rendering branches in the `Clock Display` lambda.

**Trigger from non-MQTT sources:** Any ESPHome action can set `g_display_value`, `g_display_format`, `g_display_r/g/b`, and `g_display_until_ms` directly via a `globals.set` / `lambda` action — no MQTT required.
