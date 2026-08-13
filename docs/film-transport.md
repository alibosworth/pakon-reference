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
[CONFIRMED on hardware, August 2026] observed directly: a `0xA2` disengage
without first restoring the idle speed left the motor running, and writing the
idle speed stopped it.

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

Film presence and position are read through the 30-byte ReadSensorData (0x90)
sensor state, which the idle loop polls. The scanner detects insertion by the
change in sensor values, and the scan runs until a film-end condition rather
than a fixed line count. [DOCUMENTED] that the OEM scan loop stops on a
device-signalled end rather than a byte count; the exact film-present bit
within the 30-byte structure is not pinned down here.

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
