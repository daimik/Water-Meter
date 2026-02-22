# 💧 Smart Water Meter

Pulse-based water meter monitoring system with ESPHome, featuring leak detection, usage statistics, flow rate tracking, and Home Assistant integration.

---

## 🎯 Project Overview

This project turns an **Elster V200 volumetric water meter** into a smart IoT device using a proximity sensor and a Seeed XIAO ESP32C6. Every rotation of the meter disk triggers the sensor once — that pulse is counted, stored, and published to Home Assistant in real time.

| Aspect | Details |
|--------|---------|
| **Difficulty** | ⭐⭐ Intermediate |
| **Build Time** | ~2-4 hours |
| **Cost** | ~€20-35 total |
| **Config File** | `water-meter.yaml` |
| **Board** | Seeed XIAO ESP32C6 |
| **Target Meter** | Elster V200 volumetric water meter |

---

## 📦 Hardware

### Shopping List

| Component | Supplier | Price | Notes |
|-----------|----------|-------|-------|
| **Seeed XIAO ESP32C6** | Kiwi Electronics | ~€6 | [Link](https://www.kiwi-electronics.com/nl/seeed-studio-xiao-esp32c6-20076) |
| **2.4GHz WiFi Antenna (uFL)** | Kiwi Electronics | ~€3 | [Link](https://www.kiwi-electronics.com/nl/2-4ghz-mini-flexibele-wifi-antenne-met-ufl-connector-11062) |
| **LJ18A3-8-Z/BX-5V Proximity Sensor** | AliExpress | ~€2-3 | NPN NO, 8mm, M18, 5V — [Link](https://nl.aliexpress.com/item/1005004537279157.html) |
| **Sensor Bracket (3D print)** | Thingiverse | Free | M18×1 thread, fits Elster V200 — [Link](https://www.thingiverse.com/thing:5515823) |
| **Voltage Divider Resistors** | Local/AliExpress | <€1 | 10kΩ + 6.8kΩ |
| **2× M4 bolts** | Hardware store | <€1 | To mount bracket on Elster V200 |
| **1× M18×1 locking nut** | Hardware store | <€1 | To lock sensor in bracket |
| **Total** | | **~€13-15** | Excluding enclosure |

### Additional Materials
- IP65 rated enclosure
- Low voltage wire
- Cable glands and strain relief
- Mounting hardware
- USB-C cable (for programming)
- 5V power supply

---

## 🔌 Wiring

### Proximity Sensor → ESP32C6

| Sensor Wire | Color | Connect To |
|-------------|-------|------------|
| Power | Brown | 5V |
| Ground | Blue | GND |
| Signal | Black | Voltage divider → GPIO17 (D7) |

### ⚠️ Voltage Divider (REQUIRED)

The LJ18A3-8-Z/BX-5V sensor outputs **5V logic** but the XIAO ESP32C6 GPIO pins are **3.3V only**. A voltage divider is required on the signal wire:

```
Sensor Black ──→ 10kΩ ──→ GPIO17
                           │
                          6.8kΩ
                           │
                          GND
```

This brings the 5V signal down to ~3.1V which is safe for the ESP32C6.

### 🖨️ Sensor Bracket (3D Print)

A purpose-built bracket for mounting the LJ18A3-8-Z/BX-5V on an **Elster V200** water meter is available on Thingiverse: [thing:5515823](https://www.thingiverse.com/thing:5515823)

- Designed specifically for the Elster V200
- **M18×1 thread** matches the sensor body exactly — no tapping needed
- One locking nut holds the sensor at the correct 8mm sensing distance
- Two M4 bolts fix the bracket to the meter
- Minimalistic design so you can still manually read the meter

### Antenna

The XIAO ESP32C6 has an onboard RF switch for antenna selection. The firmware automatically enables the external antenna at boot via GPIO3 and GPIO14. Simply plug the uFL antenna into the connector on the board.

---

## 📋 Pin Configuration

| Pin Label | GPIO | Function | Notes |
|-----------|------|----------|-------|
| D7 | GPIO17 | Proximity Sensor Signal | Input, pull-up, voltage divider required |
| - | GPIO3 | RF Switch Control | Reserved — antenna control |
| - | GPIO14 | Antenna Select | Reserved — external antenna |

---

## ✨ Features

### Core Functionality
- ✅ Pulse counting via NPN proximity sensor
- ✅ Configurable pulses-per-liter or liters-per-pulse
- ✅ Settable total meter value (sync with physical meter)
- ✅ All values survive reboots (flash storage)
- ✅ Pulses counted even when WiFi is offline
- ✅ Automatic WiFi reconnection

### Usage Statistics
- 📅 Water usage **today**
- 📅 Water usage **this week** (resets Monday)
- 📅 Water usage **this month** (resets 1st)
- 📅 Water usage **this year** (resets Jan 1st)
- 🔢 Raw pulse count total
- 💧 Total meter value in Liters and m³

### Flow Monitoring
- 🌊 Flow rate in L/min (updated every 60 seconds)
- 📈 Peak flow rate ever recorded
- 🔄 Reset button for peak flow

### Leak Detection
- 🚨 Automatic leak alert if water flows continuously for configurable minutes
- 🔔 Exposed as `moisture` binary sensor in HA — use for push notifications
- ⚙️ Configurable threshold (default 60 minutes, range 10–480 min)
- 🔄 Reset button for leak alert
- ✅ Auto-clears when flow stops

### Configuration (from HA dashboard)
- 🔢 **Set Watermeter Value** — sync with physical meter reading
- 🔢 **Pulses Per Liter** — set if multiple rotations = 1 liter
- 🔢 **Liters Per Pulse** — set if 1 rotation = multiple liters
- ⏱️ **Leak Detection Threshold** — minutes before alert triggers

### Diagnostics
- 📡 WiFi signal strength
- ⏰ Device uptime
- 🌐 IP address, SSID, MAC address
- 🔧 ESPHome firmware version

---

## ⚙️ Pulse Configuration

Your meter has one of these configurations. Set the correct one and leave the other at 1:

| Your Meter | Setting | Value |
|------------|---------|-------|
| 1 rotation = 1 liter *(standard)* | Either — leave both at `1` | `1` |
| 2 rotations = 1 liter | **Pulses Per Liter** | `2` |
| 5 rotations = 1 liter | **Pulses Per Liter** | `5` |
| 1 rotation = 10 liters | **Liters Per Pulse** | `10` |

> **Note:** Only ever change one setting — they both update the same internal factor and setting one overrides the other.

---

## 🔍 How Pulse Detection Works

The proximity sensor produces this signal pattern for each meter rotation:

```
Meter disk spinning...

Signal: ──────╮        ╭──────
              │  (ON)  │
              ╰────────╯
              ↑
         on_press fires here
         → 1 pulse counted
```

- `OFF → ON` transition = **1 pulse counted** (on_press)
- Duration the sensor stays ON is **completely irrelevant**
- `ON → OFF` transition = **ignored** (on_release not used)
- Debounce filters ensure noise spikes shorter than 500ms are ignored

---

## 🕐 Automatic Period Resets

All period counters reset automatically via Home Assistant time sync:

| Period | Resets When |
|--------|------------|
| **Today** | Every day at 00:00 |
| **This Week** | Every Monday at 00:00 |
| **This Month** | 1st of every month at 00:00 |
| **This Year** | January 1st at 00:00 |

> **Note:** Time sync requires HA API connection. If the device reboots while offline, period counters are preserved but time-based resets won't fire until HA reconnects.

---

## 🚨 Leak Detection

The leak detection system monitors for **continuous uninterrupted water flow**:

```
Normal use:   ██░░░░░██░░░░░██░░░░░  (flow with gaps = OK)
Leak:         ████████████████████  (continuous flow = ALERT)
```

1. Flow rate is checked every 60 seconds
2. Counter increments for each minute with flow > 0
3. Counter resets to 0 when flow stops
4. Alert fires when counter reaches threshold (default 60 minutes)
5. Alert clears automatically when flow stops

### Home Assistant Automation Example

```yaml
automation:
  - alias: "Water Leak Alert"
    trigger:
      - platform: state
        entity_id: binary_sensor.water_meter_leak_alert
        to: "on"
    action:
      - service: notify.mobile_app
        data:
          title: "⚠️ Water Leak Detected!"
          message: "Continuous water flow detected for over 60 minutes."
```

---

## 🌐 Web Interface

Access the built-in web dashboard at `http://[DEVICE_IP]`

### Sections

| Section | Content |
|---------|---------|
| **Sensors** | All meter values, flow rate, usage periods |
| **Controls** | Set meter value, configure pulse factor, leak threshold |
| **Buttons** | Reset peak flow, leak alert, period counters |
| **Diagnostics** | WiFi signal, uptime, IP, firmware version |

---

## 🏠 Home Assistant Integration

Device auto-discovers in Home Assistant once the API key is configured.

### Available Entities

#### Sensors
| Entity | Unit | Description |
|--------|------|-------------|
| `sensor.watermeter_value` | L | Total meter value |
| `sensor.watermeter_value_m3` | m³ | Total meter value in cubic meters |
| `sensor.watermeter_pulse_count` | — | Raw total pulse count |
| `sensor.water_used_last_minute` | L/min | Current flow rate |
| `sensor.peak_flow_rate` | L/min | Highest flow rate recorded |
| `sensor.water_usage_today` | L | Usage since midnight |
| `sensor.water_usage_this_week` | L | Usage since Monday |
| `sensor.water_usage_this_month` | L | Usage since 1st |
| `sensor.water_usage_this_year` | L | Usage since Jan 1st |
| `sensor.wifi_signal` | dBm | WiFi signal strength |
| `sensor.uptime` | s | Device uptime |

#### Binary Sensors
| Entity | Device Class | Description |
|--------|-------------|-------------|
| `binary_sensor.leak_alert` | moisture | ON when leak detected |

#### Numbers (configurable from HA)
| Entity | Description |
|--------|-------------|
| `number.set_watermeter_value` | Set total meter value in liters |
| `number.pulses_per_liter` | How many pulses = 1 liter |
| `number.liters_per_pulse` | How many liters = 1 pulse |
| `number.leak_detection_threshold_min` | Minutes before leak alert fires |

#### Buttons
| Entity | Description |
|--------|-------------|
| `button.reset_peak_flow` | Reset peak flow rate to 0 |
| `button.reset_leak_alert` | Clear active leak alert |
| `button.reset_daily_usage` | Reset today's counter |
| `button.reset_weekly_usage` | Reset this week's counter |
| `button.reset_monthly_usage` | Reset this month's counter |
| `button.reset_yearly_usage` | Reset this year's counter |

#### Text Sensors
| Entity | Description |
|--------|-------------|
| `text_sensor.ip_address` | Device IP address |
| `text_sensor.wifi_ssid` | Connected WiFi network |
| `text_sensor.mac_address` | Device MAC address |
| `text_sensor.esphome_version` | Firmware version |

### Energy Dashboard

The `Watermeter Value m³` sensor is configured with:
```yaml
state_class: total_increasing
unit_of_measurement: m³
device_class: water
```
This makes it compatible with the **Home Assistant Energy Dashboard** under water monitoring.

---

## 🚀 Installation

### Step 1: Configure secrets

```yaml
# secrets.yaml
wifi_ssid: "YourWiFiName"
wifi_password: "YourWiFiPassword"
```

### Step 2: Flash firmware

```bash
esphome run water-meter.yaml
```

### Step 3: Add to Home Assistant

1. Go to **Settings → Devices & Services**
2. ESPHome integration will auto-discover the device
3. Enter the API encryption key when prompted

### Step 4: Set meter value

1. Read your physical water meter
2. In HA, find **Set Watermeter Value** number entity
3. Enter your current meter reading in liters
4. Your ESPHome meter is now synced with the physical one

---

## 🔧 Troubleshooting

### Pulses not counting
- Check voltage divider wiring — 5V signal without divider will damage GPIO
- Verify sensor wire colors (Brown=5V, Blue=GND, Black=Signal)
- Check sensor LED indicator — it should light up when the metal target passes
- Confirm GPIO17 in config matches your actual wiring

### Double counting / extra pulses
- Increase debounce: change `delayed_on: 500ms` and `delayed_off: 500ms`
- Check if sensor mounting is secure — vibration can cause false triggers
- Inspect sensor distance — LJ12A3 has 4mm sensing range, mount accordingly

### WiFi not connecting
- Verify 2.4GHz network (5GHz not supported)
- Check antenna uFL connector is fully seated
- Connect to fallback AP "Water-Meter Fallback Hotspot" to verify device is alive
- Check logs via USB serial for connection errors

### Values lost after reboot
- All globals use `restore_value: true` so this should not happen
- If it does, check flash wear — after ~100,000 writes NVS can fail
- Try full erase and reflash: `esphome run --device PORT water-meter.yaml`

### Leak alert firing incorrectly
- Increase the **Leak Detection Threshold** (default 60 min) via HA
- Check if there is genuinely a slow leak (dripping tap, running toilet)
- Use **Reset Leak Alert** button to clear after investigating

---

## 📊 Technical Specifications

| Parameter | Value |
|-----------|-------|
| **Sensor Type** | LJ18A3-8-Z/BX-5V NPN NO proximity |
| **Sensor Thread** | M18×1 |
| **Sensing Range** | 8mm |
| **Sensor Voltage** | 5V |
| **Target Meter** | Elster V200 volumetric water meter |
| **Signal Logic** | 5V → 3.3V via voltage divider |
| **Pulse Detection** | Hardware interrupt (GPIO binary sensor) |
| **Debounce** | 500ms on / 500ms off |
| **Flow Rate Update** | Every 60 seconds |
| **Usage Stats Update** | Every 30 seconds |
| **WiFi Standard** | WiFi 6 (802.11ax) |
| **Antenna** | External uFL, ~80m range |
| **Flash Storage** | All counters survive reboot |
| **Web Server Port** | 80 (HTTP) |

---

## ⚖️ License

This project is provided as-is for personal and educational use.

```
⚠️  DISCLAIMER

This project involves water monitoring in your home.
Always verify readings against your physical meter.
The creator assumes NO LIABILITY for inaccurate readings,
water damage, or any other issues arising from use of this project.
```

---

**Version**: 1.0.0
**Last Updated**: 2026-02-22
**Firmware**: ESPHome 2024.x compatible
**Board**: Seeed XIAO ESP32C6

**Made with ❤️ and 💧**
