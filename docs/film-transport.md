# Film transport

Film is pulled past the fixed linear CCD by a drive motor under the motor
controller (PICM). One motor step produces one scan line, so transport speed
sets the along-film sampling and the scan resolution is a function of motor
speed and CCD line rate.

_Reconstructed from the PPB debug trace and the OEM motor-control UI,
March 2026. Motor commands are in [command-reference.md](command-reference.md);
addresses are 0x24 (F-135) / 0x44 (F-135+)._

## Engage, run, stop

The OEM software runs the transport in this order:

1. SetMotorSpeed (`0xA5`): the run speed.
2. MotorForward (`0xA0`), or MotorReverse (`0xA1`).
3. The film moves.
4. Acquire off: bank `0x82` reg 0 bit 0 clear.
5. MotorStop (`0xA2`).

[DOCUMENTED] from the trace. The names are from `TLB.dll` 3.1.0.28.

`0xA2` does not stop the motor while the CCD acquires. Clear the acquire bit
before you send `0xA2`. [CONFIRMED on hardware, August 2026, open-pakon:
https://github.com/jorshhh/open-pakon] The OEM teardown also clears acquire
first.

## Speed

`0xA5` takes a u16. The OEM software computes it from the per-unit EEPROM:
the motor speed for the mode, times the adjust word, divided by 1000. It
clamps the result to 400–9500 on the F-135 and 1000–32766 on the F-135+.
[DOCUMENTED] in `TLB.dll`. The values in the captures run from 1557 to 25726.
[CONFIRMED in captures] These values are not tenths of mm/s. The mapping from
the OEM UI's advance setting (10–355) to `0xA5` is not known. The scanner
reverses film only when the host sends `0xA1`. A scan never sends `0xA1` or a
motor speed of 0. [CONFIRMED in captures, F-135+]

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

## Open questions

- The film-present indication within the sensor state is not pinned down.
- The advance duration/distance parameter encoding is not decoded.
- The exact end-of-roll signalling is not documented.
