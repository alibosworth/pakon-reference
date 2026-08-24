# F-135+, serial 17157

Read by Mats Fagerberg ([thetalkingdrum](https://github.com/thetalkingdrum)), 24 August 2026. Everything below
is what this particular unit showed; see the
[dump comparison](../index.md) for how it sits against the others.

## Decoded contents

| Field | Value |
|---|---|
| Scanner type (`0x00C`) | 1351 = F-135 Plus |
| Serial | 17157 |
| DpiBase4_35: Offset / MotorSpeedPlus / MotorSpeedPlus_Ir | 34 / 25498 / 19107 |
| DpiBase8_35 | 66 / 11332 / 7490 |
| DpiBase16_35 | 68 / 5847 / 4793 |
| NegMatrix (3×10, per row: R G B R² G² B² RG GB BR const) | diag 0.28860 / 0.31628 / 0.32606, consts 162.075 / 401.272 / 607.741, quadratic terms ~0 |
| PosMatrix | 0.25 diagonal, no cross terms or offsets |
| MotorAdjust ×12 | 1000, 1008 alternating |

This unit has the **highest `Offset` values of any unit read**: 34 / 66 /
68, against 27–30 / 54–58 / 55–60 everywhere else. If `Offset` is the
per-unit start of the imaged region along the CCD line, as
[calibration.md](../../../calibration.md) infers, then this unit sits about
14 px further into the sensor at base 16 than the lowest one read. It
widens the known spread considerably, and any implementation that hardcodes
a window start rather than reading this word will be furthest wrong here.

## Chip health

Clean, and the only fully healthy F-135 Plus read so far. All four copies
validate and the two reads of section A's primary are byte-identical.

| Copy | CRC | Result |
|---|---|---|
| Section A primary | `0x96cec048` | valid |
| Section A backup | `0x96cec048` | valid, byte-identical to primary |
| Section B primary | `0x873e6ed3` | valid |
| Section B backup | `0x873e6ed3` | valid, byte-identical to primary |

## What this unit established

Section B is byte-identical to 5963's and 16402's, same CRC `0x873e6ed3`.
Having a second independently-read F-135 Plus to compare against is what
allowed section B to be scoped as a per-model default rather than assumed
universal across the family. The base F-135
([2233](../F135-2233/index.md)) differs from all three Plus units in one
payload byte.

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
