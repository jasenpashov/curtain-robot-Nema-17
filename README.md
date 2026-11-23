# ESPHome Curtain Robot with TMC2209 Sensorless Homing

A curtain robot that drives a NEMA 17 stepper along a standard aluminium curtain
track, with **no limit switches and no magnets anywhere on the rail**. The
TMC2209's StallGuard detects the physical end stop by sensing motor load, so the
only moving part is the carriage itself.

It runs ESPHome on an ESP32-C3 Super Mini, appears in Home Assistant as a cover
entity, and is driven by a Zigbee remote plus two automations tied to an outdoor
light sensor: open at dusk, close in bright sun.

The curtain it moves is a 2.5 m heavy blackout curtain. The key idea is
mechanical rather than electronic: an off-the-shelf printable planetary gearbox
is pushed straight into the wheel that runs on the rail, so the gearbox becomes
the hub.

📖 **Full build, wiring and troubleshooting:**
[ESPHome Curtain Robot With TMC2209 Sensorless Homing](https://www.instructables.com/ESPHome-Curtain-Robot-With-TMC2209-Sensorless-Homi/)

---

## What's in this repository

```
esphome/
  curtain-left.yaml        ESPHome config, left curtain
  curtain-right.yaml       Same, mirrored, with an offset restart schedule
  secrets.yaml.example     Template for WiFi and API keys
home-assistant/
  automations.yaml         Remote buttons + daylight automations
  lovelace-curtains.yaml   Dashboard card for both curtains
models/                    STL files for the carriage, mount and gearbox
```

---

## Why sensorless homing

A Hall sensor needs a magnet glued to the rail and a stable air gap — and the
carriage tilts under the weight of the curtain, so that gap moves. Mechanical
switches need brackets at both ends. StallGuard needs neither: the motor itself
is the sensor, and the end stop can be any hard obstruction already on the track.

It also gives obstruction detection for free. If something blocks the curtain
mid-travel, the motor senses the load and stops.

---

## Wiring

| ESP32-C3 | TMC2209 | Notes |
|---|---|---|
| GPIO3 | STEP | |
| GPIO4 | DIR | Flip `inverted` in the YAML if direction is wrong |
| GPIO5 | EN | Active low; driven manually so the driver can be reset |
| GPIO6 | DIAG | **Soldered to a bare pad**, not a header pin |
| GPIO20 (RX) | PDN_UART | Direct |
| GPIO21 (TX) | PDN_UART | Through a 1 kΩ resistor, same pad |
| 3V3 | VIO | Logic supply |
| GND | GND | |
| — | MS1, MS2 | To GND (sets UART address 0x00) |
| — | CLK | Leave unconnected |

Power is split: 12 V straight to the driver's VM, and a separate buck module
drops 12 V to 5 V for the ESP. **All grounds must be common.**

Fit a **1000 µF / 25 V electrolytic across VM and GND at the driver**, leads as
short as possible, plus a 100 nF ceramic in parallel. On a long cable run this
matters more than wire thickness — the capacitor supplies the current peaks
locally, which the cable cannot do fast enough. Skipping it causes brownout
reboots that look like a dozen other faults.

> ⚠️ TMC2209 modules vary enormously between vendors — whether the UART is one
> pad or two pins, whether a 1 kΩ resistor is already fitted, whether DIAG is
> broken out, and whether anything is labelled at all. Confirm the pinout on
> **your** board with a multimeter before applying power. Reversing VM and GND
> destroys the chip instantly.

---

## Setup

1. Copy `esphome/secrets.yaml.example` to `secrets.yaml` and fill in your WiFi.
2. Generate an API key per device: `openssl rand -base64 32`
3. Check the marking on the driver's sense resistors (R100 / R110 / R150) and
   set `rsense` to match.
4. Flash with the motor **disconnected** and watch the log. `Detected IC version:
   0x21` proves the UART link works.
5. Power down, connect the motor, power up.
6. Calibrate StallGuard (below), then write the value into `sg_threshold`.

### Calibrating StallGuard

The threshold is 0–255; higher means more sensitive. The slider in Home
Assistant changes it live, so no recompiling.

Set it to 255 and send a command — the motor should stop almost immediately,
which confirms DIAG is on the right pad. Then lower it in steps until the
carriage reaches the end stop and stops there with no false stops along the way.

**35** works here, with a 2.5 m track and a heavy blackout curtain. Yours will
differ with curtain weight, track friction and run current.

---

## Known issue: the 3-hourly restart

The external TMC2209 component carries an old fork of ESPHome's core `stepper`
component. After many hours of idling it stops producing step pulses: the ESP
stays online, the driver still answers over UART, commands are logged — but the
motor never moves. Only recreating the component fixes it.

Both configs therefore restart the ESP every three hours (offset from each
other), plus a watchdog that restarts and replays the command if the position
does not change within two seconds of a move starting. It is a workaround, not
a fix. If ESPHome ever gains native TMC2209 support, switch to it.

---

## Parts

| Part | Notes |
|---|---|
| ESP32-C3 Super Mini | Any ESP32 works; the pin numbers here are for the C3 |
| TMC2209 driver with heatsink | [AliExpress](https://www.aliexpress.com/item/1005006938902666.html) — must be a real 2209, not a 2208 (no StallGuard) |
| NEMA 17 17HS4023, 0.7 A, 9 N·cm | [AliExpress](https://www.aliexpress.com/item/1005003874936862.html) — a short-body motor fits better |
| 12 V PSU, 2 A or more | A 36 W / 3 A LED supply is a good cheap choice |
| Buck module 12 V → 5 V | Powers the ESP only |
| 1000 µF / 25 V, 105 °C + 100 nF | At the driver's VM. Not optional |
| 1 kΩ resistor | For the single-wire UART link |
| 2 × springs 10 × 32 mm | Part of the carriage |
| 3 × M3 heat-set inserts | In the gearbox output hub |
| M3 screws | [Printable screw test set](https://www.printables.com/model/1345044-hundreds-of-metric-hex-screws-version-1-m2-m3-m4-m) is handy for checking sizes |
| 2-core cable, 0.75 mm² | For the 12 V run along the track |

Geared down through the planetary reducer, 9 N·cm at the motor is ample for a
heavy curtain, and the low current keeps the driver cool.

---

## 3D printed parts

**Planetary gearbox.** A remix of
[NEMA 14 Stepper Planetary Gearbox 1:4 Compact](https://www.printables.com/model/1193642-nema-14-stepper-planetary-gearbox-14-compact)
by **CrossedGuardians**, adapted for a NEMA 17 motor. Every part prints flat
with no supports, thanks to their two-piece planet carrier — full credit to
them for the idea.

Gear set: sun 15T, planet 21T, ring 57T, module 0.657, 20° pressure angle,
ratio 1:4.8, all three planets meshing correctly at 120°. The STL files live
in `models/`.

Also used: an M12×18 hex bolt from
[Hundreds of metric hex screws](https://www.printables.com/model/1345044-hundreds-of-metric-hex-screws-version-1-m2-m3-m4-m),
scaled to 96%.

Print with no supports; the files are oriented as exported. Print the gears
solid or with at least three walls, and turn on elephant-foot compensation
(~0.15 mm) and X-Y hole compensation (~0.05 mm) — at module 0.657 those
corrections decide whether the gears mesh or bind.

**Other printed bits.** Shower curtain hooks, printed unmodified, are used to
clip the 12 V cable to the back of the curtain — not for the drive. A strip cut
from an old bicycle inner tube slips over the drive wheel like a tyre and gives
it far more grip on the rail than bare plastic.

---

## Licence

The ESPHome and Home Assistant configuration in this repository is MIT licensed
— use it however you like.

The 3D models are remixes and inherit their originals' licences
(CC BY-NC-SA 4.0, non-commercial).

---

## Credits

The English write-up, the diagrams and the gearbox corrections were worked out
with [Claude](https://claude.ai).
