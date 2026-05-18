# rpm-fun

[![ESPHome](https://img.shields.io/badge/ESPHome-%E2%89%A5%202024.12.4-000000?logo=esphome&logoColor=white)](https://esphome.io/)
[![Platform](https://img.shields.io/badge/platform-ESP32--S3-E7352C?logo=espressif&logoColor=white)](https://www.espressif.com/en/products/socs/esp32-s3)
[![Framework](https://img.shields.io/badge/framework-Arduino-00979D?logo=arduino&logoColor=white)](https://github.com/espressif/arduino-esp32)
[![Home Assistant](https://img.shields.io/badge/Home%20Assistant-compatible-41BDF5?logo=home-assistant&logoColor=white)](https://www.home-assistant.io/)
[![License](https://img.shields.io/badge/license-MIT-green)](#license)
[![Made with YAML](https://img.shields.io/badge/made%20with-YAML-CB171E?logo=yaml&logoColor=white)](rpm-fun.yaml)

ESPHome firmware for the **ESP32-S3-DevKitC-1** that turns the board into a fan controller with **Home Assistant** integration:

- up to **6 tachometer (RPM) inputs** from 4-pin PC fans
- up to **4 PWM outputs** at **25 kHz** (Intel 4-Wire Fan Specification)
- **WS2812 status LED ring** showing Wi-Fi state and fan activity
- **Native API with encryption** for Home Assistant
- **OTA updates** over the network
- **Fallback Wi-Fi access point** + captive portal when the main SSID is unreachable
- persistent speed/brightness state (`restore_value: true`)
- SNTP time sync and a ready-to-uncomment OLED display block

---

## Table of Contents

- [Hardware Requirements](#hardware-requirements)
- [Pinout](#pinout)
- [Why this layout? (hardware constraints)](#why-this-layout-hardware-constraints)
- [Wiring a 4-pin PC fan](#wiring-a-4-pin-pc-fan)
- [Power budget](#power-budget)
- [Home Assistant entities](#home-assistant-entities)
- [LED indication](#led-indication)
- [Installation](#installation)
- [`secrets.yaml`](#secretsyaml)
- [OTA updates](#ota-updates)
- [Implementation notes](#implementation-notes)
- [Troubleshooting](#troubleshooting)
- [License](#license)

---

## Hardware Requirements

| Component | Requirement |
|---|---|
| MCU board | **ESP32-S3-DevKitC-1** (v1.0 / v1.1). Use a variant **without octal-PSRAM** (`N8`, `N16`, `N8R2`), see note below |
| ESPHome | **≥ 2024.12.4** (see `esphome.min_version`) |
| Framework | Arduino (required by `esp32_rmt_led_strip` and the `pulse_counter` legacy mode used here) |
| Fans | Standard **4-pin PWM PC fans** (Intel spec): GND / +12 V / Tach / PWM. Up to 6 tach, up to 4 PWM-controlled |
| Fan power | External **+12 V** PSU. The ESP32 is powered from USB-C on the DevKitC-1. **A common ground is mandatory** (see [Power budget](#power-budget) below) |
| Pull-ups | External **10 kΩ** pull-ups from each Tach line to +3.3 V on **fans 1–4** (see note below) |

### Why "no octal-PSRAM"

ESP32-S3 modules with the `R8` suffix (8 MB octal PSRAM) wire **GPIO33 – GPIO37** to the SPI PSRAM data bus (`SPIIO4 – SPIIO7` + `SPIDQS`), per the [ESP32-S3 Hardware Design Guidelines](https://docs.espressif.com/projects/esp-hardware-design-guidelines/en/latest/esp32s3/schematic-checklist.html). Those pins **cannot be used as GPIO** on R8 boards.

This firmware touches **three** pins inside that reserved range:

- **GPIO35** — PWM output for fan 3
- **GPIO36** — tachometer input for fan 6
- **GPIO37** — PWM output for fan 4

So on an `N16R8` / `N32R8` module the firmware will fail to boot or corrupt PSRAM access. Use a non-R8 variant (`N8`, `N16`, `N8R2` quad-PSRAM is fine), or remap fans 3, 4, and 6 to free pins.

---

## Pinout

### Tachometer inputs (RPM)

| Fan | GPIO | Method | Pull-up | HA entity |
|---|---|---|---|---|
| Fan 1 | GPIO5 | PCNT (hardware) | external 10 kΩ → 3.3 V | `Fan RPM 1 (GPIO5)` |
| Fan 2 | GPIO6 | PCNT | external 10 kΩ → 3.3 V | `Fan RPM 2 (GPIO6)` |
| Fan 3 | GPIO7 | PCNT | external 10 kΩ → 3.3 V | `Fan RPM 3 (GPIO7)` |
| Fan 4 | GPIO8 | PCNT | external 10 kΩ → 3.3 V | `Fan RPM 4 (GPIO8)` |
| Fan 5 | GPIO9 | software, 13 µs glitch filter | **internal** `INPUT_PULLUP` | `Fan RPM 5 (GPIO9)` |
| Fan 6 | GPIO36 | software, 13 µs glitch filter | **internal** `INPUT_PULLUP` | `Fan RPM 6 (GPIO36)` |

The tach output of a PC fan is **open-collector / open-drain**, so it always needs a pull-up. Fans 1–4 declare the pin with the short syntax (`pin: GPIO5`), which does not propagate `INPUT_PULLUP` into the PCNT peripheral — that is why an **external resistor (4.7 – 10 kΩ to 3.3 V)** is required. Fans 5 and 6 use the long pin syntax and rely on the MCU's internal pull-up.

### PWM outputs

| Fan | GPIO | Frequency | Driver | HA entity |
|---|---|---|---|---|
| Fan 1 | GPIO17 | 25 kHz | LEDC | `Fan Speed 1 (GPIO17)` |
| Fan 2 | GPIO18 | 25 kHz | LEDC | `Fan Speed 2 (GPIO18)` |
| Fan 3 | **GPIO35** | 25 kHz | LEDC | `Fan Speed 3 (GPIO35)` |
| Fan 4 | **GPIO37** | 25 kHz | LEDC | `Fan Speed 4 (GPIO37)` |

`min_power: 0.1` clamps the duty cycle floor to 10 % so the fan always spins reliably and avoids stall noise on cold start.

### Other peripherals

| Function | GPIO | Notes |
|---|---|---|
| WS2812 LED ring (3 LEDs) | GPIO48 | RBG order, `chipset: ws2812`, `rmt_channel: 2`, `internal: true` — not exposed to HA |
| I²C SDA | GPIO41 | 400 kHz |
| I²C SCL | GPIO40 | Reserved for the optional SSD1306 OLED (block is commented out in YAML) |

---

## Why this layout? (hardware constraints)

The asymmetric "**6 tach, only 4 PWM**" layout and the split between PCNT (fans 1–4) and software pulse counting (fans 5–6) is sometimes attributed to "the ESP32 can't handle that many fans." That is **not** the actual story — ESP32-S3 silicon could drive more. The real reasons are a stack of smaller constraints in the ESPHome layer and the standard PC use case.

### What the silicon actually supports

| Peripheral | ESP32-S3 silicon capacity | Implication for fan control |
|---|---|---|
| **LEDC (PWM)** | 8 channels, 4 timers, **low-speed mode only** ([ESP-IDF docs](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/peripherals/ledc.html)) | All channels can share a single 25 kHz timer → in theory **8 PWM fans** at once |
| **PCNT** | 8 units × 2 channels = **16 hardware channels** ([ESP-IDF docs](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/peripherals/pcnt.html)) | Plenty for 6 fan tachs |
| **RMT** | 4 TX + 4 RX channels | WS2812 strip consumes 1 TX, plenty left |

So the **chip itself is not the bottleneck**. What actually shapes the layout:

### 1. ESPHome wraps PCNT with an 8-channel cap

ESPHome's `pulse_counter` component documents an explicit limit: *"A maximum of 8 channels can be used"* when `use_pcnt: true` ([ESPHome docs](https://esphome.io/components/sensor/pulse_counter.html)). This is a wrapper-level constraint, not a silicon one. For 6 fans it is **not binding**, but it caps any future expansion at 8 channels per device.

### 2. PCNT's 13 µs glitch filter ceiling

From the same ESPHome docs: *"on the ESP32, when using the hardware pulse counter \[`internal_filter`\] can not be higher than 13 µs … for the ESP8266 or with `use_pcnt: false` you can use larger intervals too."* If you want a heavier debounce (for long, noisy fan cables — common in 19" racks or 1U sleds), you must fall back to software counting.

### 3. `INPUT_PULLUP` vs PCNT pin syntax

PC-fan tachs are open-collector and need a pull-up. ESPHome's `pulse_counter` accepts two pin syntaxes:

- **short** — `pin: GPIO5` — used by fans 1–4. Concise but does not carry a `mode:` (no internal pull-up), so each line needs an **external** 4.7 – 10 kΩ resistor on the board.
- **expanded** — `pin: { number: GPIO9, mode: INPUT_PULLUP }` — used by fans 5–6. Cleanest way to enable the MCU's internal pull-up. In practice this pairs more reliably with `use_pcnt: false`, hence the software-counter fallback on those two channels.

This is the **specific reason** fans 5 and 6 dropped off PCNT: it was not a unit-count exhaustion, it was a deliberate choice to get an internal pull-up on those signals without adding more discrete components to the board.

### 4. LEDC channels vs the 4-fan PWM choice

LEDC has 8 channels on ESP32-S3, all able to share one 25 kHz timer, so **6 PWM outputs are perfectly feasible**. The reason this firmware exposes only 4 is the **typical case-fan use case**: enthusiast and SFF PC builds usually have 3–4 actively controlled case/radiator fans plus 2 "report-only" signals (CPU-block pump tach, PSU fan tach, or a non-PWM Molex fan). Putting the dial on 4 controlled + 6 monitored matches that pattern.

### TL;DR

| You might think… | …but the actual reason is |
|---|---|
| "The chip can't drive 6 PWM fans" | LEDC can do 8 at 25 kHz on a shared timer — the 4-PWM cap is a **design choice for typical PC fan setups**, not silicon |
| "PCNT ran out for fans 5–6" | PCNT has 16 channels; the **8-channel ESPHome cap** is the real ceiling, and even that isn't hit at 6 fans |
| "Software counting was a forced fallback" | It was a **pragmatic choice for clean internal pull-ups** on the open-collector tach lines (and a free filter > 13 µs option for noisy installations) |

If you want to extend to 6 PWM outputs, add two more `ledc` outputs on free GPIOs (e.g. 15, 16) and two more `number.template` entities driving them — no chip changes required.

---

## Wiring a 4-pin PC fan

Standard 4-pin PC-fan connector (looking into the fan-side socket):

```
        ┌─── Pin 1 GND       →  GND  (tied to both board GND and PSU GND)
        │
        ├─── Pin 2 +12V      →  +12 V from external PSU
        │
        ├─── Pin 3 TACH      →  ESP32 GPIO (RPM input) + pull-up to 3.3 V
        │
        └─── Pin 4 PWM       →  ESP32 GPIO (PWM output, 25 kHz)
```

**Important:**

- A **common ground** between the ESP32 and the 12 V PSU is mandatory, otherwise the tach signal is just noise.
- The ESP32 drives PWM at **3.3 V**. The Intel 4-Wire Fan Spec allows 0 – 5.25 V on the PWM input with a switching threshold around 1.8 V, so 3.3 V is a valid logic-high. The fan internally pulls up the PWM input, no external resistor needed on that line.
- The tach line emits **two pulses per revolution** — that is why the filter is `rpm = x / 2.0`.

---

## Power budget

The ESP32 is fed from USB (≤ 500 mA at 5 V is plenty for the SoC + LED ring). The **fans run off the external 12 V PSU**, which you size yourself.

Rough current draw per fan at 12 V:

| Fan class | Typical running current | Inrush at start |
|---|---|---|
| 120 mm case fan (e.g. Arctic P12) | 80 – 150 mA | up to 2× running |
| 140 mm case fan (Noctua NF-A14) | 80 – 200 mA | up to 2× running |
| 120 mm high-static-pressure (server / radiator) | 200 – 500 mA | up to 3× running |
| 40/60 mm 1U server fan | 200 – 800 mA | up to 3× running |

**Budget rule of thumb:** size the 12 V PSU for the **sum of inrush currents** of all fans you might spin up simultaneously. A 6-fan build of regular case fans is comfortably served by a **12 V / 2 A (24 W)** brick. A build with high-RPM radiator or server fans wants **12 V / 4 – 5 A** to survive simultaneous cold starts without a brownout.

Always tie the **PSU ground to the ESP32 ground** — otherwise the tach signal is just floating noise.

---

## Home Assistant entities

Exposed via the native API:

### Sensors

| Entity | Description |
|---|---|
| `sensor.fan_rpm_1` … `fan_rpm_6` | Fan speed in RPM |
| `sensor.wifi_signal` | Wi-Fi strength as a **percentage** (custom mapping `(RSSI + 100) × 2`, clamped 0 – 100) |

### Numbers (template, `restore_value: true`)

| Entity | Range | Step | Default | Purpose |
|---|---|---|---|---|
| `number.led_brightness` | 0.0 – 1.0 | 0.01 | 0.2 | LED ring brightness |
| `number.rpm_update_interval` | 1 – 60 s | 1 | 30 | Tach polling interval; restarts `rpm_update_loop` on change |
| `number.fan_speed_1` … `fan_speed_4` | 0 – 100 % | 1 | 50 | PWM duty cycle of the corresponding fan |

### Switch

| Entity | Action |
|---|---|
| `switch.restart_rpm_fun` | Soft reboot of the device |

---

## LED indication

The WS2812 ring of 3 LEDs is declared `internal: true` — Home Assistant does not see it. It is a **local status indicator** only:

| State | Colour / behaviour |
|---|---|
| Wi-Fi connected, fans idle | **Green**, steady |
| Wi-Fi disconnected, fans idle | **Magenta**, blinking 500 ms on / 500 ms off |
| Any fan spinning (RPM > 0) | **Yellow**, blinking 200 ms on / 200 ms off (interleaved with the connection-state colour — see note below) |
| Just (re)connected to Wi-Fi | Brief green + `init_pwm` re-applies saved PWM values |

> **Note on interleaving.** Two `interval:` blocks run in parallel: the **1 s block** sets the colour from Wi-Fi state, the **500 ms block** triggers the yellow fan-activity blink. When fans are spinning and Wi-Fi is connected, the visible result is **mostly yellow flashes with brief green flashes every second**, not pure yellow. This is by design — both signals stay observable at the same time.

Brightness is controlled from HA via `number.led_brightness`.

---

## Installation

### 1. Clone

```bash
git clone https://github.com/ilia-ae/rpm-fun_esphome.git
cd rpm-fun_esphome
```

### 2. Install ESPHome

```bash
# pip
pip install esphome

# or Docker
docker pull ghcr.io/esphome/esphome
```

### 3. Create `secrets.yaml`

Place it next to `rpm-fun.yaml` (template below).

### 4. Compile and flash

First time over USB:

```bash
esphome run rpm-fun.yaml
# or, with Docker:
docker run --rm -v "${PWD}":/config --device=/dev/ttyACM0 -it ghcr.io/esphome/esphome run rpm-fun.yaml
```

Subsequent updates **over the air**:

```bash
esphome upload rpm-fun.yaml --device rpm-fun.local
```

### 5. Add to Home Assistant

The device announces itself via mDNS as `rpm-fun.local`. Confirm the integration in **Settings → Devices & Services → ESPHome** and paste the `api_ha` key from `secrets.yaml`.

---

## `secrets.yaml`

Template (create this file next to `rpm-fun.yaml`):

```yaml
wifi_ssid: "YourWiFiSSID"
wifi_password: "YourWiFiPassword"

# 32-byte base64 key for the native API encryption
# Generate: `openssl rand -base64 32`
api_ha: "PUT_32BYTE_BASE64_KEY_HERE="

# Password for OTA updates
ota_key: "your-strong-ota-password"
```

> A template lives at [`secrets.yaml.example`](secrets.yaml.example) — `cp secrets.yaml.example secrets.yaml` and fill it in. `secrets.yaml` is already in [`.gitignore`](.gitignore); **never commit it**.

### Fallback hotspot

If the main Wi-Fi is unreachable at boot or drops out, the device exposes an open AP for recovery:

| Field | Value |
|---|---|
| SSID | `Rpm-Fun Fallback Hotspot` |
| Password | `uStrxuFDPIOY` (hard-coded in YAML — **change it** if you deploy beyond a trusted network) |
| Captive-portal URL | opens automatically; manual fallback: <http://192.168.4.1> |

Connect any phone/laptop to the SSID, accept the captive-portal prompt (or open <http://192.168.4.1> directly), and you get a web form to set new Wi-Fi credentials without re-flashing.

---

## OTA updates

ESPHome native OTA platform. Password is taken from `!secret ota_key`. To push an update:

```bash
esphome upload rpm-fun.yaml --device rpm-fun.local
```

ESPHome will open a TCP session to the device (default port 3232) and stream the new binary.

---

## Implementation notes

### Why fans 5 – 6 are on software counting

See the [Why this layout?](#why-this-layout-hardware-constraints) section above for the full story. Short version: this is **not** a PCNT capacity issue; it is a deliberate move to enable the MCU's internal `INPUT_PULLUP` on those two open-collector tach lines via the expanded pin syntax. If you want to switch them back to PCNT, change `use_pcnt: false` → `true`, drop the `mode: INPUT_PULLUP` from the pin block, add an external 4.7 – 10 kΩ pull-up to 3.3 V on the board, and remove `internal_filter` (it behaves differently under PCNT).

### Why `init_pwm` fires on `on_connect`

After a reset or brown-out, `restore_value: true` restores the target percentages in `number.fan_speed_*`, but ESPHome **does not** auto-call `set_action` on restore. The `init_pwm` script runs from `wifi.on_connect`, syncing the actual LEDC outputs back to the saved values. Without it, the fans would sit at LEDC's default duty until the user nudges the slider in HA.

### The RPM polling loop

`pulse_counter` with `update_interval: never` means **"never auto-publish"**. Polling is driven explicitly by the `rpm_update_loop` script (`mode: restart`), which runs `component.update` on all six sensors in sequence with a delay of `rpm_update_interval × 1000 ms`. Changing the interval in HA restarts the loop via `on_value: script.execute rpm_update_loop`.

`mode: restart` matters: when the interval changes, the old loop is aborted and a new one starts with the new delay — no overlapping concurrent loops.

### Wi-Fi → percent

ESPHome reports Wi-Fi signal in dBm by default. The lambda `quality = (RSSI + 100) × 2`, clamped to 0 – 100, gives a dashboard-friendly percentage (−50 dBm → 100 %, −100 dBm → 0 %).

---

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| Tach reads 0 on fans 1 – 4 | Missing external 4.7 – 10 kΩ pull-up from Tach to 3.3 V. Add the resistor. |
| RPM jitters ± 50 at a steady speed | Filter too short or no common ground between PSU and ESP. Tie the grounds. |
| Fan refuses to start, chirps | `min_power` too low, or the fan's PWM input is fussy about 3.3 V. Try raising `min_power` to `0.2`. |
| Firmware hangs at boot | Module is `R8` (octal PSRAM); GPIO33 – 37 are reserved for PSRAM, and this firmware uses GPIO35 (PWM 3), GPIO36 (tach 6) and GPIO37 (PWM 4). Use a non-R8 variant or remap fans 3, 4, and 6. |
| LED ring shows the wrong colour | Incorrect `rgb_order`. Try swapping `RBG` ↔ `GRB` ↔ `RGB` in the `light` block. |
| Device not visible in HA | The `api.encryption.key` in `secrets.yaml` must be valid base64 of 32 bytes (`openssl rand -base64 32`). |
| OTA fails | Check free space (≥ 1 MB on the ESP32-S3 `ota` partition). |

---

## Extensions

The YAML contains a commented SSD1306 72×40 OLED block (I²C, address `0x3C`, on SDA = 41 / SCL = 40). Uncomment the `display:` block (the matching `font:` block is already present) to show six fan RPMs, SNTP time, and Wi-Fi quality on the screen.

---

## License

MIT.

---

## References

- [ESP-IDF — LED Control (LEDC) on ESP32-S3](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/peripherals/ledc.html)
- [ESP-IDF — Pulse Counter (PCNT) on ESP32-S3](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/peripherals/pcnt.html)
- [ESPHome — `pulse_counter` sensor](https://esphome.io/components/sensor/pulse_counter.html)
- [ESPHome — `ledc` output](https://esphome.io/components/output/ledc/)
- [Intel 4-Wire PWM Fan Specification](https://www.formfactors.org/) (background on the 25 kHz target and the open-collector tach signal)
