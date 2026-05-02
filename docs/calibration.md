# Calibration

Before scanning, the OEM programs the CCD (gain, offset, per-channel exposure)
and computes a per-column correction from calibration passes. This is what
makes the raw stream usable: it normalises the lamp profile and each sensor
element's response.

_From the OEM driver and Ansel pipeline reverse engineering, and the parameter
table read seen in captures, March–May 2026._

## The parameter table read

Immediately after the open handshake, the OEM reads a structured
parameter/calibration table over USB control transfers: vendor request `0xA4`
triggers, `0xA9` reads, pulling the table in fixed 32-byte chunks at increasing
offsets. [DOCUMENTED] from captures: a fixed number of these pairs, resolution
independent. The table holds device capability/calibration constants; the full
field layout is not decoded here.

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

## Colour vs. calibration

This CCD-level calibration (gain/offset/exposure/fixed-pattern) is distinct from
the colour negative inversion. Calibration makes the raw linear CCD values
correct and uniform; the C-41 inversion (LUT + matrix) is a separate, later
stage documented in [color-pipeline.md](color-pipeline.md).
