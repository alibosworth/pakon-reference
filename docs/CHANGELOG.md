# Changelog

What changed in the reference, when, and whether each change added a fact,
tightened one, or reversed one. Reversals are listed as such: a fact that
carried a confidence marker and turned out wrong is a normal event here,
and hiding it would defeat the markers. Dates are when the change was
published; each entry links the page it touched. Newest first.

## 2026-08-19

- **Addition, `dx-barcode.md`:** frame numbering is now reproduced live with
  synthetic replies, upgrading the numbering facts from decompilation-only
  to confirmed. The key requirement, found by decompilation and confirmed
  on hardware: a type-7 film-start event (byte 1 bit 1) must precede the
  codes or the numbering pass declares DX unusable without looking at them.
  The accepted geometry settles the pitch units (810 position counts per
  code at Base 4, table lines = counts / 4). Also corrected: the engine's
  DX debug table is `DxCode.txt`, switched on by `DxCreateDebugFiles`
  (the page previously said that switch produced no file on a 135-line
  unit; the earlier attempt had not restarted the engine), and
  `PakonDxLog.txt` stays empty even on a successful pass.


- **Addition, `dx-barcode.md`:** a "How the host reads it" section: the
  sensor read is event-driven (service flag → acknowledge → `0x90`), the
  34-byte reply layout, the position counter (resets at `0x91`, 2 counts per
  scan line at Base 4), the type-3 byte layout and its 19-bit parity, the
  type-5 position event a code depends on, the type 7/8 edge events, the
  engine's frame-numbering acceptance rule, and what the DX registry
  switches actually do. Two follow-up items closed by it. Sourced from live
  observation on an F-135+ through a logging bridge (with synthetic replies
  for the layout claims) and from `TLB.dll` 3.1.0.28. Frame numbering from
  synthetic replies is explicitly not yet achieved.
- **Tightening, `dx-barcode.md`:** the reference unit's DX fault narrowed:
  all four photodetectors respond (register `0x93`), no barcode modulation,
  no type-3 ever returned.
- **Addition, `command-reference.md`:** `0x93` ReadDxSensors; notes on
  `0x90`/`0x91`/`0x92`; the sensor-response open question rewritten.

- **Addition, `index.md`:** a standing request for corrections and
  additions, with what a useful report contains; the same line in the site
  footer. "This author" replaced with the name in the credits and in
  `image-stream.md`, since the phrase assumes a single author.

## 2026-08-18

- **Addition, `calibration.md`:** the EEPROM select request's `wValue` is
  `((0x50 | n) << 1) | direction`: it reaches any I2C EEPROM at
  `0x50`–`0x57`, and bit 0 is the direction, so a write is the even-valued
  select followed by `0xA2` instead of `0xA9`. From the engine's own read
  routine, read out by pakon-mac for its transport allow-list; the read
  direction confirmed on hardware, the write direction from disassembly
  only. Also `per-unit-data-and-safety.md`: a read tool "issues" the OEM's
  sequence rather than "replays" it (nothing is played back).

## 2026-08-17

- **Correction, `calibration.md`:** the first word of each per-resolution
  triple in the EEPROM had been called an "optical offset" as if that were
  known. The OEM's name for it is `Offset` and it says nothing about its
  meaning; the page now gives the name, the values seen, and the reason for
  reading it as the per-base start of the imaged CCD region, marked
  inferred. Also states pakon-mac's read-once EEPROM report as one
  project's observation, linked, rather than as a property of the parts.
- **Addition, `per-unit-data-and-safety.md`:** a new page for anyone running
  unofficial code against these scanners: what data is per-unit and where
  it lives (EEPROM vs registry), how to back it up with the OEM's own read
  sequence and nothing else, and what host software can damage (the FX2
  wedge, PIC bootloader row erase, blind writes to the I2C EEPROMs, LED
  current ceilings), each pinned to the file and commit where it was
  observed. Reads with pakon-tlx-macos's `eedump.py`; layout and the
  incident reports from pakon-mac.
- **Addition and correction, `calibration.md`:** the read the OEM does at
  start-up over vendor requests `0xA4`/`0xA9`, previously an undecoded
  "parameter table", is the scanner's per-unit EEPROM (serial, per-resolution
  `Offset` word and motor speeds, motor-adjust words, the two 3x10 colour
  matrices; not the light calibration). Documents the read, the two-copy
  layout with CRC-32, the field map, and how the OEM falls back to the
  backup copy on a bad primary (first good copy wins, no repair, warning
  only if both fail), confirmed by the engine's own routine and by the
  registry it populates. Closes the matching open questions on the PPB and
  USB/firmware pages. Layout from pakon-mac; reads with pakon-tlx-macos's
  `eedump.py`, extended to verify both copies.
