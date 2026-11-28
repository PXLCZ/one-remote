# One Remote: project notes and protocol reference

Handoff doc so any session (or person) can pick this up without the original chat.

## What it is
A single-file Web Bluetooth app (`index.html`) that controls several brands of cheap BLE LED lights from one page.
Live at **https://pxlcz.github.io/one-remote/** (GitHub Pages, `main` branch, root folder).

- **iPhone:** open in the **Bluefy** browser (Safari has no Web Bluetooth).
- **Mac / Android / PC:** Chrome or Edge.
- Each light accepts only **one BLE connection**, so close the official apps first.

## Owner's lights
| Light | Bluetooth name | Family |
|---|---|---|
| LED strip (Lotus Lantern X app) | `ELK-BLEDDM 45` | ELK-BLEDOM |
| BYIUA snowflake icicle curtain, 33 ft, 384 LEDs, 32 drops (Hello Fairy app) | `Hello Fairy-435DPPPP` | Hello Fairy (Hysiry) |
| Older Govee light(s) | `ihoment_H…` / `Govee_H…` | Govee |

## Features in the app
- **All Lights:** on/off, color swatches, hue, brightness, sleep timer.
- **Sound mode:** mic-reactive Energy / Rhythm / Roll / Spectrum, plus a tap-tempo BPM fallback.
- **Per light:** rename, "In All" toggle, color, hue, brightness, raw hex sender (Advanced), Bluetooth log.
- **ELK:** 22 built-in effects plus speed.
- **Hello Fairy:** scenes 1–58 (type a number to jump), favorites, scene loop, **DIY per-section painter** with motion, speed and saved patterns.
- **Govee:** color-command style picker (Auto / Classic / Bulb / Segmented) and a keep-alive.

## Protocols

### ELK-BLEDOM (Lotus Lantern X)
Service `0xFFF0` with write char `0xFFF3` (fallback `0xFFE0` / `0xFFE1`). Use full 128-bit UUID strings, because Bluefy rejects short IDs. Frames are 9 bytes: `7E xx CMD … EF`.
- **On / off:** `7E 00 04 F0 00 01 FF 00 EF` / `7E 00 04 00 00 00 FF 00 EF`
- **Color:** `7E 00 05 03 RR GG BB 00 EF`
- **Brightness (0–100):** `7E 00 01 BB 00 00 00 00 EF`
- **Effect (0x87–0x9C):** `7E 00 03 EE 03 00 00 00 EF`
- **Speed:** `7E 00 02 SS 00 00 00 00 EF`

### Hello Fairy
- **Service:** `49535343-fe7d-4ae5-8fa9-9fafd205e455`
- **Write:** `49535343-8841-43f4-a8d4-ecbe34729bb3`
- **Notify:** `49535343-1e4d-4bd9-ba61-23c647249616`

Frame: `AA CMD LEN data… CK`. CK is the sum of the previous bytes mod 256, but the official app often sends a literal `BB` instead and the device accepts both.
- **Power:** `AA 02 01 01|00`
- **HSV color:** `AA 03 07 01 HHhh SSss VVvv`. Hue 0–359; sat and val are 0–1000 (percent × 10).
- **Scene:** `AA 03 04 02 NN VVvv`

**DIY, decoded from a PacketLogger trace of the official app (2026-11-27):**
- **Order map:** `AA A6 B2 00 01 … B1 CK`. This identity table of 178 entries is sent before DIY.
- **State:** `AA D0 04 MM 64 SS ST BB`
  - MM is motion: 0 = static, 3 was seen; others untested.
  - SS is speed, 1–100.
  - ST is 00 = editing, 01 = playing (03 also seen, meaning unknown).
- **Paint:** `AA C4 LEN 00 [IDhi IDlo HUE VAL]×n BB`
  - HUE is hue° / 2 (00 = red, 3C = green).
  - VAL was always `FA` in the app.
  - At most 24 sections per packet.
- The curtain reports **26 DIY sections** (indices 0–25).
- **Still unconfirmed:** white (currently sends HUE `FF`), blank/off (currently sends HUE 0 with VAL `00`), what motion values other than 0 and 3 do, and which physical snowflake each section maps to.
- The curtain has no text feature (snowflakes, not a matrix).

### Govee
- **Service:** `00010203-0405-0607-0809-0a0b0c0d1910`
- **Write:** `…2b11`

Packets are 20 bytes: `33 CMD data…`, zero-padded, with byte 19 = XOR of bytes 0–18. Send keep-alive `AA 01` every 2 s or the light disconnects.
- **Power:** `33 01 01|00`
- **Brightness:** `33 04 LL`. LL is 0–254 for Classic and 0–100 for Segmented.
- **Color:**
  - Classic: `33 05 02 R G B`
  - Bulb: `33 05 0D R G B`
  - Segmented: `33 05 15 01 R G B 00 00 00 00 00 FF 7F`
- Auto-style uses Segmented for H6053, H6072, H6102, H6199, H617A and H617C; everything else gets Classic.

## Not supported, and why
- **Feit:** mostly Wi-Fi Tuya bulbs; the BLE fallback is encrypted with per-device keys. Would need the Tuya cloud API via a small server, or Home Assistant.
- **Nanoleaf panels:** local HTTP API on port 16021, but an https page can't call plain-http LAN devices, so this needs a local relay.
- **Nanoleaf Essentials:** proprietary BLE/Thread, so not feasible. A possible workaround is buttons that run iOS Shortcuts (`shortcuts://run-shortcut?name=…`) for lights that are in Apple Home.

## Open to-dos
1. Capture one more trace in the Hello Fairy app's DIY: one white bulb, one blank bulb, each motion/direction button, and the brightness slider. Confirm the guesses above.
2. Map the 26 sections to physical snowflakes (paint section 1, see where it lights).
3. Delete the Bluetooth logging profile from the iPhone afterwards (Settings → General → VPN & Device Management).

## Capturing BLE traffic (how the DIY protocol was found)
1. Get PacketLogger from **Additional Tools for Xcode** (in the Hardware folder).
2. Install Apple's **Bluetooth for iOS** logging profile on the iPhone, then restart the phone.
3. Plug the iPhone in and choose File → New iOS Trace.
4. Use the official app, then save the `.pklg` file.

Decode it by parsing records (little-endian: 4-byte length, 8-byte timestamp, 1-byte type, payload). Types 2 and 3 are ACL TX/RX. Within ACL, L2CAP CID 4 carries ATT; opcode `0x52` is Write Command (handle `0x0009`) and `0x1B` is Notify (handle `0x000B`).
