# Changelog

What changed in the reference, when, and whether each change added a fact,
tightened one, or reversed one. Reversals are listed as such: a fact that
carried a confidence marker and turned out wrong is a normal event here,
and hiding it would defeat the markers. Dates are when the change was
published; each entry links the page it touched. Newest first.

## 2026-08-20 (later)

- **Correction, `dx-barcode.md`:** "DX sensor" was used throughout as though
  the subsystem were one component, and the reference unit's fault was
  attributed to "a failed DX IR sensor", which reads as the detector. The path
  has two halves, an emitter one side of the film and four photodetectors the
  other, and the page now says so where the subsystem is introduced. The
  attribution was also weaker than the page's own evidence: the detectors read
  about 248 with no film and drop to 110-135 when film covers them, which they
  could only do if the emitter were lit. **That inference was wrong and is
  withdrawn**: the imaging illuminant's own IR channel shines across the film
  path for Digital ICE, and a DX detector near it would see that as background
  whether or not the DX emitter works, with film attenuating it the same way.
  Nor is it established that those four detectors are the DX barcode readers.
  A dead or weak emitter is in fact a leading candidate, since all four failing
  identically points at the one element they share. The page now says the cause
  is not narrowed, and names the two measurements that would separate the
  cases: reading the detectors with the lamp off, and identifying which of the
  four are the DX readers.

- **Addition, `dx-barcode.md`:** the page now says at the top why a reader
  would care about supplying a DX code rather than reading one. The frame
  numbers become the frame labels and the default filenames, so a roll with no
  readable code saves as a single file, and two ordinary situations cause
  that: a failed DX sensor, common on machines this old, and film that carries
  no edge code at all. The page's method is introduced there too: the
  engine's requirements were established by replaying substituted sensor
  replies and watching which conditions it tested. Previously the word
  "synthetic" appeared thirteen times without ever being introduced. The
  vocabulary is now consistent, and a section heading that read "What a
  synthetic scan needs" no longer uses "scan" to mean a run of replayed
  replies. The registry-switch paragraph now carries the
  conventions' "A note about the Pakon implementation" callout, so a reader is
  not left to work out that those settings belong to the OEM software rather
  than to the scanner.

- **Correction, `dx-barcode.md`:** two claims about the "A" half-frame code
  are withdrawn, both of which assumed a fixed relationship between the edge
  code and the pictures. There is none: the code is imprinted at the factory
  at a constant pitch and the frames are exposed later by the camera at
  whatever phase the loading produced, so a picture may fall over a
  whole-frame label, over its "A" label, or across both (see
  [35mm-dx-edge-code](https://github.com/alibosworth/35mm-dx-edge-code)).
  So an "A" code arriving where a plain one was expected is not in itself an
  error, and the engine's fixed shift for "A"-slot codes is not a correction
  for code spacing: it is about 0.70 of a frame at every resolution, roughly
  27 mm of film, where the code pitch is 19 mm. The page now states the shift
  and the measured synthetic-sequence phase as what they are, measurements of
  one arrangement, without the causal reading. It also now says plainly that a
  strip labelled `1A, 2A, 3A, 4A` is a correct reading rather than a fault:
  any phase is legitimate, and a synthetic sequence aims at the whole-frame
  series only because it is tidier.

- **Removal, `dx-barcode.md`:** "Because it reads the film edge, DX is legible
  even on cut strips" is deleted. It did not say what it meant, and the
  comparison it implied does not hold: film reaches one of these scanners
  developed and cut, with the cassette long since discarded, so the latent
  edge barcode is not merely still legible, it is the only DX the scanner
  could ever read. Added in its place, under Follow-up work, the question the
  cut-strip case actually raises: a cut through a code leaves a partial one at
  each end of a strip, and what the controller and the numbering pass do with
  that is not known.

## 2026-08-20

- **Addition and one reversal, `dx-barcode.md`:** the host-side DX read is
  rewritten with everything established through 2026-08-20 and reproduced from
  synthetic replies across all six resolution and Digital ICE configurations.
  New: the `0x91` (SetScanLineParams) value that names each configuration, with
  a table matched to OEM Windows captures; the engine's expected frame pitch
  `width x 38400 / 23700`; the per-configuration divisor (4/6/8, doubling to
  8/12/16 with Digital ICE); the half-frame slot shift `width x 0x695F / 23700`
  the engine adds to "A"-slot codes; the film-found offset clamping (the engine
  records the film start but usually substitutes a nominal offset for it); and
  the code-to-picture unit conversion. **Reversal:** the earlier draft's
  reading that synthetic codes were "placed about twice as densely as real
  film" is withdrawn. It rested on the unreconciled 2-counts-per-image-row
  figure; the accepted pitch is close to a frame's length in image rows, so
  the density is normal. The counter-to-film-travel relation stays marked
  unresolved. Command/reply captures of all six configurations are published
  at pakon-captures and linked.

## 2026-08-19

- **Addition, `resources/dx-product-code-table.md`:** a new Resources section,
  starting with the OEM's DX product-code table: all 361 assigned
  (product, generation) pairs with their ISO speeds and the file's own
  product-line labels, plus the original `common-ProdCodeTable.dpi` text.
  Carrying that file is a stated exception to the "facts, not artifacts"
  rule, argued in `CONVENTIONS.md`: the assignments are PIMA's rather than
  the OEM's, and a lookup table summarised is a lookup table wasted.

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
