# Mitsubishi ceiling cassette (SLZ-KF12NA): infrared control findings

Notes from controlling a Mitsubishi 4-way ceiling cassette that has no wired controller,
using an ESP32 that imitates its handheld infrared remote.
This is a report, not a library. There is no code in it. Hobby work, no warranty, and
nothing here is safety advice.

How the report is organised:
- **Part 1** is what we measured, captured or watched happen on this unit.
- **Part 2** is what we are not sure of.
- **Part 3** is what we took from other people's work and did not check ourselves.

Conventions: byte numbers count from 0 at the first byte of the 17-byte frame, and bits
count from 0 at the least significant bit. Temperatures are Fahrenheit, because our
handheld works in Fahrenheit.

## Part 1. Confirmed on this unit

### 1. The unit, and why infrared

| Part | What it is |
|---|---|
| Indoor unit | SLZ-KF12NA, 4-way ceiling cassette, 12 kBtu |
| Outdoor unit | SUZ-KA12NA2 |
| Handheld | PAR-SL100A-E, wireless (infrared) remote |
| Receiver | PAR-SF9FA-E, wireless receiver panel in the cassette's corner |

- The cassette has no wired controller, only the wireless kit. Its CN105 port is in the
  ceiling space behind joists with no access, so we never connected to it. **Everything
  here is the infrared path only.**
- **Infrared is one-way.** The unit never answers, so nothing can be read back: not the
  setpoint, not the room temperature, and not whether the unit obeyed. Our controller keeps
  a local "staged" copy of the state, and that copy is never confirmed. Anyone using the
  handheld makes it stale.
- Nothing in the frame fields we have decoded is a room temperature. In AUTO the unit
  controls from its own sensor, and we cannot feed it a reading the way we can with a
  ducted air handler on CN105.
- For the same reason the "Wait" state described in the CN105 report is invisible here.

### 2. The frame

- **Carrier 38 kHz.** We make it with the ESP32's hardware PWM at 8-bit resolution, 85/255
  duty (33%), switched on and off for each mark.
- Pulse-distance coding: the bit is in the length of the **space** after each mark, and
  bits go out **least significant bit first**.

| Timing | Microseconds |
|---|---|
| Header mark | 3500 |
| Header space | 1705 |
| Bit mark | 488 |
| Zero space | 389 |
| One space | 1222 |
| Closing mark | 488 |

- A frame is **17 bytes, 136 bits**: 275 edges (2 header, 136 mark/space pairs, 1 closing
  mark), about **152 ms** on the air.
- We capture through a demodulating receiver module, which shifts the widths by tens of
  microseconds. Our decoder finds the split between short and long spaces from the data
  instead of using fixed constants, and a test with +/-25 us of jitter still decodes.

| Byte | Content |
|---|---|
| 0 to 4 | fixed: `23 CB 26 21 04` |
| 5 | bit 6 power (1 = on); bits 0 to 5 setpoint, F minus 27 |
| 6 | bits 0 to 2 mode |
| 7 | bit 0 always 1; bits 1 to 3 fan; bits 4 to 6 vane |
| 8 | fixed: `04` |
| 9 to 12 | `00` |
| 13 | fixed: `26` |
| 14, 15 | `00` |
| 16 | checksum |

- **Checksum** = sum of bytes 0 to 15, low 8 bits. Byte 16 is not included in its own sum.
- Every bit not listed is zero in everything we send. We do not know what those bits
  carry (see Part 2).
- Bytes 0 to 4, 8 and 13 were the same on every handheld press we captured.
- Worked example, built by our own code and checked against the rule above: power on,
  cool, 72 F, fan setting 2, vane middle gives
  `23 CB 26 21 04 6D 01 23 04 00 00 00 00 26 00 00 F4`.

### 3. What the handheld does (from captures)

We learned the tables below by capturing the handheld, button by button, with a 38 kHz
demodulating receiver module (VS1838B, TSOP4838 or TSOP38238, any of them) on a GPIO
with the internal pull-up enabled. Capture is raw: every mark and space, with no library
decode that could quietly assume the wrong protocol.

- **Every button re-transmits the whole state**, all fields, not just the one you changed.
- **The power bit is only meaningful on the power button's own frame.** The remote sets
  bit 6 of byte 5 to 1 only on the frame the POWER button itself sends. Every other
  button (fan, mode, vane, temperature) sends 0 there, which is also exactly what a
  power-off press sends. So in a captured frame the bit is not a reading of anything,
  unless nothing else changed since the previous frame. That is how a power press is
  recognised: nothing else moved, and the bit changed.
