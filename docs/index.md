# Pakon F-X35 Reference

An implementation-agnostic reference for how the Kodak/Pakon **F-X35** film
scanners (F-135, F-135+, F-235, F-335) actually work: USB identity and firmware
loading, the PPB command protocol, image stream formats, calibration, the
colour pipeline, film transport, and the differences between the models.

This documents **facts about the hardware and its protocol**, so that any
implementation (driver, scanning tool, decoder) can be built or checked
against it. It is not a driver, not a scanning application, and it contains no
Kodak software, firmware, or data files.

## Why this exists

The knowledge needed to drive these scanners has been re-derived from scratch
by every project that attempted it, because no neutral reference existed. The
scanners are long out of support, the film community depends on them, and the
facts about how they work should be public and free.

## Provenance and timeline

This is a distilled, redacted public view of a private reverse-engineering
corpus built since March 2026: analysis of the original Windows software for
interoperability, USB and driver-level captures of real hardware, firmware
disassembly, and live experiments on physical scanners. The corpus itself
cannot be published because it contains Kodak software artifacts, firmware,
and decompilation output; this reference carries only the derived facts. Each document is dated
to when the underlying research established it:

- **March 8, 2026**: first findings (USB/PPB protocol, driver architecture,
  the colour pipeline, Digital ICE)
- **March 13-19, 2026**: image stream and planar format, DX barcode tables,
  ICC profiles
- **April 26-27, 2026**: driver-level captures of a real F-135+ across four
  resolution/IR configurations
- **May 2, 2026**: firmware inventory, F-135+ PICL Plus disassembly, IOCTL
  capture decode; first live driving of the F-135+ (via Stefan Dierauf's
  libpakon)
- **June-July 2026**: the fixed-pattern-correction constant; the exact ColNeg
  inversion formula
- **August 12-13, 2026**: the F-135+ driven end to end in the
  [pakon-macos](https://github.com/jorshhh/pakon-macos) project, verifying
  much of the corpus on hardware and adding the reply direction,
  per-resolution strides, and motor behaviour

## Where to start

- New to the protocol? Read [USB identity & firmware](usb-identity-and-firmware.md)
  then the [PPB protocol](ppb-protocol.md).
- Decoding a scan? See the [image stream](image-stream.md) and the
  [colour pipeline](color-pipeline.md).
- Working out model differences? The
  [scanner family](scanner-family.md) page is the map.

## Related projects and credit

Independently licensed projects; facts verified in or contributed by them are
credited inline. This reference documents facts, not their code, so their
software licenses do not extend to it.

- [FX35](https://github.com/ktkaufman03/FX35) by Kai Kaufman: modernised
  open-source Windows driver; the driver-level captures used here were made
  with a logging-instrumented build of it.
- libpakon by Stefan Dierauf: an independent C++ driver and "Pakon Studio"
  app (private); the marker-bit row-origin technique was shown to this author
  by its author.
- [pakon-macos](https://github.com/jorshhh/pakon-macos) by Jorge Rangel
  (AGPL-3.0): open-source cross-platform driver and web app; the F-135 and
  F-135+ were verified working here, and the hardware-only facts in these
  pages were derived by driving a real scanner with it.

## Conventions and license

## Not comprehensive

This reference documents what has been established, and no more. It is not a
complete description of these scanners: reply semantics for most commands,
several payload layouts, the parameter table's fields, the F-235/F-335
protocol, and the OEM's film-specific processing are among the known gaps.
Each page ends with its open questions, so absence of a fact here means it is
not yet established, not that it does not matter.

Claims carry confidence markers (`[CONFIRMED]` / `[DOCUMENTED]` /
`[INFERRED]`) and sources; see [Conventions](CONVENTIONS.md) for how they are
applied. Licensed [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/):
the point of this reference is that nobody should have to pay for, or
re-derive, these facts.
