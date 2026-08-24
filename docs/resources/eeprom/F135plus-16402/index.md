# F-135+, serial 16402

Read by Ali Bosworth ([alibosworth](https://github.com/alibosworth)), 17 August 2026. This is the reference unit most of
the rest of this documentation was verified against. Everything below is
what this particular chip showed; see the [dump comparison](../index.md)
for how it sits against the others.

## Decoded contents

| Field | Value |
|---|---|
| Scanner type (`0x00C`) | 1351 = F-135 Plus |
| Serial | 16402 |
| DpiBase4_35: Offset / MotorSpeedPlus / MotorSpeedPlus_Ir | 30 / 25726 / 19278 |
| DpiBase8_35 | 58 / 11434 / 7557 |
| DpiBase16_35 | 60 / 5900 / 4836 |
| NegMatrix (3×10, per row: R G B R² G² B² RG GB BR const) | diag 0.29299 / 0.28521 / 0.32000, consts 165.785 / 429.782 / 638.181, quadratic terms ~0 |
| PosMatrix | 0.25 diagonal, no cross terms or offsets |
| MotorAdjust ×12 | 1000, 1008 alternating |

Boot chip (`0x51`): `c0 05 0f 35 f2 07 aa 04` then `02`, the standard
F-135 / F-135+ bytes, which are the same on every unit and are replaceable.
The first 8 are the personality record itself; see
[usb-identity-and-firmware.md](../../../usb-identity-and-firmware.md#the-personality-mechanism).

## The fault in this chip

**Section A's primary copy has one corrupted byte.** Offset `0x0A5` reads
`0x48` where the backup has `0x00`. It is the high byte of the float
`PosMatrix1`, so that value reads 131072.0 in the primary and 0.0 (correct)
in the backup.

| Copy | Stored CRC | Computed | Result |
|---|---|---|---|
| Section A primary | `0xa9a14ed0` | `0x76970f5b` | **fails** |
| Section A backup | `0xa9a14ed0` | `0xa9a14ed0` | valid, use this one |
| Section B primary | `0x873e6ed3` | `0x873e6ed3` | valid |
| Section B backup | `0x873e6ed3` | `0x873e6ed3` | valid, identical to primary |

The fault is stable across four reads and two power cycles (the three
saved reads of section A's primary are byte-identical), so it is a
stored-data fault, not a read artifact. It touches only the slide
(positive film) matrix.

**This unit is part of why the reference insists on reading both copies.**
The OEM software handles the fault silently: it reads the primary, finds
the CRC bad, reads the backup, uses that, and shows no warning. The
registry export from this unit holds `PosMatrix1 = 0.000000`, which is the
backup's value, confirming that is what it did. So the scanner has worked
normally for years with a corrupted copy on the chip, and nothing in the
software would ever have said so. A dump of the primaries alone could not
have revealed it either. Only reading both copies and checking the CRCs
did.

It is not the only one. Unit 16275 also has a damaged section A primary and
a good backup, with different damage: a corrupted length byte plus a
17-byte block at a page boundary. Two of the five units read are in this
state, so it is worth treating as a normal thing to find rather than bad
luck. See [the comparison](../index.md#the-units).

If restoring this chip, write the **backup** copy of section A, not the
primary. Section B is identical in both copies.

## Files

| File | Size | Notes |
|---|---|---|
| [`eeprom_0x52_sectionA_primary.bin`](eeprom/eeprom_0x52_sectionA_primary.bin) | 398 B | **CRC bad** (byte `0x0A5`), kept as the record of the fault |
| [`eeprom_0x52_sectionA_backup.bin`](eeprom/eeprom_0x52_sectionA_backup.bin) | 398 B | CRC valid, the good copy |
| [`eeprom_0x52_sectionB_primary.bin`](eeprom/eeprom_0x52_sectionB_primary.bin) | 36 B | CRC valid |
| [`eeprom_0x52_sectionB_backup.bin`](eeprom/eeprom_0x52_sectionB_backup.bin) | 36 B | CRC valid, identical to primary |
| [`eeprom_0x52_sectionA_primary_read1_cycle1.bin`](eeprom/eeprom_0x52_sectionA_primary_read1_cycle1.bin) | 398 B | earlier read, power cycle 1, identical |
| [`eeprom_0x52_sectionA_primary_read2_cycle2.bin`](eeprom/eeprom_0x52_sectionA_primary_read2_cycle2.bin) | 398 B | earlier read, power cycle 2, identical |
| [`eeprom_0x51_boot_personality.bin`](eeprom/eeprom_0x51_boot_personality.bin) | 256 B | the boot chip; the personality record is its first 8 bytes |
| [`SHA256SUMS`](eeprom/SHA256SUMS) | | hashes for the above |

A registry export from this unit (the OEM's own decode of the EEPROM plus
the light calibration it measured) exists in the private archive but is
not reproduced here; what it contains is described on the
[per-unit data and safety](../../../per-unit-data-and-safety.md) page.

Read on macOS via
[pakon-tlx-macos](https://github.com/pablonavarrob/pakon-tlx-macos), with
the OEM stack (tlx.dll / TLB.dll 3.1.0.28) under Wine and the scanner
running the vendor `Pakon7.hex` firmware.