- **Our first conclusion about the polarity was backwards.** It said 0 means on, from
  comparing two captures with no physical check. A bench test with a real cassette
  responding to a transmitted frame showed the opposite: **1 = on, 0 = off**, a direct
  mapping. Do not trust a polarity read from captures alone.
- **The remote sends one extra full-state frame the moment it wakes from an idle
  display**, before it starts responding to new presses. So the first frame after a pause
  may not be the button you pressed.
- **Two bursts per press.** Our capture notes record the 17-byte state frame followed 65
  to 103 ms later by a 5-byte header-only tail. Another note of ours says the handheld
  sends the state frame twice. We did not settle which is right, and it may depend on the
  button.
- **A change from the previous decoded frame is not proof of a physical press.** The first
  frame after clearing the log has nothing to compare with, and a repeat shows no change.
  Our log labels each line CHANGED, NO-CHANGE or FIRST so that a line is never read as
  "this button did this".
- **A capture bug that cost time:** our receiver published a finished frame only when a
  later edge arrived. The last frame of a capture was missing and the log lagged the
  presses by one, which mis-assigned the modes until it was fixed.
- A frame ends on a space longer than 10 ms. Edges under 40 us are contact noise, and
  fewer than 12 edges is not a frame. A 17-byte frame is about 275 edges, so a 512-edge
  buffer has plenty of headroom.

### 4. The fields

All three values below were read from the handheld's own presses. The numbering is the
wire value, not the order on the remote.

**Setpoint** (byte 5, bits 0 to 5): degrees F minus 27, whole degrees only. The six bits
cover 27 to 90 F. We clamp to 50 to 90 F, which is the range our controller declares, not
a limit we found on the handheld. Example: 72 F is raw 45 (`2D`), so with power on byte 5 is
`6D`. A round trip through the encoder and decoder is lossless across the whole range.

**Mode** (byte 6, bits 0 to 2):

| Value | Mode |
|---|---|
| 0 | fan |
| 1 | cool |
| 2 | heat |
| 3 | auto |
| 5 | dry |

Values 4, 6 and 7 have never appeared. Cool was confirmed directly by a capture after the
lag fix above. The other four were aligned from an earlier capture that lagged by one
frame, so treat those as read from captures only.

**Fan** (byte 7, bits 1 to 3), a three-bit field:

| Value | Fan |
|---|---|
| 0 | speed 1 |
| 1 | speed 2 |
| 2 | speed 3 |
| 3 | speed 4 |
| 4 | auto |

Values 5 to 7 have never been sent. That is four speeds plus auto, recorded by stepping
through the handheld's fan button. **Do not borrow a generic library's two-bit fan table.**

**Vane** (byte 7, bits 4 to 6), seven values:

| Value | Position |
|---|---|
| 3 | topmost |
| 5 | upper middle |
| 2 | middle |
| 1 | lower middle |
| 0 | bottommost |
| 4 | auto |
| 6 | swing (the squiggle button) |

- **The raw codes do not run in physical order.** Top to bottom the codes are 3, 5, 2, 1,
  0. Our controller labels the five positions 1 to 5 from the top and keeps an explicit
  table from label to code. It does not do arithmetic on the code.
- The names follow the order the remote steps through them. See Part 2.

### 5. The ceiling bias

- The cassette controls to its own ceiling-mounted sensor, which we can neither read nor
  feed (IR is one-way and the frame carries mode/temp/fan/vane only). The only lever is
  the setpoint we send.
- A single value, `ceilingF`, is **added** to the target: send 72, and it heats until its
  own sensor reads 72, at which point the room is cooler. Positive means the ceiling reads
  warmer than the room.
- One value for all modes — not per-mode. The cassette has AUTO where it picks its own
  direction, and with no read-back there is no way to know which way it went.
- Rounded to whole degrees, not truncated, so a negative bias rounds symmetrically.
- Range limit +/- 15 F; step 1 F because the IR frame carries whole degrees only.

### 6. Transmit hardware and timing

- **IR_REPEATS = 2** with a 16 ms gap. The unit acts on a group of bursts, not a single
  one. A single frame flashes perfectly on a phone camera and is silently ignored — this
  was the "transmitter obviously firing but the unit never responds" symptom.
- Transmit runs with the FreeRTOS scheduler suspended (not a true critical section, because
  `ledcWrite()` takes a mutex). Without it, a 75,769 µs stall was caught on a live capture
  — mid-transmit task switching corrupts the frame while the log still says 200 OK.
- Hardware: TSAL6200-class 5 mm 940 nm emitter, driven through a transistor. Reach tested
  to 6 ft; the install target is 5 ft with margin.
- Aim is the whole game: single ~40° bevel up, or two facets at ~25° and ~55°, brackets the
  23°–59° elevation range from a panel at 4–5 ft looking up at an 8–9 ft cassette.
