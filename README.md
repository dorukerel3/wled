# WLED-Zero-S3

A complete, self contained hardware and firmware project for a custom addressable LED lighting system: a custom **ESP32-S3 controller motherboard**, three **WS2812B light stick PCBs** that daisy chain together, and a tailored build of **[WLED](https://github.com/wled/WLED) v16.0.1** configured for the exact hardware. This repository contains everything needed to reproduce the system from end to end: Gerber and drill files, pick and place data, bills of materials, 3D STEP models, the full patched firmware source tree, and a prebuilt single file flash image.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Hardware Gallery](#hardware-gallery)
- [System Architecture](#system-architecture)
- [Key Features](#key-features)
- [Motherboard](#motherboard)
- [Light Sticks](#light-sticks)
- [Firmware Customizations](#firmware-customizations)
- [Repository Contents](#repository-contents)
- [Fabrication and Assembly Files](#fabrication-and-assembly-files)
- [Flashing the Prebuilt Release](#flashing-the-prebuilt-release)
- [First Boot](#first-boot)
- [Building the Firmware from Source](#building-the-firmware-from-source)
- [Author and License](#author-and-license)

---

## Project Overview

The system consists of a controller motherboard and three purpose built LED PCBs ("light sticks") of graduated lengths. The motherboard hosts a bare **ESP32-S3-WROOM-1-N16R8** module, USB Type C power and programming, Li ion battery charging, 3.3 V regulation, a 5 V logic level shifter for the LED data line, and a MOSFET high side power switch that lets the firmware cut power to the LED chain entirely.

The three light sticks carry **11**, **6**, and **4** WS2812B LEDs respectively, **21 in total**, which is exactly what the firmware is compiled for (`PIXEL_COUNTS=21`). The firmware is a minimal, targeted fork of WLED `v16.0.1`: one new build environment and a single line source patch that makes the boot power state a compile time default, so a freshly flashed unit ships with its LED power stage off and correct switch polarity, needing no configuration after flashing.

---

## Hardware Gallery

The physical production results of the custom hardware, motherboard and light sticks:

### Motherboard
<img src="images/motherboard_front.jpg" width="400" alt="Motherboard Front"> <img src="images/motherboard_back.jpg" width="400" alt="Motherboard Back">

### Longest Light Stick
<img src="images/led_strip_longest_front.jpg" width="400" alt="Longest Light Stick Front"> <img src="images/led_strip_longest_back.jpg" width="400" alt="Longest Light Stick Back">

### Middle Light Stick
<img src="images/led_strip_middle_front.jpg" width="400" alt="Middle Light Stick Front"> <img src="images/led_strip_middle_back.jpg" width="400" alt="Middle Light Stick Back">

### Shortest Light Stick
<img src="images/led_strip_shortest_front.jpg" width="400" alt="Shortest Light Stick Front"> <img src="images/led_strip_shortest_back.jpg" width="400" alt="Shortest Light Stick Back">

---

## System Architecture

```text
 USB Type C (GCT USB4220) ──► TP4057 Li ion charger ──► Battery (B+/B− pads)
        │                                              │
        └────────────► AP2112K-3.3 LDOs ──► 3.3 V rail ──► ESP32-S3-WROOM-1-N16R8
                                                               │           │
                                            GPIO1 (LED data) ──┘           └── GPIO2 (power switch)
                                                   │                              │
                                       74AHCT1G125 level shifter        NMOS + PMOS high side switch
                                                   │                              │
                                                   ▼                              ▼
                                     DIN ► Longest stick (11× WS2812B) ► Middle stick (6×) ► Shortest stick (4×)
                                                        21 LEDs total, switched 5 V rail
```

- **Data path:** the ESP32-S3 drives the WS2812B chain from `GPIO1` through a **74AHCT1G125** buffer, shifting the 3.3 V signal to clean 5 V logic for reliable data at full supply voltage.
- **Power path:** `GPIO2` controls an **NMOS plus PMOS MOSFET high side switch** (active high) so the entire LED rail can be powered down in software, giving true zero standby draw on the strip rather than just black pixels.
- **Chaining:** each stick exposes `5V`, `GND`, `DIN`/`DOUT` pads, so the three boards cascade into a single logical strip of 21 pixels.

---

## Key Features

- **Boot state off by default**, via a new `WLED_TURN_ON_AT_BOOT` compile time flag patched into WLED's source (upstream hardcodes it to stay on).
- **Correct power switch polarity out of the box** (`RLYPIN=2`, `RLYMDE=1`), matched to the board's active high MOSFET high side switch topology.
- **Single merged flashable image**: bootloader, partition table, and application combined into one `.bin` written at offset `0x0`, so flashing needs only a single offset.
- **16 MB flash / 8 MB octal PSRAM support**, targeting the ESP32-S3-WROOM-1-N16R8 module specifically.
- **Complete fabrication package** for all four PCBs: Gerbers, drill files, pick and place, BOM, and 3D STEP models.
- **Built on a stable upstream release** (`v16.0.1`), with the full source tree preserved inside the repository for reproducible builds and future upstream syncs.

---

## Motherboard

| Component | Part | Package | Role |
|-----------|------|---------|------|
| U2 | ESP32-S3-WROOM-1 (N16R8) | RF module | MCU: 16 MB flash, 8 MB octal SPI PSRAM, WiFi/BLE |
| J3 | GCT USB4220-03-1040-C | USB Type C receptacle | Power input and USB programming |
| TP4057 | TP4057 | TSOT-23-6 | Single cell Li ion charge management |
| AP2112_1, AP2112_2 | AP2112K-3.3 | SOT-23-5 | 3.3 V LDO regulation |
| U4 | 74AHCT1G125 | SOT-23-5 | 3.3 V → 5 V LED data level shifting |
| NMOSS1 | N channel MOSFET | SOT-23 | Gate driver stage for the high side switch |
| PMOSS1 | P channel MOSFET | SOT-23 | High side power switch for the LED rail |
| — | 8× resistors (2×1 kΩ, 2×5.1 kΩ, 3×10 kΩ, 1×330 Ω) | 0805 | Gate drive, CC1/CC2 termination, pull ups, charge programming |
| — | 2× 1 µF capacitors | 0805 | LDO input/output decoupling |
| J2 | 1×4 pin header (DNP) | 2.54 mm | Auxiliary/debug header |
| — | 5× test points (`5V`, `B+`, `B−`, `DIN`, `GND`) | 3.0 mm pads | Board bringup and battery/strip wiring |

Full references, footprints, and datasheet links are in [`Motherboard_Files/Motherboard.csv`](Motherboard_Files/Motherboard.csv); exact placement coordinates in [`Motherboard_Files/Motherboard-all-pos.csv`](Motherboard_Files/Motherboard-all-pos.csv).

### Firmware pin map

| Signal | Pin | Configuration |
|--------|-----|---------------|
| LED data out | GPIO1 | `DATA_PINS=1` |
| LED power switch | GPIO2 | `RLYPIN=2`, `RLYMDE=1` (active high) |
| LED count | — | `PIXEL_COUNTS=21` |
| Flash | — | 16 MB, DIO, 40 MHz |

---

## Light Sticks

Three variants of the same design language: a 5 V WS2812B chain with 0.1 µF decoupling at each LED and pad based chaining.

| Board | WS2812B count | Decoupling caps | Pads |
|-------|---------------|-----------------|------|
| Longest | 11 | 7× 0.1 µF | `5V`, `DIN`, `GND`, + |
| Middle | 6 | 3× 0.1 µF | `5V`, `DOUT`, `GND`, TP1–TP3 |
| Shortest | 4 | 2× 0.1 µF | `5V`, `DIN`, `DOUT`, `GND`, `VDD`, `VSS` |

**Total: 21 LEDs**, matching the firmware's compiled pixel count exactly. All LEDs use the `WS2812B PLCC4 5.0×5.0 mm` footprint on a 3.2 mm pitch.

---

## Firmware Customizations

Two changes on top of a clean `v16.0.1` checkout, both preserved in [`Motherboard_Files/firmware/`](Motherboard_Files/firmware/):

**[`platformio_override.ini`](Motherboard_Files/firmware/platformio_override.ini)** (new file) defines the `esp32s3` build environment:

```ini
[env:esp32s3]
extends = env:esp32s3dev_16MB_opi
build_flags = ${env:esp32s3dev_16MB_opi.build_flags}
  -D WLED_RELEASE_NAME=\"ESP32S3_Custom\"
  -D DATA_PINS=1
  -D PIXEL_COUNTS=21
  -D RLYPIN=2
  -D RLYMDE=1
  -D WLED_TURN_ON_AT_BOOT=false
```

**[`wled00/wled.h`](Motherboard_Files/firmware/wled00/wled.h)** carries a single line patch exposing `turnOnAtBoot` as a build flag, since upstream WLED hardcodes it:

```diff
// LED CONFIG
-WLED_GLOBAL bool turnOnAtBoot _INIT(true);                 // turn on LEDs at power up
+#ifndef WLED_TURN_ON_AT_BOOT
+  #define WLED_TURN_ON_AT_BOOT true
+#endif
+WLED_GLOBAL bool turnOnAtBoot _INIT(WLED_TURN_ON_AT_BOOT); // turn on LEDs at power up
```

This build sets the flag to `false`, so the LED power stage never energizes without explicit user action.

---

## Repository Contents

```text
.
├── Motherboard_Files/
│   ├── firmware/                                        Full WLED v16.0.1 source tree (patched: wled00/wled.h,
│   │                                                    new: platformio_override.ini)
│   ├── WLED_16.0.1_ESP32-S3_16MB_opi_MERGED_dio40m_v2.bin   Prebuilt merged flash image (write at 0x0)
│   ├── Motherboard_Gerber&Drill.zip                     Fabrication files (Cu, mask, paste, silkscreen, edge cuts, PTH/NPTH drill)
│   ├── Motherboard.csv                                  Bill of materials
│   ├── Motherboard-all-pos.csv                          Pick and place component positions
│   └── Motherboard_Design.step                          3D model
├── Longest_Light_Stick_Files/                           11× WS2812B, pads: 5V, DIN, GND
│   ├── Longest_light_stick_Gerber&Drill.zip             Fabrication files (Cu, mask, paste, silkscreen, edge cuts, PTH/NPTH drill)
│   ├── Longest_light_stick.csv                          Bill of materials (11× WS2812B, 7× 0.1 µF 0805)
│   ├── Longest_light_stick-all-pos.csv                  Pick and place component positions
│   └── Longest_Light_Stick_Design.step                  3D model
├── Middle_Light_Stick_Files/                            6× WS2812B, pads: 5V, DOUT, GND, TP1–TP3
│   ├── Middle_light_stick_Gerber&Drill.zip              Fabrication files (Cu, mask, paste, silkscreen, edge cuts, PTH/NPTH drill)
│   ├── Middle_light_stick.csv                           Bill of materials (6× WS2812B, 3× 0.1 µF 0805)
│   ├── Middle_light_stick-all-pos.csv                   Pick and place component positions
│   └── Middle_Light_Stick_Design.STEP                   3D model
├── Shortest_Light_Stick_Files/                          4× WS2812B, pads: 5V, DIN, DOUT, GND, VDD, VSS
│   ├── Shortest_light_stick_Gerber&Drill.zip            Fabrication files (Cu, mask, paste, silkscreen, edge cuts, PTH/NPTH drill)
│   ├── Shortest_light_stick.csv                         Bill of materials (4× WS2812B, 2× 0.1 µF 0805)
│   ├── Shortest_light_stick-all-pos.csv                 Pick and place component positions
│   └── Shortest_Light_Stick_Design.STEP                 3D model
├── images/                                              README gallery assets
│   ├── motherboard_front.jpg                            Assembled motherboard, component side
│   ├── motherboard_back.jpg                             Assembled motherboard, reverse side
│   ├── led_strip_longest_front.jpg                      Longest light stick, LED side
│   ├── led_strip_longest_back.jpg                       Longest light stick, reverse side
│   ├── led_strip_middle_front.jpg                       Middle light stick, LED side
│   ├── led_strip_middle_back.jpg                        Middle light stick, reverse side
│   ├── led_strip_shortest_front.jpg                     Shortest light stick, LED side
│   └── led_strip_shortest_back.jpg                      Shortest light stick, reverse side
├── LICENSE                                              MIT license
└── README.md                                            This document
```

---

## Fabrication and Assembly Files

Each of the four PCB folders is a complete manufacturing package:

- **`*_Gerber&Drill.zip`**: full Gerber set (front/back copper, solder mask, paste, silkscreen, board outline) plus PTH and NPTH Excellon drill files, ready to upload to any fab (JLCPCB, PCBWay, etc.).
- **`*.csv`**: the bill of materials with references, quantities, footprints, and datasheet links.
- **`*-all-pos.csv`**: pick and place position files (X/Y, rotation, side) for assembly services.
- **`*.step`**: 3D STEP models for enclosure design and mechanical verification.

---

## Flashing the Prebuilt Release

1. Download [`Motherboard_Files/WLED_16.0.1_ESP32-S3_16MB_opi_MERGED_dio40m_v2.bin`](Motherboard_Files/WLED_16.0.1_ESP32-S3_16MB_opi_MERGED_dio40m_v2.bin).
2. Install [esptool](https://github.com/espressif/esptool):
   ```bash
   python -m pip install esptool
   ```
3. Put the board into download mode if it doesn't reset into it automatically: hold **BOOT**, tap **RESET**, release **BOOT**.
4. Erase and flash (replace `COMx` with the board's serial port):
   ```bash
   python -m esptool --chip esp32s3 --port COMx erase_flash
   python -m esptool --chip esp32s3 --port COMx --baud 921600 write_flash 0x0 WLED_16.0.1_ESP32-S3_16MB_opi_MERGED_dio40m_v2.bin
   ```
   The image is a single merge of bootloader, partition table, and application, so it is written at offset `0x0` with no further offsets needed.

---

## First Boot

1. The board opens a WiFi access point named `WLED-AP`, password `wled1234`.
2. Connect to it and browse to `4.3.2.1` to enter your home WiFi credentials.
3. Once joined, reach the UI at `http://wled-XXXXXX.local`. The `XXXXXX` is the last 6 hex digits of the board's MAC address, visible in the esptool output during flashing, or via your router's DHCP client list if `.local` does not resolve.
4. LEDs and the power switch stay off on every boot by default, so no configuration is required. Turn them on from the UI when ready.

---

<p align="center">
  <img src="images/wled_logo_akemi.png" width="220" alt="WLED Akemi logo">
</p>

## Building the Firmware from Source

```bash
git clone https://github.com/dorukerel3/WLED-Zero-S3.git
cd WLED-Zero-S3/Motherboard_Files/firmware
python -m pip install platformio
python -m platformio run -e esp32s3
```

This produces `.pio/build/esp32s3/firmware.bin` (application only). To reproduce the merged single file release image:

```bash
python -m esptool --chip esp32s3 merge_bin \
  -o WLED_16.0.1_ESP32-S3_16MB_opi_MERGED_dio40m_v2.bin \
  --flash_mode dio --flash_freq 40m --flash_size 16MB \
  0x0     .pio/build/esp32s3/bootloader.bin \
  0x8000  .pio/build/esp32s3/partitions.bin \
  0x10000 .pio/build/esp32s3/firmware.bin
```

---

## Author and License

Designed and developed by **Doruk Erel**, [dorukerel.com](https://dorukerel.com).

Firmware based on [WLED](https://github.com/wled/WLED). Released under the **MIT License**. See the [`LICENSE`](LICENSE) file for the full text.
