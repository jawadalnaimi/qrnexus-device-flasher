# QR Nexus Alarm Emulator — Web Flasher

A browser-based installer (ESP Web Tools). Anyone with a Chromium browser can flash an
**ESP32-WROOM** without installing anything — and the firmware here carries **no secrets**
(WiFi + token are entered on the device after flashing, via its `QRNexus-Setup` portal).

## Files

| File | Offset | What |
|---|---|---|
| `app.bin` | `0x10000` | the firmware (no baked secrets) |
| `bootloader.bin` | `0x1000` | ESP32 bootloader |
| `partitions.bin` | `0x8000` | partition table |
| `boot_app0.bin` | `0xe000` | OTA boot selector |
| `manifest.json` | — | ESP Web Tools manifest (lists the parts above) |
| `index.html` | — | the flasher page |

## Host it (one-time)

ESP Web Tools needs **HTTPS** (or `http://localhost`). Easiest is GitHub Pages:

```bash
# from a repo that contains this flasher/ folder
# Settings -> Pages -> deploy from branch -> /flasher  (or push to a gh-pages branch)
```
Then share the URL. Locally you can also test with:
```bash
cd flasher && python3 -m http.server 8000   # open http://localhost:8000
```

## Recipient flow
1. Open the page in **Chrome/Edge** (desktop), plug in the ESP32.
2. **Connect & Install** → pick the serial port → **Install** (full erase + flash, ~1 min).
3. Join the device's **`QRNexus-Setup-XXXX`** WiFi → open `http://192.168.4.1`.
4. Enter WiFi + the `qrn_dev_…` token (minted in app.qrnexus.io → Alarm Panels → key icon).
   The device reboots, connects, and starts reporting. Its web UI is then on your LAN.

To re-provision later: the device's web UI has a **Reconfigure / Change WiFi** button (wipes
stored config and reboots into the setup portal).

## Rebuild these binaries

From `SIA_CID_device/` (builds WITHOUT `secrets.h` so nothing sensitive is embedded):
```bash
mv esp32_alarm_emulator/secrets.h esp32_alarm_emulator/secrets.h.local 2>/dev/null
arduino-cli compile --fqbn esp32:esp32:esp32 --output-dir build_dist esp32_alarm_emulator
mv esp32_alarm_emulator/secrets.h.local esp32_alarm_emulator/secrets.h 2>/dev/null
cp build_dist/esp32_alarm_emulator.ino.bin            flasher/app.bin
cp build_dist/esp32_alarm_emulator.ino.bootloader.bin flasher/bootloader.bin
cp build_dist/esp32_alarm_emulator.ino.partitions.bin flasher/partitions.bin
# boot_app0.bin comes from the esp32 core (tools/partitions/boot_app0.bin)
```
Verify no secrets leaked: `strings flasher/app.bin | grep -E 'qrn_dev_|<your-wifi-pass>'` → no output.

> Note: distributing `app.bin` hides your **source code**, but a plain `.bin` still exposes
> any embedded strings. This build embeds no secrets, so that's fine. If you ever need the
> firmware unreadable off the physical chip, enable ESP32 **flash encryption + secure boot**
> (per-device provisioning) — separate from this web flasher.