- The ESP32-P4 board's function DIP switch selects whether IO29 (the IR drive pin) goes to
  the GPIO header or the wireless module. Position 1 (header free) is required for the IR
  blaster to work at all.
- CN105 uses a separate dedicated UART3 port, so nothing here depends on the DIP switch.

## Part 2. Not confirmed, or only partly

- **Two bursts per press.** Our capture notes record the 17-byte state frame followed
  65 to 103 ms later by a 5-byte header-only tail. Another note says the handheld sends
  the state frame twice. We did not settle which is right, and it may depend on the
  button.
- **Mode values 4, 6, 7 (dry especially).** Dry came from the lagged capture, not the
  confirmed one. Cool was confirmed directly after the lag fix; the other four were
  aligned from the earlier misaligned log.
- **Bytes 0 to 4, 8 and 13 were the same on every press we captured.** We do not know
  whether they carry anything on presses we never made.
- **Bits not listed in section 2** — including everything in bytes 9 to 12 and 14 to 15 — are
  zero in everything we send. We do not know what they carry.
- **The receiver module is uncharacterised.** We assume a 38 kHz demodulator needs about
  0.3–1 mW/m²; the cassette's own receiver could need more or less.
- **Vane ordering.** The physical order top-to-bottom is codes 3, 5, 2, 1, 0. We label
  positions 1 to 5 top to bottom. Whether "1" is really "highest/closest to ceiling" (as
  wall-unit reports say) was not independently confirmed for this cassette.

## Part 3. From other people's work, not checked by us

### Comparison with IRremoteESP8266's Mitsubishi136

IRremoteESP8266 publishes a Mitsubishi136 protocol. Its first four header bytes match ours
(`23 CB 26 21`), but almost everything else differs — our cassette is **not** standard
Mitsubishi136, so do not drop a generic library in and expect it to work:

| Item | Our cassette | IRremoteESP8266 Mitsubishi136 |
|---|---|---|
| Header mark | 3500 µs | 3324 µs |
| Header space | 1705 µs | 1474 µs |
| Bit mark | 488 µs | 467 µs |
| Zero space | 389 µs | 351 µs |
| One space | 1222 µs | 1137 µs |
| Temperature | °F (raw = F − 27) | °C (raw = C − 17) |
| Fan field | 3 bits (5 values) | 2 bits (4 values) |
| Vane/swing | 3 bits (7 values) | 4 bits swing |
| Checksum | sum of bytes 0–15, low 8 bits | byte-inverted copy in second half |
| Min repeat | 2 (single frame ignored) | kNoRepeat |
| Mode values | fan 0, cool 1, heat 2, auto 3, dry 5 | cool 1, heat 2, auto 3, dry 5, fan 0 |

IRremoteESP8266 is LGPL v2.1. We did not use it, and it is not a source for anything in
this report.

### echavet/MitsubishiCN105ESPHome

- **Issue #672 (fan speed mismatches on ceiling cassettes):** MLZ-KP18NA and MLZ-KP06NA
  cassettes skip `0x02` and `0x04` entirely in the CN105 fan byte, sequencing `0x00 → 0x01 →
  0x03 → 0x05`. The same reporter says vane controls differ on cassettes too. We did not
  test this — it is a CN105 observation, not an IR one.
- **Issue #283 (horizontal vane resets):** the vane setting resets to "`|`" after a
  temperature change or reboot on some units; fixed upstream by keeping a sticky last
  user-selected vane and re-applying it. We did not test this.
- Neither issue concerns the PAR-SL100A-E handheld or SLZ-KF12NA specifically.

### What we bought / fitted

- TSAL6200-class emitters (5 mm, 940 nm, 100 mA class, stated radiant intensity 40/72 mW/sr
  min/typ). uxcell 10-packs were bought; Vf at 100 mA is verified against the 1.35–1.6 V
  window on arrival.
- VS1838B / TSOP4838 / TSOP38238 demodulating receiver modules for capture.

## Credits

- **crankyoldgit/IRremoteESP8266**: https://github.com/crankyoldgit/IRremoteESP8266 — LGPL v2.1.
  Published the Mitsubishi136 timing constants and field layout we compared against. We did
  not use the library; this report only notes where our frame diverges from it.
- **echavet/MitsubishiCN105ESPHome**: https://github.com/echavet/MitsubishiCN105ESPHome
  — cassette fan-byte and vane reports (issues #672 and #283). Neither was tested here.

This report contains no code from either project. The frame layout and timing are ours,
measured on one unit. A different cassette model may behave differently.

If something here is wrong for your unit, please say so. We tested one cassette.