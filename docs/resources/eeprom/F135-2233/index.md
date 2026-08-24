# F-135, serial 2233

Read by Mats Fagerberg ([thetalkingdrum](https://github.com/thetalkingdrum)), 24 August 2026. Everything below
is what this particular unit showed; see the
[dump comparison](../index.md) for how it sits against the others.

This is the only **base F-135** read so far; every other unit is an F-135
Plus. That makes it the reference point for which fields are per-model
rather than per-unit.

## Decoded contents

| Field | Value |
|---|---|
| Scanner type (`0x00C`) | 1350 = F-135 (the Plus reads 1351) |
| Serial | 2233 |
| DpiBase4_35: Offset / MotorSpeedPlus / MotorSpeedPlus_Ir | 27 / 8162 / 6119 |
| DpiBase8_35 | 54 / 3627 / 2720 |
| DpiBase16_35 | 54 / 2325 / 1530 |
| NegMatrix (3×10, per row: R G B R² G² B² RG GB BR const) | diag 0.27680 / 0.27967 / 0.26158, consts 163.380 / 441.381 / 650.963, quadratic terms ~0 |
| PosMatrix | 0.25 diagonal, no cross terms or offsets |
| MotorAdjust ×12 | 1000, 1008, **1008**, 1008, then 1000 / 1008 alternating |

The motor speeds are roughly a third of an F-135 Plus's at bases 4 and 8.
That is the mechanical difference between the models, not anything
per-unit.

## Chip health

Clean. All four copies validate and the two reads of section A's primary
are byte-identical.

| Copy | CRC | Result |
|---|---|---|
| Section A primary | `0x86181423` | valid |
| Section A backup | `0x86181423` | valid, byte-identical to primary |
| Section B primary | `0x2a582d50` | valid |
| Section B backup | `0x2a582d50` | valid, byte-identical to primary |

## What this unit established

Section B is **not** the same on every scanner in the family. This unit's
section B differs from all three F-135 Plus units in exactly one payload
byte: the third motor-adjust word is `0x03F0` where the Plus has `0x03E8`,
so section B looks like a factory default scoped per model rather than a
family-wide constant. That reading currently rests on this single
base-model unit; more would confirm it.

The scanner type at `0x00C` was identified from this unit alongside 5963,
and cross-checked against the OEM client's own error log.

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
