# Pakon F-X35 scanner reference

An implementation-agnostic reference for how the Kodak/Pakon F-X35 film
scanners (F-135, F-135+, F-235, F-335) actually work: USB identity and
firmware loading, the PPB command protocol, image stream formats, calibration,
the colour pipeline, film transport, and the differences between the models.

This is documentation of **facts about the hardware and its protocol**,
written so that any implementation (driver, scanning tool, decoder) can be
built or checked against it. It is not a driver, not a scanning application,
and it contains no Kodak software, firmware, or data files.

## Why this exists

The knowledge needed to drive these scanners has been re-derived from scratch
by every project that attempted it, because no neutral reference existed. The
scanners are long out of support, the film community depends on them, and the
facts about how they work should be public and free.

## Provenance and timeline

This is a **distilled, redacted public view of a private reverse-engineering
corpus** built by Ali Bosworth since March 2026: analysis of the original
Windows software (TLXClientDemo, TLA/TLB/TLC, PakonIMAu) for interoperability,
USB and driver-level captures of real hardware, firmware disassembly, and live
experiments on physical scanners. The private corpus cannot be published
because it contains Kodak software artifacts, decompilation output, and
firmware; this repository carries only the derived facts, and each document is
dated to when the underlying research established it.

Timeline of the underlying work:

- **March 8, 2026**: first findings: USB/PPB protocol, driver architecture,
  the Ansel colour pipeline, Digital ICE.
- **March 13–19, 2026**: image stream / planar format, DX barcode tables,
  ICC profiles and colour spaces.
- **April 26–27, 2026**: driver-level captures of a real F-135+
  (serial 16402) across four resolution/IR configurations (the capture files
  embed their own timestamps).
- **May 2, 2026**: firmware file inventory, F-135+ PICL Plus disassembly,
  IOCTL capture decode. Stefan Dierauf granted access to his libpakon project
  (a private codebase); the F-135+ was first driven live using it, moving the
  work from analysis of captures to interacting with the scanner directly.
- **August 12–13, 2026**: the F-135+ driven end to end in the pakon-macos
  project by Jorge Rangel (https://github.com/jorshhh/pakon-macos), verifying
  much of the corpus on live hardware and settling several open questions.
  Beyond verification, driving
  the scanner produced genuinely new facts that transmit-only captures could
  not: the **reply (RX) direction** of the protocol (reply-frame structure, the
  `0x88` READ marker byte, first-open-only handshake replies), the
  **per-resolution image strides** and that IR-off scans carry no IR lane, and
  the **motor run/stop register** behaviour. Facts confirmed on hardware are
  marked and dated accordingly.

Cross-confirmation with independent projects (pakon-macos, the FX35 Windows
driver, and Stefan Dierauf's libpakon) is cited throughout where it exists.
Facts verified on
real hardware in August 2026 are marked `[CONFIRMED on hardware, August 2026]`.
Every fact carries a confidence marker and a source; see
[CONVENTIONS.md](docs/CONVENTIONS.md).

## Contents

| Document | Covers |
|---|---|
| [docs/usb-identity-and-firmware.md](docs/usb-identity-and-firmware.md) | Cold/warm enumeration, the personality mechanism, FX2 firmware selection and loading |
| [docs/ppb-protocol.md](docs/ppb-protocol.md) | The command frame format, packet types, status codes, bus addresses, bridge open |
| [docs/command-reference.md](docs/command-reference.md) | Per-controller command sets (light/CCD, motor, host) with payloads and sequences |
| [docs/image-stream.md](docs/image-stream.md) | The bulk image stream: chunking, row strides, channel interleave, IR lane, marker bit |
| [docs/calibration.md](docs/calibration.md) | Register banks, dark/bright calibration, the fixed-pattern-correction formula |
| [docs/color-pipeline.md](docs/color-pipeline.md) | The C-41 inversion (ColNeg), film-base normalisation, rendering stages |
| [docs/film-transport.md](docs/film-transport.md) | Motor control, advance/eject sequences, the run/stop register, film sensing |
| [docs/dx-barcode.md](docs/dx-barcode.md) | The DX film barcode subsystem across the model family |
| [docs/scanner-family.md](docs/scanner-family.md) | What differs between the F-135, F-135+, F-235, and F-335 |

## Related projects and credit

Independently licensed projects; facts verified in or contributed by them are
credited inline. This reference documents *facts* (not their code), so their
software licenses do not extend to it.

- [FX35](https://github.com/ktkaufman03/FX35) by Kai Kaufman: modernised
  open-source Windows driver; the driver-level captures used here were made
  with a logging-instrumented build of it.
- libpakon by Stefan Dierauf: an independent C++ driver and "Pakon Studio"
  app (private); the marker-bit row-origin technique was shown to this author
  by its author.
- [pakon-macos](https://github.com/jorshhh/pakon-macos) by Jorge Rangel
  (AGPL-3.0): open-source cross-platform driver and web app; the F-135 and
  F-135+ were verified working here, and the hardware-only facts above were
  derived by driving a real scanner with it.

## License

[CC BY 4.0](LICENSE): use it for anything, including commercial work, with
attribution. The point of this repository is that nobody should have to pay
for, or re-derive, the facts.