- **Correction, `scanner-family.md`:** the F-135 and F-135+ are lit by an
  LED array with per-channel current and duty control (R, G, B, IR), not a
  lamp. The at-a-glance table had taken the OEM's own word "lamp" literally.
  The F-235 stays a lamp (as documented, not observed); the F-335 stays LED.
- **Addition and correction, `scanner-family.md`:** a section on the OEM
  host software stack. The shipped COM server is a `tlx.dll` facade over
  three per-model engine builds, and two things about it are not what the
  names suggest: the letters A/B/C do not follow the model numbers 135/235/
  335 (`TLB.dll` serves the F-135 and F-135+, `TLA.dll` the F-235,
  `TLC.dll` the F-335), and the F-135 and F-135+ share one engine despite
  different controller boards. **Reverses** the earlier description of the
  host software as a single shared "TLX/TLA" stack, and re-attributes facts
  this reference had sourced to "TLA.dll": those were read from the F-235
  engine, or from captures of 135-line hardware, and are labelled
  accordingly. Established from the binaries' own string tables and
  `tlx.dll`'s dispatch, and confirmed on both 135-line generations.
- **Correction, `CONVENTIONS.md`:** cite OEM binaries by PE file version
  (every engine, the facade and the demo client are 3.1.0.28), and hash the
  copy actually analysed, since installed engine DLLs are sometimes patched
  by tooling; name the engine DLL a fact came from, since they are
  per-model builds.
- **Scope note, `color-pipeline.md`:** the LUT+matrix path documented on
  that page is the F-235 engine's; the 135-line engine resolves that kernel
  but never calls it for colour negatives and applies a per-unit 3x10
  transform instead. The 135-line colour path is now an explicit open
  question rather than an implied match.
- **Correction, `image-stream.md`:** the 20480-byte bulk transfer size is
  the OEM host's ring-packet choice (the driver only enforces a 0x5000
  ceiling on a host-supplied value), not a property of the endpoint, whose
  own maximum packet size is 512 bytes.
- **Addition, front page:** a directory of the third-party Pakon clients,
  and a sources-and-credit list separated from it. Automatic site
  deployment on every merge (the site had been a manual snapshot and had
  gone stale).

## 2026-08-14

- **Addition, every page:** an "Open questions" section per page, and a
  statement that the reference is not comprehensive.

## 2026-08-13

- **Additions from hardware, several pages:** the F-135+ driven end to end
  by open-source code (the pakon-macos project). Added the reply direction
  of the protocol (reply-frame structure, the `0x88` READ marker byte,
  first-open-only handshake replies), per-resolution image strides, that
  IR-off scans carry no IR lane, and the motor run/stop register behaviour,
  all marked `[CONFIRMED on hardware, August 2026]`. **Under review** as of
  17 August: independent implementations dispute the stride table and the
  run/stop register (the value written may be the CCD acquire word rather
  than a speed); those markers stand until the reference's own captures are
  re-checked, and will be corrected here if they fall.
- **Addition, `CONVENTIONS.md`:** the scope rule separating what the
  hardware and OEM software do from how the OEM's code is structured.
- **Addition:** credit for contributing projects (FX35, libpakon,
  pakon-macos); the MkDocs site.

## 2026-07-10

- **Addition, `color-pipeline.md`:** the exact ColNeg inversion LUT,
  `D(x) = 3500 * log10(16383 / x)`, and the LUT-then-matrix ordering, from
  numeric analysis. (Now scoped to the F-235 engine; see 2026-08-17.)

## 2026-06-03

- **Addition, `calibration.md`:** the fixed-pattern-correction gain
  formula, partial (prefix terms and clamp still open).

## 2026-05-02

- **Additions:** USB identity and firmware loading; the scanner-family
  page; calibration register banks.

## 2026-05-01

- **Addition, `dx-barcode.md`:** the DX barcode subsystem: encoding, the
  product-code table, hardware, per-model differences.

## 2026-03-13

- **Addition, `image-stream.md`:** the bulk image stream: chunking,
  interleave, IR lane, the exported raw format.

## 2026-03-08

- **First publication:** scaffold, the PPB protocol, and the command
  reference from the March findings.
