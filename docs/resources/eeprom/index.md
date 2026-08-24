# EEPROM dumps

Decoded per-unit EEPROM contents from every F-X35 unit read so far, side
by side, with the raw dumps for each. The point of collecting them is that
a single unit cannot tell you which fields a factory calibration actually
measured, which are per-model defaults, and which are constant across the
whole family, and that distinction is what you need in order to know what
a dump is worth, what a fault looks like, and what would actually have to
be restored.

The layout these are decoded against is documented in
[calibration.md](../../calibration.md#layout). How to read your own chip is
on the [per-unit data and safety](../../per-unit-data-and-safety.md) page.

## The units

| Unit | Type (`0x00C`) | Model | Chip health | Read by |
|---|---|---|---|---|
| [2233](F135-2233/index.md) | 1350 | F-135 | all four copies valid | Mats Fagerberg ([thetalkingdrum](https://github.com/thetalkingdrum)) |
| [5963](F135plus-5963/index.md) | 1351 | F-135 Plus | all four copies valid | Mats Fagerberg ([thetalkingdrum](https://github.com/thetalkingdrum)) |
| 16275 | 1351 | F-135 Plus | partial dump, CRC unverifiable | [pakon-mac](https://github.com/gazzdingo/pakon-mac) |
| [16402](F135plus-16402/index.md) | 1351 | F-135 Plus | section A primary fails CRC | Ali Bosworth ([alibosworth](https://github.com/alibosworth)) |
| [17157](F135plus-17157/index.md) | 1351 | F-135 Plus | all four copies valid | Mats Fagerberg ([thetalkingdrum](https://github.com/thetalkingdrum)) |

Unit 16275 is not reproduced here. It is a 256-byte raw I2C read covering
chip bytes `0x001`–`0x0FF` only, offset by one byte (its first byte, the low
byte of the section length, is missing), and it carries no CRC, no section A
tail and no section B. It decodes cleanly once realigned, so its values are
included in the tables below, but the file itself lives in
[that project's repository](https://github.com/gazzdingo/pakon-mac). Its
final byte differs from every other unit, which is consistent with the
read-degradation behaviour that project documents rather than a real
difference.

## Section A, field by field

`Offset` / `MotorSpeed` / `MotorSpeed (IR)` per resolution base:

| Unit | Base 4 | Base 8 | Base 16 |
|---|---|---|---|
| 2233 (F-135) | 27 / 8162 / 6119 | 54 / 3627 / 2720 | 54 / 2325 / 1530 |
| 5963 (F-135+) | 29 / 25512 / 19117 | 58 / 11338 / 7494 | 59 / 5850 / 4796 |
| 16275 (F-135+) | 27 / 25802 / 19335 | 54 / 11467 / 7580 | 55 / 5917 / 4850 |
| 16402 (F-135+) | 30 / 25726 / 19278 | 58 / 11434 / 7557 | 60 / 5900 / 4836 |
| 17157 (F-135+) | 34 / 25498 / 19107 | 66 / 11332 / 7490 | 68 / 5847 / 4793 |

NegMatrix, reduced to the part that varies (the quadratic and cross-term
columns are ≈ 0 on every unit, so each is in effect a 3×4 affine):

| Unit | Diagonal | Constants |
|---|---|---|
| 2233 (F-135) | 0.27680 / 0.27967 / 0.26158 | 163.4 / 441.4 / 651.0 |
| 5963 (F-135+) | 0.28383 / 0.32009 / 0.33758 | 145.6 / 386.7 / 602.9 |
| 16275 (F-135+) | 0.28920 / 0.27580 / 0.27820 | 159.6 / 444.8 / 635.5 |
| 16402 (F-135+) | 0.29299 / 0.28521 / 0.32000 | 165.8 / 429.8 / 638.2 |
| 17157 (F-135+) | 0.28860 / 0.31628 / 0.32606 | 162.1 / 401.3 / 607.7 |

## Per-unit, per-model, and constant fields

**Per-unit.** The serial, the three `Offset` words, the six motor-speed
words, and NegMatrix. These are the fields a factory calibration actually
measured, and they are the reason a dump of your own unit is worth having.
Motor speeds vary about 1% between units of the same model; `Offset` varies
more than expected: 27 to 34 at base 4, and 54 to 68 at base 16, a spread
of 14 px on a 2000 px line.

**Per-model.** Section B (below) and the scanner type at `0x00C`. Also the
motor speeds, which differ by roughly 3.1× between the base F-135 and the
Plus at bases 4 and 8, a mechanical difference between the models, not a
per-unit one. The ratio of IR to normal motor speed within a unit is
another: identical on all three Plus units (0.749 / 0.661 / 0.820 at bases
4 / 8 / 16) and different on the base F-135 (0.750 / 0.750 / 0.658), which
suggests these ratios are design constants rather than measurements.

**Constant across everything read.** PosMatrix, the plain 0.25 diagonal,
bit-identical on all four complete dumps and across both models, so a
shared constant and not a factory measurement. The revision word at `0x008`,
400 everywhere.
The 120-byte tail at `0x116`–`0x18D`, zero on every complete dump.

## Section B

Twelve motor-adjust words and a trailing `u32`. It is **identical on all
three F-135 Plus units** (CRC `0x873e6ed3`, words alternating 1000 / 1008)
and differs on the one base F-135 read in exactly one payload byte: the
third word is `0x03F0` where the Plus has `0x03E8`, giving CRC
`0x2a582d50`.

So section B looks like a factory default scoped per model. Either the
OEM's motor-speed calibration has never been run on any unit read so far,
or it does not write here. The base-model reading rests on one unit; more
base F-135 dumps would settle it.

## Read your own unit

The quickest route is
**[pakon-eeprom-backup-web](https://alibosworth.github.io/pakon-eeprom-backup-web/)**,
which does the whole procedure from a Chrome or Edge tab with nothing to
install: both copies of both sections, all four CRCs checked, primary
compared with backup, and a zip of the results. It reads only. The
procedure it follows, the two vendor requests involved, the command-line
alternatives, and the reasons to read both copies are on the
[per-unit data and safety](../../per-unit-data-and-safety.md) page.

Do it while your scanner works. A chip fault is silent: unit
[16402](F135plus-16402/index.md) has a corrupted primary copy of section A
that the OEM engine has been quietly routing around for years, and nothing
in the software ever said so. The only way to know is to read both copies
and check the CRCs yourself.

## Using someone else's dump

These files are published so the decode can be checked and the spread can
be seen, not as a substitute for your own backup. But if a chip is already
lost, a donor dump from the **same model** is a reasonable last resort, and
better than leaving it blank: when both copies of a section fail, the OEM
engine does not stop: it raises a warning and runs on whatever it read, so
a blank or corrupt chip means undefined motor speeds and an undefined CCD
window, which is worse than slightly wrong ones.

Roughly what a same-model donor would cost you. These figures are the
differences **observed between the few units read so far** (four complete
dumps, only three of them the same model), so treat them as an indication
of scale, not as bounds. A donor could be further off than anything in this
table, and how much a given error matters in practice has not been tested
on hardware.

| | Difference seen between units | Likely consequence |
|---|---|---|
| `Offset` | up to 14 px of 2000 | image shifted along the frame height, under 1% on this sample |
| MotorSpeed | about 1% within a model | aspect-ratio error of roughly the same order |
| NegMatrix | up to about 20% on a channel | a colour cast, correctable downstream |
| Serial | wrong | cosmetic; appears in logs and the registry |
| PosMatrix, section B | identical on every unit read, if the model matches | none expected |

The last row is the one most likely to change as more units are read:
"identical on every unit read" is a statement about four dumps, not a
guarantee about the family.

Match the scanner type at `0x00C` (1350 vs 1351) before considering it:
that is the practical reason the field matters. Using a base F-135's
section A on a Plus would set motor speeds to about a third of correct,
which is not a cosmetic error. Re-run Light Correction afterwards, since
that calibration is per-unit, lives in the registry rather than the EEPROM,
and will be measured fresh on the actual hardware.

Two cautions about the write itself, which are the real risk rather than
the donor values. It needs the vendor `0xA2` request, which no tool
published here issues. And it is irreversible: writing a donor over a chip
that still had one valid copy destroys your unit's true values permanently
if you never dumped them. Read your own chip first, whatever else you do:
if it still validates, you never need any of this. See the
[safety page](../../per-unit-data-and-safety.md) before going near a write.

## Contributing a dump

More units, and especially more base F-135 units, would firm up the
per-model claims. Section B in particular rests on a single base-model
dump. Both copies of both sections please, with the CRC results, and say
whether the repeat read followed a power cycle.

All dumps here were taken with the OEM engine's own read sequence,
read-only, using `tools/eedump.py` from
[pakon-tlx-macos](https://github.com/pablonavarrob/pakon-tlx-macos).

## Layout of a unit folder

Mirrors the archive these come from:

```
<model>-<serial>/
  index.md                             what this unit's data showed
  eeprom/
    eeprom_0x52_sectionA_primary.bin   section A, primary copy (0x000)
    eeprom_0x52_sectionA_backup.bin    section A, backup copy (0x400)
    eeprom_0x52_sectionB_primary.bin   section B, primary copy (0x800)
    eeprom_0x52_sectionB_backup.bin    section B, backup copy (0xA00)
    eeprom_0x51_boot_personality.bin   boot EEPROM, where read
    (any extra reads, named with read number and power cycle)
    SHA256SUMS
```
