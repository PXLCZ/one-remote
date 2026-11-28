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
- **Effects:** Rainbow, Fade, Jump, Breathe, Christmas and Candle on every light in All, in step, with a speed slider and an option to give each light a different color. Like Sound mode, the page works out each frame and sends it in each light's own protocol, so different brands stay in sync. The lights' built-in effects can't do that, because each runs on its own controller.
- **Sound mode:** mic-reactive Energy / Rhythm / Roll / Spectrum, plus a tap-tempo BPM fallback.
- **Per light:** rename, "In All" toggle, color, hue, brightness, raw hex sender (Advanced), Bluetooth log.
- **ELK:** 22 built-in effects plus speed.
- **Hello Fairy:** scenes 1–58 (type a number to jump), favorites, scene loop, **DIY per-section painter** with motion, speed, brightness and saved patterns.
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

**DIY, decoded from PacketLogger traces of the official app (2026-11-27 and 2026-11-28):**
- **Order map:** `AA A6 B2 00 01 … B1 CK`. This identity table of 178 entries is sent before DIY, and the light answers `AA A6 01 B2 03`.
- **State:** `AA D0 04 MM 64 SS ST BB`
  - MM is motion: 0 = static. The app's Mode row has seven buttons: two curved arrows, ↓, ↑, two sparkles and a pin. ↓ sends 4 and ↑ sends 3; the other five aren't mapped yet. In One Remote, 1, 2 and 5–7 are untested.
  - The `64` never changes, not even when the brightness slider moves.
  - SS is speed, 1–100.
  - ST is 00 = editing, 01 = playing. The app also sends 03 as DIY opens, just before 00; meaning unknown.
- **Paint:** `AA C4 LEN 00 [IDhi IDlo HUE VAL]×n BB`
  - HUE is hue° / 2 (00 = red, 3C = green). `FF` is white and `FE` is blank: the app's editor shows `FE` sections as empty, and they stay dark.
  - VAL is brightness: 0–250, where `FA` is full and `00` is off. The app's DIY brightness slider repaints every section with VAL = ⌊percent × 2.5⌋, about every 50 ms while dragging.
  - At most 24 sections per packet.
  - Two single-section paints started with `01` instead of `00`; meaning unknown.
- **Opening DIY**, the app sends the order map, state 03, state 00, its saved canvas, then state 01.
- The curtain reports **26 DIY sections** (indices 0–25), and each section is one bulb. The app draws them as one string; section 1 lit the tip of a snowflake arm and section 2 the next bulb in.
- **Still unconfirmed:** the other five Mode buttons, what `AA D7 02 01 00 DD` does (seen once), where sections 3–26 sit, and whether every snowflake shows the same pattern.
- The curtain has no text feature (snowflakes, not a matrix).

**What the official app sends on connect:**
1. It turns on notifications (handles `0x0006` and `0x000C`).
2. Clock sync: `AA 05 07 TT TT TT TT 00 02 58 BB`, where TT is the Unix time, big-endian. The last three bytes never changed.
3. `AA 00 00 BB`. The light replies with its firmware version (`3.1.4M89`) and `1A` (26, the section count).
4. It queries `AA 0C 00 BB` (the reply is `64`), `AA 19 00 BB` and `AA 01 00 AB` (status), then sends `AA 20 03 00 01 00 BB`. Their meaning is unknown, and One Remote works without them.

**Scenes from the official app:** `AA 03 07 02 NN 03 E8 00 64 00 FF CK`. Its length byte says 7, but the frame carries 8 bytes. The light replies with the scene number and a 7-color list: hues `00 3C 78 1E 5A 87 FF` (red, green, blue, yellow, cyan, purple, white), each with VAL `FA`. One Remote's shorter `AA 03 04 02 NN VVvv` also works.

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
1. Map the other five DIY Mode buttons (tap each once in a short trace) and work out `AA D7 02 01 00 DD`. White, blank, brightness, speed and the ↓ / ↑ buttons are confirmed (2026-11-28).
2. Finish mapping the 26 sections to bulbs: section 1 is the tip of a snowflake arm and section 2 the next bulb in.
3. Delete the Bluetooth logging profile from the iPhone afterwards (Settings → General → VPN & Device Management).

## Capturing BLE traffic (how the DIY protocol was found)
1. Get PacketLogger from **Additional Tools for Xcode** (in the Hardware folder).
2. Install Apple's **Bluetooth for iOS** logging profile on the iPhone, then restart the phone.
3. Plug the iPhone in and choose File → New iOS Trace.
4. Use the official app, then save the `.pklg` file.
5. Open **[decode.html](https://pxlcz.github.io/one-remote/decode.html)** and drop the file in. It reads the file locally, keeps only the lights' packets, labels the known commands and starts a new step at each pause. Share the decoded text rather than the `.pklg`, because a raw trace holds all of the phone's Bluetooth traffic.

File format, for decoding by hand: records are little-endian (4-byte length, 8-byte timestamp, 1-byte type, payload); some files are big-endian, and the decoder handles both. Types 2 and 3 are ACL TX/RX. Within ACL, L2CAP CID 4 carries ATT; opcode `0x52` is Write Command (handle `0x0009`) and `0x1B` is Notify (handle `0x000B`).

### Capture checklist (used for the 2026-11-28 trace)
Leave about 5 seconds, hands off, between steps. The decoder starts a new step after each pause, so its steps line up with this list. Close One Remote first, since the curtain accepts only one connection, and check that the logging profile is still on the iPhone.

In the Hello Fairy app:
1. Open DIY.
2. Paint section 1 red. This is a known packet, so it confirms the trace lines up.
3. Paint section 2 white.
4. Make section 3 blank.
5. Tap each motion/direction button once, in order, and note the order.
6. Drag the brightness slider slowly from lowest to highest.
7. Drag the speed slider from lowest to highest.
8. Press play, if the app has a play button.

Save the trace, decode it, and send the decoded text along with the button order from step 5.
