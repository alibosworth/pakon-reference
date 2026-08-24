# F-135+, serial 5963

Read by Mats Fagerberg ([thetalkingdrum](https://github.com/thetalkingdrum)), 23 August 2026. Everything below
is what this particular unit showed; see the
[dump comparison](../index.md) for how it sits against the others.

## Decoded contents

| Field | Value |
|---|---|
| Scanner type (`0x00C`) | 1351 = F-135 Plus |
| Serial | 5963 |
| DpiBase4_35: Offset / MotorSpeedPlus / MotorSpeedPlus_Ir | 29 / 25512 / 19117 |
| DpiBase8_35 | 58 / 11338 / 7494 |
| DpiBase16_35 | 59 / 5850 / 4796 |
| NegMatrix (3×10, per row: R G B R² G² B² RG GB BR const) | diag 0.28383 / 0.32009 / 0.33758, consts 145.573 / 386.672 / 602.873, quadratic terms ~0 |
| PosMatrix | 0.25 diagonal, no cross terms or offsets |
| MotorAdjust ×12 | 1000, 1008 alternating |

## Chip health

Clean. All four copies validate and the two reads of section A's primary
are byte-identical.

| Copy | CRC | Result |
|---|---|---|
| Section A primary | `0x6b061a7a` | valid |
| Section A backup | `0x6b061a7a` | valid, byte-identical to primary |
| Section B primary | `0x873e6ed3` | valid |
| Section B backup | `0x873e6ed3` | valid, byte-identical to primary |

This unit has a hardware fault, but it is on the **LED board**, not the
EEPROM; the chip reads clean and CRC-verified as above. Worth recording as
a counter-example in both directions: a scanner can be visibly broken while
its irreplaceable data is perfectly intact, just as
[16402](../F135plus-16402/index.md) works normally while carrying a
corrupted copy.

## What this unit established

The OEM client's error log for this unit prints "Scanner Type 1351",
matching the decoded value at `0x00C` exactly. That is what turned the word
at `0x00C` from an unnamed scalar into a confirmed model discriminator, and
it means the model can be read from a raw dump without running the OEM
software at all.

## Files

| File | Size | Notes |
|---|---|---|
| [`eeprom_0x52_sectionA_primary.bin`](eeprom/eeprom_0x52_sectionA_primary.bin) | 398 B | CRC valid |
| [`eeprom_0x52_sectionA_backup.bin`](eeprom/eeprom_0x52_sectionA_backup.bin) | 398 B | CRC valid, identical to primary |
| [`eeprom_0x52_sectionB_primary.bin`](eeprom/eeprom_0x52_sectionB_primary.bin) | 36 B | CRC valid |
| [`eeprom_0x52_sectionB_backup.bin`](eeprom/eeprom_0x52_sectionB_backup.bin) | 36 B | CRC valid, identical to primary |
| [`eeprom_0x52_sectionA_primary_read2.bin`](eeprom/eeprom_0x52_sectionA_primary_read2.bin) | 398 B | second read, identical |
| [`SHA256SUMS`](eeprom/SHA256SUMS) | | hashes for the above |

No boot personality (`0x51`) read for this unit. Whether the second read
followed a power cycle is not recorded. Read on macOS via
[pakon-tlx-macos](https://github.com/pablonavarrob/pakon-tlx-macos)
(`tools/eedump.py`).
