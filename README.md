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
Windows software (TLXClientDemo, the per-model engine DLLs TLA/TLB/TLC,
PakonIMAu) for interoperability,
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
- **August 14–17, 2026**: the reference compared against the other open
  Pakon clients (pakon-mac, pakon-macos, pakon-tlx-macos, psix, PakonClient)
  and against the shipped OEM binaries directly. Settled which OEM engine DLL
  serves which model (`TLB.dll` for the F-135 and F-135+, `TLA.dll` for the
  F-235, `TLC.dll` for the F-335), from `tlx.dll`'s own dispatch and the
  engines' bus-address tables; confirmed by decompiling `TLB.dll` that the
  135-line engine resolves the LUT+matrix colour kernel but never calls it
  for colour negatives; and brought the OEM's F235 SDK manuals into the
  sources. The client comparison also surfaced disputes about command
  names, per-resolution strides and the motor stop sequence, held back for
  verification.

Cross-confirmation with independent projects (pakon-macos, the FX35 Windows
driver, and Stefan Dierauf's libpakon) is cited throughout where it exists.
Facts verified on
real hardware in August 2026 are marked `[CONFIRMED on hardware, August 2026]`.
Every fact carries a confidence marker and a source; see
[CONVENTIONS.md](docs/CONVENTIONS.md).

## Contents

| Document | Covers |
|---|---|
| [docs/per-unit-data-and-safety.md](docs/per-unit-data-and-safety.md) | What data is unique to each unit and where it lives, how to back it up safely, what can be damaged and how, and what is recoverable. Read first. |
| [docs/usb-identity-and-firmware.md](docs/usb-identity-and-firmware.md) | Cold/warm enumeration, the personality mechanism, FX2 firmware selection and loading |
| [docs/ppb-protocol.md](docs/ppb-protocol.md) | The command frame format, packet types, status codes, bus addresses, bridge open |
| [docs/command-reference.md](docs/command-reference.md) | Per-controller command sets (light/CCD, motor, host) with payloads and sequences |
| [docs/image-stream.md](docs/image-stream.md) | The bulk image stream: chunking, row strides, channel interleave, IR lane, marker bit |
| [docs/calibration.md](docs/calibration.md) | Register banks, dark/bright calibration, the fixed-pattern-correction formula |
| [docs/color-pipeline.md](docs/color-pipeline.md) | The C-41 inversion (ColNeg), film-base normalisation, rendering stages |
| [docs/film-transport.md](docs/film-transport.md) | Motor control, advance/eject sequences, the run/stop register, film sensing |
| [docs/dx-barcode.md](docs/dx-barcode.md) | The DX film barcode subsystem across the model family |
| [docs/scanner-family.md](docs/scanner-family.md) | What differs between the F-135, F-135+, F-235, and F-335; the OEM host software stack and which engine DLL serves which model (TLB = 135 line, TLA = F-235, TLC = F-335) |

## Open-source Pakon clients

A list of emerging third-party Pakon clients:

| Project | Approach | Models driven | License |
|---|---|---|---|
| [PakonClient](https://github.com/eatfrog/PakonClient) by Henri Toivonen | C# client that drives the OEM COM server (`tlx.dll`) on Windows, plus research notes. | F-135+ (via the OEM stack) | not stated at time of writing |
| [pakon-macos](https://github.com/jorshhh/pakon-macos) by Jorge Rangel | Userspace driver plus web app; capture replay of the OEM sequences with its own decoder and C-41 inversion. Linux and macOS. Protocol and imaging notes. | F-135; F-135+ via a pending contribution | AGPL-3.0 |
| [pakon-mac](https://github.com/gazzdingo/pakon-mac) by Guy Langford-Lee | Userspace driver, Electron app, and a reimplementation of the OEM colour engine checked against the vendor DLLs under emulation; regenerable per-unit calibration. macOS, Windows, Linux. Extensive research notes. | F-135+ | MIT with a non-commercial restriction |
| [pakon-tlx-macos](https://github.com/pablonavarrob/pakon-tlx-macos) by Pablo Navarro | Runs the unmodified OEM Windows software under Wine on macOS, replacing only the kernel driver with a shim to libusb. Kodak's own colour science and Digital ICE, no reimplementation. Notes on the OEM's driver contract and behaviour. | F-135 | MIT |
| [psix](https://github.com/veroc/psix) by Oliver Roch | Userspace driver and local web app on Linux: firmware load, transport, C-41 inversion, IR dust removal. | F-135+ | not stated at time of writing |

## Sources and credit

Facts verified in or contributed by other projects are credited inline. What
each source contributed:

- [FX35](https://github.com/ktkaufman03/FX35) by ktkaufman03: modernised
  open-source Windows kernel driver; the driver-level captures used here were
  made with a logging-instrumented build of it.
- libpakon by Stefan Dierauf: an independent C++ driver and "Pakon Studio"
  app (private); the marker-bit row-origin technique was shown to this author
  by its author.
- [pakon-macos](https://github.com/jorshhh/pakon-macos) by jorshhh: the F-135 and F-135+ were verified working there, and the
  hardware-only facts here were derived by driving a real scanner with it.
- [pakon-mac](https://github.com/gazzdingo/pakon-mac) by gazzdingo: stated the TLA/TLB/TLC = F-235/F-135/F-335 file mapping before it was
  established here; that the one 135-line build also serves the F-135+ is
  this reference's addition.
- [pakon-tlx-macos](https://github.com/pablonavarrob/pakon-tlx-macos) by pablonavarrob: because it forwards every call the unmodified OEM stack
  makes, it is a ground-truth witness to the OEM's host-side behaviour on an
  original F-135.
  Its `tools/eedump.py` is the read-only EEPROM reader, replaying the OEM
  engine's own transfer sequence, used for the per-unit EEPROM facts in
  [calibration.md](docs/calibration.md#the-per-unit-eeprom).
- [PakonClient](https://github.com/eatfrog/PakonClient) by eatfrog: the source of the OEM F235 SDK manuals (June 2004) cited here.

## License

[CC BY 4.0](LICENSE): use it for anything, including commercial work, with
attribution. The point of this repository is that nobody should have to pay
for, or re-derive, the facts.
