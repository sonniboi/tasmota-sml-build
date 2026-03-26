# tasmota-sml-build

Tasmota firmware builds with SML support for 1MB ESP8266 devices (Sonoff Basic, ESP-01, Hichi IR reader).

This is a continuation of [darth-hp/tasmota-hichi-sml-firmware](https://github.com/darth-hp/tasmota-hichi-sml-firmware), which is no longer maintained.

## What is this?

[Tasmota](https://tasmota.github.io) is an open-source firmware for ESP8266/ESP32 devices. This repo builds a custom variant with **SML (Smart Message Language)** support enabled, which is required to read data from smart electricity meters via an IR optical head.

The standard Tasmota builds do not include SML support. This build adds:
- `USE_SCRIPT` — Tasmota scripting engine
- `USE_SML_M` — SML parser for smart meters

## Compatible Hardware

- Sonoff Basic (1MB flash)
- ESP-01 / ESP-01S (1MB flash)
- Any ESP8266 with 1MB flash and IR optical head (e.g. Hichi Smartmeter reader)

## Installation

### First-time install
Flash via serial using [Tasmota Web Installer](https://tasmota.github.io/install/) or esptool.

### OTA Update (existing Tasmota device)

> **Important for 1MB devices:** Web UI file upload fails with "Not enough space".
> Use the OTA URL method instead — the device downloads directly from GitHub.

In the Tasmota console or via HTTP API:

```
OtaUrl https://github.com/sonniboi/tasmota-sml-build/releases/latest/download/tasmota-sml.bin.gz
Upgrade 1
```

Or for a specific version:
```
OtaUrl https://github.com/sonniboi/tasmota-sml-build/releases/download/v15.3.0/tasmota-sml.bin.gz
Upgrade 1
```

## Building

Triggered manually via [GitHub Actions](.github/workflows/build.yml):

1. Go to **Actions** → `Build Tasmota SML (1MB)` → **Run workflow**
2. Enter the desired Tasmota tag (e.g. `v15.3.0`)
3. A GitHub Release with `tasmota-sml.bin.gz` is created automatically

## SML Script Example

After flashing, configure the SML reader in Tasmota under **Tools → Script**:

```
>D
>B
->sensor53 r
>M 1
+1,3,o,0,300,Energy,1,30,2F3F210D0A,063030300D0A
1,1.8.0*00(@1),Input,KWh,Total,2
#
```

This reads OBIS 1.8.0 (total energy consumption in kWh) from the meter.
See [Tasmota SML documentation](https://tasmota.github.io/docs/Smart-Meter-Interface/) for more OBIS codes.

## Credits

- [arendst/Tasmota](https://github.com/arendst/Tasmota) — the Tasmota firmware
- [darth-hp/tasmota-hichi-sml-firmware](https://github.com/darth-hp/tasmota-hichi-sml-firmware) — original idea and build config
