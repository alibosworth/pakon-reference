# The DX barcode subsystem

35mm film carries a **DX barcode** along its edge, encoding the film product
and speed. The scanners read it with a dedicated infrared sensor and use it to
pick film-specific processing. This subsystem is independent of the visible
image path.

_From reverse engineering of the OEM DX handling and the product-code table,
March 2026; product-code decoding cross-checked May 2026; host-read protocol
and frame numbering established live August 2026._

## What the barcode encodes

The edge barcode follows the PIMA/ISO DX standard and encodes two numbers:

- **DX Part 1**: a 7-bit "product code" (called ProdCode internally):
  manufacturer + product line.
- **DX Part 2**: a "generation code": the specific emulsion/speed within that
  product line.

Together they identify the film so the correct ISO speed and processing can be
selected. [DOCUMENTED] from the OEM standard and the product-code table.

## The product-code table

The OEM ships a lookup table mapping (product code, generation code) → film
identity and ISO, at
`F-X35 COM SERVER\anselinstalldir\dataPathItems\common\common-ProdCodeTable.dpi`.
It was last revised 6 June 2006, so **any film launched after 2006 has no
complete entry** and falls back to a default ISO. [DOCUMENTED] from the table
file header. Every assigned entry, and the original file, are in
[Resources: the OEM's DX product-code table](resources/dx-product-code-table.md).

A concrete consequence: modern Kodak Portra encodes as (product code 95,
generation code 14). Product code 95 (Kodak's premium 35mm colour-negative
class) is in the table, but generation code 14 was not allocated when the table
was frozen, so a modern Portra strip hits a known-product / unknown-generation
combination and the lookup yields the default ISO. [CONFIRMED 2026-05-01] the
(95, 14) encoding was verified two independent ways: a computational decode of
a scan, and an independent edge-scanner database. After the DX standards body
stopped publishing assignments around 2006, manufacturers self-allocated within
their existing product-code ranges, which is exactly what (95, 14) shows.

## Hardware and firmware

DX reading uses a **separate IR sensor**: IR is transmitted through the film
and read on the other side, with analog gain set by adjustable pots ("DxPots").
Because it reads the film edge, DX is legible even on cut strips. [DOCUMENTED]
from the service description.

The barcode is decoded **inside the light-controller firmware** (PICL), not on
the host: the firmware interprets the analog signal and returns a pre-decoded
result. The host sends commands to start/stop DX scanning, adjust the DxPots,
and read the result; it never sees the raw analog waveform. [DOCUMENTED] from
the OEM driver's DX command usage.

## Per-model differences

DX handling is integrated into the light controller on the F-135 and F-135+,
but the F-235 and F-335 carry a **dedicated DX board** with its own
microcontroller (firmware prefixes `DX` on the F-235, `DY` on the F-335). The
F-135/F-135+ have no separate DX firmware. [DOCUMENTED] from the firmware
inventory; see [scanner-family.md](scanner-family.md).

## How the host reads it

_Observed live on an F-135+ through a user-space bridge that logs every
command and reply between the OEM engine and libusb, 2026-08-18/19, and read
from `TLB.dll` 3.1.0.28 (SHA-256 `5866ec56…`) decompilation for
interoperability. The synthetic-reply experiments below rewrite the reply in
that bridge; no scanner firmware or OEM binary is modified._

**The read is event-driven, not polled.** [CONFIRMED live, ~10 scans] The
engine never issues ReadSensorData (PICL `0x90`) on a timer during a scan.
The sequence is: the HOST poll reply carries bit `0x80` ("a controller wants
service"), the engine reads PICL register `0x02` and finds bit `0x02` set,
writes the service acknowledge (`0x06`, payload `00 02`), and only then
issues the 30-byte `0x90` read. On the reference unit, whose sensor decodes
nothing, this happens about once per scan (and once during the pre-scan
calibration), so a scan sees one or two sensor reads. Asserting the same two
bits in the bridge makes the engine acknowledge and read on demand, at any
cadence tried (0.3–3 s), which is how the experiments below were driven.

**Reply layout.** [CONFIRMED live; matches the engine's parser] The reply is
34 bytes: the usual 4-byte PPB header (`01 20 <PICL address> 08`), then a
30-byte payload:

| payload bytes | content |
|---|---|
| 0–1 | position counter, big-endian 16-bit |
| 2 | entry count |
| 3… | entries, 5 bytes each: type in the low nibble of byte 0, then 4 bytes |

The engine sign-extends the 16-bit position with its own wrap counter. On
the reference unit the idle reply is count 1, type 0 (a "no entry" entry);
the position counter is live in every reply. It restarts at the scan-line
trigger (`0x91`). At Base 4 it advances about 1105 counts/s against 553
image rows/s on the image endpoint, a ratio of 2 counts per row. [CONFIRMED,
one measured scan; the ratio at other resolutions is INFERRED] Its rate
scales with the motor speed (about 506 counts/s at Base 16). What one count
corresponds to in film travel is **not** established: 2 counts per row would
put a 35 mm frame at roughly 3200 counts, while the spacing the engine
accepts as a frame pitch is about half that (see Frame numbering), and the
two have not been reconciled.

**Type-3 entry: the decoded barcode.** [CONFIRMED by decompilation and by
live acceptance of a synthetic entry] Bytes after the type byte, for the
two-slot mode the 135-line engine runs:

| byte | bits |
|---|---|
| 1 | bit 0 = "A" (half-frame) code, bit 1 = plain code (see type 5) |
| 2 | bits 7–2 = frame\_raw[5:0]; bit 1 must be 0; bit 0 = parity spare |
| 3 | bits 7–6 = product[1:0]; bit 5 = invalid (must be 0); bits 4–1 = generation; bit 0 = frame\_raw[6] |
| 4 | bits 4–0 = product[6:2] |

`frame_raw` is the frame number in half frames (`frame × 2 + half`; a 24-
exposure roll runs 2..49, exactly the range the F235 COM reference gives),
7-bit two's complement for the leader labels (−2 prints "00", −4 "X"). Parity
is even over the 19 bits of bytes 2, 3 and 4[4:0], the same rule as the
on-film code (18 data bits + parity). A synthetic entry with product 79 /
generation 11 injected this way makes the engine report exactly that in the
client and clear its "Product And Specifier" warning. [CONFIRMED live,
2026-08-18]

**A type-3 entry has no position of its own.** [CONFIRMED by decompilation]
The engine takes the code's position from a slot filled by a preceding
type-5 entry (byte 1 = flag, bytes 2–3 = 16-bit position; flag bit 0 selects
the "A" slot, and the type-3's byte-1 bit must name the same slot). A code
without a matching type-5 is dropped before the frame table; the
product/generation vote happens earlier, which is why product can be
accepted while frame numbering is not.

**A film start must be recorded, or numbering never runs.** [CONFIRMED by
decompilation and live, 2026-08-19] The numbering pass declares DX unusable
before looking at any code unless a film-start position was recorded during
the scan. Only a type-7 entry with byte-1 bit 1 set records a usable one
(its own position, bytes 2–3); a type-5 with bit 1 set also satisfies the
existence test but stores a sentinel that ruins the per-picture placement.
Once film start is set, further type-7 entries are treated as picture-edge
events. This was the single condition separating rejected synthetic
sequences from accepted ones: the same code stream failed without the
type-7 and numbered every frame with it.

**Event entries.** [CONFIRMED live] After film passed the sensors during a
Film Track Test, the reference unit's controller returned four 5-byte
entries of types 7 and 8 alternating, each with a 16-bit position: the
sensor's picture-edge events (an 8/7 pair brackets an edge; the engine keeps
a list of the midpoints). Types 5–8 are the only entries this unit's
controller has ever produced on its own; it has never produced a type 3.

**Frame numbering.** [CONFIRMED from decompilation and live with synthetic
replies, 2026-08-19] After the scan the engine walks the table of positioned
codes and accepts numbering only if it finds at least three consecutive
codes it did not have to interpolate or correct: consecutive positions one
half pitch apart within ±1/8 of the expected pitch, frame\_raw stepping by 1
per code (35 mm), the half-frame parity matching, and the run starting on an
odd frame\_raw (an "A" code). Missing codes are interpolated, surplus ones
dropped, and the picture then takes the code whose position, converted to
lines and shifted by the fixed distance between the DX sensor and the CCD
line, falls inside the picture span the image framing found. If no such run
exists the scan carries the "DX Read" warning and every frame is labelled
`DX_Error`, and because the default filenames are then identical, saving the
roll writes a single file.

The accepted geometry, at Base 4 on the reference unit: one code per
810 position counts, i.e. a frame pitch of 1620, with the film start at the
first code. That is what the engine accepts, and it is consistent inside the
engine's own arithmetic: its expected half pitch is about 801 counts, and its
code table prints positions divided by 4, in the same units as its picture
framing (pictures 375 wide at a pitch of ~405 = 1620/4). Whether 1620 counts
is also the *physical* length of a frame is a separate question and the
answer appears to be no: the OEM's resolution table gives 1603 lines per
frame at Base 4, and at the measured 2 counts per image row a frame would
span roughly 3200 counts. So these codes are placed about twice as densely as
real film would carry them, and the engine accepts them because its own
spacing test is satisfied. [CONFIRMED that it is accepted; the relationship
between counts, lines and film travel is UNRESOLVED] A stream of type-7 film start + type-5/type-3
pairs in that geometry was accepted end to end: no DX warning, the client
showing product 79 / specifier 11 and frames 1–4 (the COM frame-number
property is in half frames, so it displays 2/4/6/8), and one file per frame
on save. [CONFIRMED live, 2026-08-19]

**Registry switches.** [CONFIRMED live] Under `…\Pakon\TLB\Scan\Test`,
`DxCreateDebugFilesCommunication = 1` writes a PPB traffic log for the motor
controller (`Logs\PakonPpbDebugDx<serial>.txt`), nothing DX-specific.
`DxCreateDebugFiles = 1` (read at engine start; on a 64-bit host it must land
in the 32-bit registry view) makes the engine write `Logs\DxCode.txt` after
each scan: the decoded code table (ID, frame, line), `Good Dx Count`, the
film-found position, and the picture-framing results — the engine's own
ground truth for DX debugging. [CONFIRMED live, 2026-08-19; an earlier
failed attempt had not restarted the engine] `Logs\PakonDxLog.txt` stayed
empty even on a successful pass; what would ever fill it is unknown.
`DxCalibrationFilmOffset` is consumed only by the sensor pot calibration,
not by numbering. [CONFIRMED by decompilation]

## Note on the reference hardware

On the specific F-135+ used for much of this research (serial 16402), DX
reading fails. In the captured sensor responses examined (three reads,
including one during an active DX scan), the DX-related fields were exactly
zero while other leading bytes were nonzero. [CONFIRMED] for those captured
responses. A failed DX IR sensor on that unit is the suspected explanation,
not a proven failure mode. [INFERRED]

Narrowed 2026-08-18: during the client's Film Track Test the engine polls
PICL register `0x93` (4 bytes, one per DX sensor) every ~3 ms while running
the transport. On this unit all four values sit near 248 with no film and
each drops to roughly 110–135 while film covers it, in two pairs a few
seconds apart, so none of the four photodetectors is electrically dead.
None shows the fast clear/dark alternation a barcode would produce, and the
controller has never returned a type-3 entry. The fault is between the
detectors and the decoded code (optics, gain, or the controller's decoder),
not a sensor that reads zero. [CONFIRMED readings; INFERRED conclusion]

## Follow-up work

The open DX questions all trace to the reference unit's (serial 16402)
non-functioning DX reads, so the unlock is another scanner with a working DX
sensor:

- capture the DX command exchange (start scan, pot adjustment, sensor reads)
  on a unit that decodes film successfully: one scan of coded film through a
  logging bridge would show the real reply stream — the accepted synthetic
  geometry above says what the engine requires, not what a real controller
  emits (event grouping, extra event types, how "no film" and "decode
  failed" differ);
- (answered above, 2026-08-18) which bytes of the sensor response carry the
  decoded product/generation values;
- (answered above, 2026-08-18) whether the DX result arrives in the polled
  sensor state or as an event: as an event;
- (answered above, 2026-08-19) the engine's DX debug output: it is
  `DxCode.txt`, switched on by `DxCreateDebugFiles`.

The reference unit's fault is a fault of the individual scanner, not a
protocol fact, but it shaped which DX facts could be verified against live
hardware versus derived from the OEM software.

## Further reading

This page covers how the *scanner* reads and uses the DX code. For the DX
code system in general, see:

- [35mm-dx-edge-code](https://github.com/alibosworth/35mm-dx-edge-code): a
  concise structural reference for the DX latent edge barcode itself: physical
  layout, bit encoding, field positions, and the PIMA/ISO 1007 decoding method.
- [The Big Film Database](https://thebigfilmdatabase.merinorus.com): an
  independent, community DX-code database (webcam edge-scanner decodes), for
  looking up specific films including modern emulsions absent from the OEM
  2006 product-code table.
