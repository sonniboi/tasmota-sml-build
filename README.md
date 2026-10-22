# tasmota-sml-build

Tasmota firmware builds with **SML support** for **1 MB ESP8266** devices (Hichi IR reader,
Sonoff Basic, ESP-01).

Continuation of [darth-hp/tasmota-hichi-sml-firmware](https://github.com/darth-hp/tasmota-hichi-sml-firmware),
which is no longer maintained.

## Why this repo exists

[Tasmota](https://tasmota.github.io) is an open-source firmware for ESP8266/ESP32. The official
builds **do not include SML support**, which is what you need to read a smart electricity meter
through an IR optical head. This repo builds a variant with:

- `USE_SCRIPT` — Tasmota scripting engine
- `USE_SML_M` — Smart Message Language parser

`USE_RULES` is disabled to make room, because a 1 MB device has very little of it.

## Compatible hardware

- Hichi / generic IR optical head with ESP8266, 1 MB flash
- Sonoff Basic (1 MB), ESP-01 / ESP-01S (1 MB)

Note that such a device often reports itself as "Sonoff Basic" on the info page — that is just the
module template, not the actual hardware.

## Releases

Each release contains **both** images, and you need both for an OTA on a 1 MB device:

| Asset | Purpose |
|---|---|
| `tasmota-sml.bin.gz` | the actual firmware (SML + scripting) |
| `tasmota-minimal.bin.gz` | intermediate step for the two-stage OTA, built **without** the override |
| `tasmota-sml.bin` | uncompressed, for serial flashing |

## Installation

### First-time install (serial)

Flash `tasmota-sml.bin` with [Tasmota Web Installer](https://tasmota.github.io/install/) or esptool.

### OTA update — read this before you try it

On a 1 MB device an OTA is a **two-stage** process, and two things fail silently if you get it
wrong. Both were verified on a real device.

**1. A Web UI file upload does not work.** It fails with *"Not enough space"*. The device has to
download the firmware itself, via `OtaUrl`.

**2. Downloading straight from GitHub over HTTPS does not work either.** The ESP8266 has roughly
18 kB of free heap, which is not enough for the TLS handshake; the transfer dies with
`{"Upgrade":"Failed HTTP error: connection lost"}`. **Serve the files over plain HTTP from your own
LAN instead.**

**3. Both files must sit next to each other.** The full image (~500 kB compressed) does not fit
into the free flash (~292 kB). Tasmota detects this on its own and then requests
`tasmota-minimal.bin.gz` **from the same directory**, flashes it, reboots, and pulls the full image
afterwards. If that file is missing you get `{"Upgrade":"Failed File Not Found (404)"}` and the
device stays on the old firmware.

```bash
# on a machine in the same LAN
mkdir otasrv && cd otasrv
V=v16.0.0
curl -sL -O https://github.com/sonniboi/tasmota-sml-build/releases/download/$V/tasmota-sml.bin.gz
curl -sL -O https://github.com/sonniboi/tasmota-sml-build/releases/download/$V/tasmota-minimal.bin.gz
python3 -m http.server 9876 --bind 0.0.0.0

# then, against the device (replace both IPs)
curl "http://<device-ip>/cm?cmnd=OtaUrl%20http%3A%2F%2F<server-ip>%3A9876%2Ftasmota-sml.bin.gz"
curl "http://<device-ip>/cm?cmnd=Upgrade%201"
```

The whole thing takes about 90 seconds and involves two reboots. **The device delivers no meter
readings in between**, so do not start an OTA at a moment when you need the reading.

Verify afterwards:

```bash
curl -s "http://<device-ip>/cm?cmnd=Status%202"   | jq .StatusFWR          # version
curl -s "http://<device-ip>/cm?cmnd=Status%204"   | jq .StatusMEM.Sensors  # must contain 53
curl -s "http://<device-ip>/cm?cmnd=Script%201"                            # must be ON
curl -s "http://<device-ip>/cm?cmnd=Sensor53%20d3"                         # raw dump on
curl -s "http://<device-ip>/cs?c2=0"                                       # watch OBIS arrive
curl -s "http://<device-ip>/cm?cmnd=Sensor53%20d0"                         # raw dump off
```

**Your script survives the update** — settings are kept across an OTA. It is only lost on a config
reset or a serial flash with erased config, so back it up first: `curl -s http://<device-ip>/s10`.

### Two small traps in the HTTP API

- The script editor lives at **`/s10`**, not at `/ays`. The latter returns 404 and makes it look
  like scripting is missing. In the UI: Main Menu → Tools → Edit Script.
- `cmnd=Script` **without a parameter** answers `{"Command":"Error"}`. Use `Script 1`. Same for
  `Sensor53` → `Sensor53 d`.

### Since Tasmota 15.4.0.2: the referer check

`SetOption128` now defaults to **OFF**, which means the referer check is active and requests
**without** a referer header are rejected too (log: `Referer '' denied`). This affects `/cm`, `/dl`,
`/cs` and the config pages — not the start page. Either set `SetOption128 ON`, or send a header
with every call: `curl -H 'Referer: http://<device-ip>/' …`.

## Building

Manual trigger via [GitHub Actions](.github/workflows/build.yml):

1. **Actions** → `Build Tasmota SML (1MB)` → **Run workflow**
2. Enter an upstream Tasmota tag, e.g. `v16.0.0`
3. A release of the same name is created with all assets attached

The workflow checks out upstream at that tag, copies `user_config_override.h` over it and runs
`pio run -e tasmota`. The minimal image is built in a second pass **without** the override — with
`USE_SCRIPT`/`USE_SML_M` it would be too large to serve its purpose.

Binaries are also uploaded as a workflow artifact (30 days), so a failure in the release step does
not throw away the build.

## Configuring the meter

After flashing, set up the reader under **Tools → Edit Script** (`/s10`). Which script you need
depends on what your meter speaks — and these are two genuinely different protocols:

**SML** (most modern meters, binary, usually 9600 baud):

```
>D
>B
->sensor53 r
>M 1
+1,3,s,0,9600,Power
1,77070100010800ff@1000,Total consumption,kWh,Total,4
1,77070100020800ff@1000,Total feed-in,kWh,Export,4
1,77070100100700ff@1,Current power,W,Power,0
#
```

**OBIS ASCII / IEC 62056-21** (older meters, plain text, 300 baud, needs a wake-up sequence):

```
>D
>B
->sensor53 r
>M 1
+1,3,o,0,300,Energy,1,30,2F3F210D0A,063030300D0A
1,1.8.0*00(@1),Input,KWh,Total,2
#
```

The `o` vs `s` in the descriptor line is what selects the protocol. Many meters only reveal their
full data set — decimals and instantaneous power — once the **`inF`** (extended info) flag has been
enabled on the meter itself, usually with a flashlight held against the optical eye.

See the [Tasmota Smart Meter Interface docs](https://tasmota.github.io/docs/Smart-Meter-Interface/)
for the descriptor syntax and more OBIS codes.

## License

Licensed under the **GNU General Public License v3.0** — see [LICENSE](LICENSE). That is the license
of Tasmota itself, and the release binaries are builds of Tasmota with the override applied, so they
fall under it as well.

**Corresponding source** of a release: the upstream Tasmota tag named in the release, plus
`user_config_override.h` and the workflow in this repository at the commit the release was built from.
Nothing else is added to the firmware.

## Credits

- [arendst/Tasmota](https://github.com/arendst/Tasmota) — the firmware
- [darth-hp/tasmota-hichi-sml-firmware](https://github.com/darth-hp/tasmota-hichi-sml-firmware) — original idea and build config
