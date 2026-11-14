# Film transport

Film is pulled past the fixed linear CCD by a drive motor under the motor
controller (PICM). One motor step produces one scan line, so transport speed
sets the along-film sampling and the scan resolution is a function of motor
speed and CCD line rate.

_Reconstructed from the PPB debug trace and the OEM motor-control UI,
March 2026. Motor commands are in [command-reference.md](command-reference.md);
addresses are 0x24 (F-135) / 0x44 (F-135+)._

## Engage, run, stop

A transport run is:

1. `SetMotorCalibration` (0xA5): home/adjust parameters
2. `EngageFilmDrive` (0xA0): start the drive
3. `SetMotorSpeed` (0x82): set the run speed (16-bit tenths of mm/s + mode)
4. ... motion ...
5. return the speed register to its idle value
6. `DisengageFilmDrive` (0xA2): release the drive

[DOCUMENTED] from the advance and scan sequences in the trace.

An important behavioural detail: the run/stop state is carried by the speed
register (the sub-register-0 `SetMotorSpeed` value), **not** by
DisengageFilmDrive alone. Disengaging releases the drive clutch; if the speed
register is left at a running value the motor keeps turning. A clean stop
returns the speed register to its idle value first, then disengages.
[CONFIRMED on hardware, August 2026, in the pakon-macos project by Jorge
Rangel: https://github.com/jorshhh/pakon-macos] observed directly: a `0xA2`
disengage without first restoring the idle speed left the motor running, and
writing the idle speed stopped it. This causal fact was not visible in the
transmit-only OEM captures; it was learned by driving the transport.

## Speed

Advance speed is a 16-bit value in tenths of mm/s. The OEM UI allows forward
10–355 (1.0–35.5 mm/s) and reverse −10 to −355; 0 is invalid. The scanner does
not reverse film unless explicitly commanded. [DOCUMENTED] from the OEM UI and
the trace.

## Frame advance vs. continuous scan

Two distinct motions:

- **Frame advance**: move the film a fixed amount without scanning. Each step
  engages the drive, runs the transport for a written duration/distance
  parameter (roughly one frame's worth of travel), polls the status until that
  motor move completes, then stops. This is an **open-loop, distance-based
  move**: the controller does not sense where a frame lies; it just moves the
  film the requested amount. [DOCUMENTED] from the advance sequence.
- **Continuous scan**: engage once and run steadily while the CCD acquires
  line by line, until end-of-roll. [DOCUMENTED] from the scan sequence.

## Film sensing and end-of-roll

Film is sensed by the two DX sensors (see [dx-barcode.md](dx-barcode.md)).
The host reads them two ways: as flags in the `0x90` sensor state, and as raw
levels in register `0x93`.

**Flags.** The high nibble of the first `0x90` entry's type byte: `0x20` film
at the entry sensor, `0x10` at the exit sensor, `0x40` emulsion down, `0x80`
tail first. [CONFIRMED: named in `TLB.dll` 3.1.0.28 (sha256 5866ec56…), seen
in F-135 captures] A full roll runs `0` → `2` → `3` → `1` → `0`. Emulsion
down has not been seen.

**Levels.** `0x93` bytes 0–1 are the entry sensor, 2–3 the exit sensor.
[CONFIRMED on two units, November 2026, open-pakon:
https://github.com/jorshhh/open-pakon]

| | Empty | Under film | Sensor spacing |
|---|---|---|---|
| F-135, serial 3054 | 190 / 216 / 224 / 202 | below 170 | 1.4 s at speed 9500 |
| F-135+, serial 138 | 244 / 242 / 246 / 242 | 45–125 | 0.6 s at speed 25381 |

Under film, at least one byte of each pair drops; the DX code makes it
flicker. Levels are per unit. The strip stops under the exit sensor after it
leaves the rollers and must be pulled out by hand.

**End of roll.** The scanner never ends the scan. With acquisition on it keeps
streaming open-gate data after the film, or with no film at all.
[CONFIRMED on both units]

The host decides. The OEM watches the line level: film starts below
`DetectFilm_G` and ends back above `DetectWhite_G`, both rebased from the
first three lines. `NoFilmTimeOut` bounds the wait. [CONFIRMED in `TLB.dll`,
not yet seen in an OEM run; green channel INFERRED from the names]

*Reversed, November 2026:* this page said the scan stopped on a
device-signalled end.

## Frame boundary detection

Frame boundaries are **not** sensed by the transport hardware: the motor
controller has no notion of where a frame is (see frame advance above). They
are determined **in software, from the scan data**: the OEM analyses the image
stream for the density transitions between exposures. [DOCUMENTED] the OEM
framing works on the scan data, not a hardware sensor.

Because framing is a software determination from image content, it is
independent of the transport: advance moves a fixed distance, and where the
frames actually fall is worked out afterwards from what was scanned. See also
[image-stream.md](image-stream.md).

## Open questions

- The advance duration/distance parameter encoding is not decoded.
- The Film Track Test's margin below the empty level has not been found.
- Emulsion down (`0x40`) has not been observed on hardware.
- How the OEM computes `DetectFilm_G` and `DetectWhite_G`.
