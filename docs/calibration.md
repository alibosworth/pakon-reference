# Calibration

Before scanning, the OEM programs the CCD (gain, offset, per-channel exposure)
and computes a per-column correction from calibration passes. This is what
makes the raw stream usable: it normalises the lamp profile and each sensor
element's response.

_From the OEM driver and Ansel pipeline reverse engineering, and the parameter
table read seen in captures, March–May 2026._

## The per-unit EEPROM

What earlier drafts of this page called "the parameter table read" is the OEM
software reading the scanner's per-unit EEPROM at start-up: a serial I2C
EEPROM at 7-bit address `0x52` on the motherboard, read through the FX2
bridge with two vendor control requests. It holds the data that makes one
unit that unit: serial number, per-resolution optical offset and motor
speeds, motor-adjust words, and the two colour matrices. It does **not**
hold the light calibration (LED currents and duty cycles); the OEM keeps
those in the Windows registry, written when its light-correction routine
runs. [CONFIRMED] contents by live read on an F-135+ (serial 16402), August
2026, matching the layout decoded from the OEM engine by the
[pakon-mac](https://github.com/gazzdingo/pakon-mac) project
(`docs/69-calibration-auto-load.md`).

### The read

Two vendor requests, both with `wIndex = 0x1234`, repeated per chunk:

| Step | bmRequestType | bRequest | wValue | wLength |
|---|---|---|---|---|
| select | `0x40` (vendor OUT, no data) | `0xA4` | `0x00A5` | 0 |
| read | `0xC0` (vendor IN) | `0xA9` | byte offset into the EEPROM | ≤ 32 |

`0x00A5` is the EEPROM's 8-bit I2C address with the read bit set
(`(0x52 << 1) | 1`); the select is re-issued before every read. The OEM
reads 32 bytes at a time at increasing offsets (`0x008`, `0x028`, `0x048`,
…), which is the "fixed number of pairs, resolution independent" pattern the
captures showed. [CONFIRMED] on hardware; the same two requests, and no
others, are all that is needed to read the chip. The reads here were made
with the [pakon-tlx-macos](https://github.com/pablonavarrob/pakon-tlx-macos)
project's `tools/eedump.py`, which issues exactly this sequence (decoded
from a live capture of the OEM engine's own transfers), extended to read
the backup copies and check the CRCs. There is a matching write
request (`0xA2`) which the OEM's normal scanning path never issues; only its
Calibration Wizard writes the EEPROM.

Note that `0xA9` with `wIndex = 0` is a different operation: the 8-byte
personality read served by the stage-1 loader (see
[usb-identity-and-firmware.md](usb-identity-and-firmware.md)). The
`wIndex` value selects which.

### Layout

The data is stored **twice**. Two sections, each a header
`{u32 length; u32 crc32}` followed by the payload, with the CRC-32 (the
standard reflected zlib/PKZIP polynomial) taken over the payload only.
Each section has a backup copy:

| Section | Primary | Backup | Length |
|---|---|---|---|
| A | `0x000` | `0x400` | 398 bytes (8 header + 390) |
| B | `0x800` | `0xA00` | 36 bytes (8 header + 28) |

Section A payload, absolute byte offsets, little-endian:

| Offset | Type | Field |
|---|---|---|
| `0x008`, `0x00C` | u32, u32 | two unnamed words (400 and 1351 on the reference unit) |
| `0x010` | u32 | **scanner serial number** |
| `0x014` / `0x016` / `0x018` | u16 ×3 | resolution base 4: optical Offset, MotorSpeed, MotorSpeed (IR) |
| `0x01A` / `0x01C` / `0x01E` | u16 ×3 | resolution base 8: same three |
| `0x020` / `0x022` / `0x024` | u16 ×3 | resolution base 16: same three |
| `0x026` – `0x09D` | f32 ×30 | **NegMatrix0–29**: the colour-negative correction, 3 rows × 10 |
| `0x09E` – `0x115` | f32 ×30 | **PosMatrix0–29**: the positive (slide) correction, 3 rows × 10 |
| `0x116` – `0x18D` | 120 bytes | unused (zero on every unit seen) |

Section B payload: twelve u16 motor-adjust words (`0x808`–`0x81E`, values
near 1000 and clamped by the OEM to 900–1100), then one unnamed u32.

Each 3×10 matrix row is `R, G, B, R², G², B², RG, GB, BR, constant`, i.e. a
second-order colour matrix; on the reference unit the quadratic columns are
all ≈ 0 and NegMatrix is in effect a 3×4 affine (diagonal ≈ 0.29 / 0.29 /
0.32, constants ≈ +166 / +430 / +638), and PosMatrix is the plain 0.25
diagonal with no cross terms or offsets. These are per-unit factory
values, not shared constants; the OEM also mirrors them into the registry.
[CONFIRMED] layout and values on serial 16402; layout first established
by [pakon-mac](https://github.com/gazzdingo/pakon-mac) from the OEM engine.

Reference-unit motor constants, for scale (per-unit; do not copy):

| Base | Offset | MotorSpeed | MotorSpeed (IR) |
|---|---|---|---|
| 4 | 30 | 25726 | 19278 |
| 8 | 58 | 11434 | 7557 |
| 16 | 60 | 5900 | 4836 |

### Verifying a read

Read both copies of both sections, check all four CRCs, and compare
primary with backup. On the reference unit the backup of section A and
both copies of section B validate; the primary of section A fails its CRC
because of a single byte, `0x0A5`, which reads `0x48` where the backup has
`0x00` (the high byte of `PosMatrix1`, turning a 0.0 into 131072.0). Stable
across four reads and two power cycles, so a stored-data fault, not a read
artifact; it touches only the slide matrix.

How the OEM uses the two copies: at start-up it reads the primary of each
section, checks the length and CRC, and if either is bad reads the backup
and uses that instead. The first good copy wins; the primary is not
repaired. Only if both copies of a section fail does it keep what it read
and raise the SDK's `INITIALIZEW_EEPROM_BLANK` /
`INITIALIZEW_EEPROM_CHECKSUM_BAD` warning (a warning, not an error).
[CONFIRMED] two ways on the reference unit: the retry loop in the engine's
EEPROM-to-registry routine, and the registry it populates, which holds the
backup's `PosMatrix1 = 0.0`, not the primary's corrupted value, with no
warning shown. So a unit can carry a bad primary copy for years without
anyone noticing. Anyone archiving their own unit's EEPROM should read both
copies; a dump of the primaries alone cannot show whether it is good.

The [pakon-mac](https://github.com/gazzdingo/pakon-mac) project reports that on its unit these EEPROMs returned good
data only on the first transaction after a power cycle and degraded on
later reads while still reporting success. The reference unit shows no such
behaviour (four consistent reads), so treat that as unit-dependent: read
once per power cycle, and confirm by comparing across power cycles rather
than by re-reading within one.

## CCD register banks

The CCD is configured through register writes on the command channel (light
controller). The reverse engineering identified banked registers for scan
geometry and exposure/gain:

- a scan-window/geometry bank (start, end, and timing/span of the active CCD
  window)
- an exposure/gain bank (per-channel R/G/B exposure, and per-channel dark trim
  with a sign bit)
- a profile-mode selector (visible / IR / complete) that gates whether the IR
  channel is acquired

[DOCUMENTED]/[INFERRED] from the register-write sequences; the profile-mode
selector corresponds to the IR/Digital-ICE fourth channel appearing in the
image stream (see [image-stream.md](image-stream.md)).

## Dark / bright calibration and fixed-pattern correction

The OEM computes a **per-column fixed-pattern correction** from calibration
lines (a set of dark reference lines and a set of bright, open-gate lines) so
that each CCD column's gain and offset are individually normalised.
[DOCUMENTED] the per-column dark-offset subtraction and per-column gain
multiplication are visible in the OEM line processor; the exact
gain-computation formula was reverse-engineered later (documented below).

A practical constraint worth recording: on this hardware the lamp only
illuminates while the film transport is running, so the bright calibration pass
must be taken with the motor moving: the calibration is driven, not a
motor-off measurement. [INFERRED] from the lamp/transport behaviour.

### The per-column gain formula (partial)

The per-column gain computation was reverse-engineered from the OEM
`CiConfigFixedPatternCorrection` routine. The known form is:

```
gain[column] = (125 · 2^32) / (bright[column] − dark[column] − <prefix terms>)
```

with a dark target near 300 and a bright target near 64000, and the result
clamped to a maximum. **This is a partial formula**: the `<prefix terms>` in
the denominator and the exact clamp value are not independently established
here, so it is not yet implementable as written. What is solid is the constant
`125 · 2^32` (= `0x7d00000000`), which was independently cross-confirmed
against the same constant in Stefan Dierauf's libpakon, and the dark/bright
targets. [DOCUMENTED, partial; June 2026]

## Colour vs. calibration

This CCD-level calibration (gain/offset/exposure/fixed-pattern) is distinct from
the colour negative inversion. Calibration makes the raw linear CCD values
correct and uniform; the C-41 inversion (LUT + matrix) is a separate, later
stage documented in [color-pipeline.md](color-pipeline.md).

## Open questions

- The fixed-pattern-correction formula is partial (prefix terms and clamp).
- The two unnamed u32 words at the top of section A and the one at the end
  of section B, and the 120 unused bytes.
- The register banks are only partially mapped.
