# The DX barcode subsystem

35mm film carries a **DX barcode** along its edge, encoding the film product
and speed. The scanners read it with a dedicated infrared sensor and use it to
pick film-specific processing. This subsystem is independent of the visible
image path.

The scanners also take the **frame numbers** from that barcode, and those
numbers become the frame labels and the default filenames when a roll is
saved. So when no code is read, every frame is labelled `DX_Error` and the
default filenames are all identical, which collapses a saved roll into one
file. Two ordinary situations produce that: **the DX sensor fails**, which is
common on these long out of support machines, and **the film carries no edge
code**, which is true of respooled motion-picture stock and much else.

That is why this page documents not only how the host reads the code but what
the engine requires to accept one. Those requirements were established by
**replaying the sensor replies**: a bridge between the OEM software and the
scanner substitutes a decoded code for what the faulty sensor returned, and
the engine's response says which conditions it is testing. Facts established
that way are marked as such below. The same technique is what lets an
implementation supply a film code and frame numbers by hand.

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

DX reading uses a **separate IR path**, distinct from the imaging optics: an
emitter on one side of the film and photodetectors on the other, with analog
gain on the detector side set by adjustable pots ("DxPots"). There are four
detectors, read as four bytes from PICL register `0x93`. "DX sensor" below
means that assembly as a whole; where a fault is attributed to one half or the
other, it says which. [DOCUMENTED] from the service description.

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
command and reply between the OEM engine and libusb (2026-08-18 to 20), and
read from `TLB.dll` 3.1.0.28 decompilation for interoperability. Full
command/reply captures of all six resolution and Digital ICE configurations
are published at
[pakon-captures](https://github.com/alibosworth/pakon-captures). Some facts
below were established by replaying substituted replies through that bridge,
as described above; that is noted where it applies, and no scanner firmware or
OEM binary is modified._

**The trigger names the resolution and IR state.** [CONFIRMED live, six
configurations] The scan-line trigger `WRITE PICL 0x91` (SetScanLineParams)
carries a 16-bit little-endian value identifying which of the six
combinations of scan resolution and Digital ICE the host is about to run. It
is issued twice per scan, once for the pre-scan calibration and once for the
transport pass, with the same value both times. The values seen, each
matched against an independent OEM Windows capture of the same unit:

| resolution | Digital ICE | `0x91` value |
|---|---|---|
| Base 4 | off | `0x0107` |
| Base 4 | on | `0x00c5` |
| Base 8 | off | `0x0075` |
| Base 8 | on | `0x004d` |
| Base 16 | off | `0x003c` |
| Base 16 | on | `0x0031` |

The motor rate written just before transport (`WRITE PICM MOTOR RATE`) is a
second, independent identifier of the configuration: it is distinct for each
of the six, and decreases as the resolution and IR load rise.

**The read is event-driven, not polled.** [CONFIRMED live, ~10 scans] The
engine never issues ReadSensorData (PICL `0x90`) on a timer during a scan.
The sequence is: the HOST poll reply carries bit `0x80` ("a controller wants
service"), the engine reads PICL register `0x02` and finds bit `0x02` set,
writes the service acknowledge (`0x06`, payload `00 02`), and only then
issues the 30-byte `0x90` read. On the reference unit, whose sensor decodes
nothing, this happens about once per scan (and once during the pre-scan
calibration), so a scan sees one or two sensor reads. Asserting the same two
bits in the bridge makes the engine acknowledge and read on demand, at any
cadence tried (0.3–3 s), which is how the replayed replies below were
delivered.

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
the position counter is live in every reply, and restarts at the `0x91`
trigger.

**Type-3 entry: the decoded barcode.** [CONFIRMED by decompilation and by
live acceptance of a substituted entry] Bytes after the type byte, for the
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
on-film code (18 data bits + parity). An entry carrying product 79 /
generation 11, substituted this way, makes the engine report exactly that in the
client and clear its "Product And Specifier" warning. [CONFIRMED live,
2026-08-18]

**A type-3 entry has no position of its own.** [CONFIRMED by decompilation]
The engine takes the code's position from a slot filled by a preceding
type-5 entry (byte 1 = flag, bytes 2–3 = 16-bit position; flag bit 0 selects
the "A" slot, and the type-3's byte-1 bit must name the same slot). A code
without a matching type-5 is dropped before the frame table; the
product/generation vote happens earlier, which is why product can be
accepted while frame numbering is not.

**The engine moves the half-frame code.** [CONFIRMED by decompilation and
live, 2026-08-20] The type-3 handler does not use an "A"-slot code's position
as given: it adds a fixed shift `this+0x58 = width × 0x695F / 23700`, which is
1138, 1707 and 2276 counts at Base 4, 8 and 16. A plain-slot code is not
shifted. That is the same fraction of the frame pitch at every resolution,
about 0.70 of a frame, so it represents a fixed distance along the film of
roughly 27 mm rather than anything derived from the 19 mm code spacing; the
gap between the DX sensor and the imaging line is the obvious candidate but is
not established here. For an implementation what follows is that a substituted
"A" code must be placed that many counts early to be recorded where intended,
which was confirmed by pre-compensating: the code then appeared in the
engine's own table at the position aimed at.

**A film start must be recorded, but the engine mostly ignores its value.**
[CONFIRMED by decompilation and live] The numbering pass declares DX unusable
before looking at any code unless a film-start position was recorded during
the scan (a type-7 entry with byte-1 bit 1 set; a type-5 with bit 1 set
satisfies the existence test but stores a sentinel that ruins placement). So
the type-7 is required. Its *position*, however, is largely discarded:
`sub_10015520` derives a film-found offset (`this+0x3c`) from it, then bounds
that against a nominal offset computed from the scan geometry alone, and
substitutes the nominal when the derived value falls outside a window of
`width × 2000 / 23700` either side. On the reference unit the substitution
fired on nearly every scan (`FilmFoundOffset` equalled `uiOffsetCcdTest` in
the debug log), so the phase was set by the nominal, not by the injected film
start. The nominal, per resolution, is `width / divisor × 16042 / 23700`,
which is about 169 without IR at every resolution (`width / divisor` is 250)
and about twice that with IR.

**Event entries (types 7 and 8).** [CONFIRMED live] The controller emits
type-7 and type-8 entries of its own, each with a 16-bit position: sensor
edge events (an 8/7 pair brackets a film edge; the engine keeps a list of the
midpoints). These carry flag byte `0x01`, not the `0x02` that marks a film
start, so on this engine's two-slot path they establish neither the film
start nor, once a film start exists, anything the numbering uses. Types 5–8
are the only entries this unit's controller has ever produced on its own; it
has never produced a type 3.

**Frame numbering.** [CONFIRMED from decompilation and live with replayed
replies, 2026-08-19/20] After the scan the engine walks the table of
positioned codes and accepts numbering only if it finds at least three
consecutive codes it did not have to interpolate or correct: consecutive
positions one half pitch apart within ±1/8 of the expected pitch, `frame_raw`
stepping by 1 per code (35 mm), the half-frame parity matching, and the run
starting on an odd `frame_raw` (an "A" code). A code closer to its neighbour
than seven-eighths of the expected half pitch is dropped; a gap wider than
the tolerance is filled by interpolation. If no accepted run exists the scan
carries the "DX Read" warning and every frame is labelled `DX_Error`, and
because the default filenames are then identical, saving the roll writes a
single file.

**The expected pitch is a function of the scan width.** [CONFIRMED by
decompilation and live, all six configurations] `sub_10015520` computes the
expected half pitch as `width × 19200 / 23700`, so a frame pitch of
`width × 38400 / 23700`: 1620, 2430 and 3240 counts at Base 4, 8 and 16.
Digital ICE doubles it (the transport runs a second IR pass, so the position
counter advances twice as far per frame): 3240, 4860 and 6480. Each value was
confirmed by a run of replayed replies that numbered correctly; the OEM resolution
table's slightly different figures (1603 / 2405 / 3206) also pass, both being
inside the ±1/8 window.

**The divisor, and Digital ICE.** [CONFIRMED live, all six configurations]
The engine reduces raw positions to normalised line units by dividing by
`this+0x14`, its per-configuration divisor: 4, 6, 8 at Base 4, 8, 16 without
IR, doubling to 8, 12, 16 with it. The divisor is legible in any scan with
the debug log on: it is the film-start position divided by the `FilmFoundDx`
line. Why IR doubles it is localised in the decompilation but not fully read
out (the value is set in the scan-parameter marshalling, where the current
decompiler output drops the argument); the OEM resolution table itself sets
the divisor to 4/6/8 with no IR term, so the doubling is applied downstream.
[the doubling is UNRESOLVED as to mechanism; the values are CONFIRMED]

**Which picture a code labels.** [from decompilation; the unit conversion
CONFIRMED, the exact comparison INFERRED] Placement is not done in raw
counts. A code's position is converted to a line coordinate
`(position − FilmFoundOffset) / divisor` and compared against the picture
coordinates the image framing found, which are in the same units (pictures
come out about 405 line-units apart and 375 wide, at every resolution, which
is what makes the framing normalised). Which code a given picture ends up with
therefore depends on where the pictures fall, and on real film that is not
fixed: the edge code is imprinted at the factory at a constant pitch while the
frames are exposed later by the camera at whatever phase the loading produced,
so a picture may sit over a whole-frame label, over its "A" half-frame label,
or across both. That is the reason both labels exist.

So a strip whose pictures come out labelled `1A, 2A, 3A, 4A` is a correct
reading, not a fault: it says the frames were exposed nearer the half-frame
labels than the whole-frame ones. Any phase is legitimate.

When the code positions are supplied rather than read from film, the phase is
a setting rather than something the film imposes, so a whole-frame series can simply be chosen as the tidier result.
Landing the configured number on the first picture that way needed the
sequence started 1 half-frame early at Base 4 and Base 8 and 2 at Base 16.
Those are measurements of one arrangement on one unit, not a property of the
scanner.
[CONFIRMED live for the configurations tested]

**Everything the engine requires, in one place.** [CONFIRMED live, all six
configurations, 2026-08-20] Numbering was reproduced from replayed replies by:
recording a film start (type 7, bit 1); placing one type-5 position event and
one type-3 code per half frame on a grid of `width × 19200 / 23700` counts
(doubled for IR); sending "A"-slot codes early by `width × 0x695F / 23700` to
survive the engine's shift; and starting the sequence a frame ahead so the
first picture takes the configured number. The client then showed the product
and specifier, numbered the frames, and saved one file each. The relationship
between the position counter and physical film travel is the one piece still
not reconciled: the counter was measured at about 2 counts per image row
during streaming, yet a frame's accepted pitch (about 1620 counts at Base 4)
is close to its length in image rows (1603), which implies about 1 count per
row. The two measurements have not been squared. [UNRESOLVED]

**Registry switches: the engine's own DX log.**

> **A note about the Pakon implementation.** The settings below belong to the
> OEM Windows software, not to the scanner. They are recorded because the file
> one of them produces is the engine's own account of how it read a scan, which
> is how most of the behaviour above was checked.

[CONFIRMED live] Under `…\Pakon\TLB\Scan\Test`,
`DxCreateDebugFilesCommunication = 1` writes a PPB traffic log for the motor
controller (`Logs\PakonPpbDebugDx<serial>.txt`), nothing DX-specific.
`DxCreateDebugFiles = 1` (read at engine start; on a 64-bit host it must land
in the 32-bit registry view) makes the engine write `Logs\DxCode.txt` after
each scan: the decoded code table (ID, frame, PAR, Line), `Good Dx Count`,
the film-found position and offsets, and the picture-framing results. Reading
it: `Line = position / divisor − FilmFoundOffset`, and the divisor is the
film-start position over `FilmFoundDx`. It is the engine's own ground truth
for DX debugging. [CONFIRMED live, 2026-08-19; an earlier failed attempt had
not restarted the engine] `Logs\PakonDxLog.txt` stayed empty even on a
successful pass; what would ever fill it is unknown.
`DxCalibrationFilmOffset` is consumed only by the sensor pot calibration, not
by numbering. [CONFIRMED by decompilation]

## Note on the reference hardware

On the specific F-135+ used for much of this research (serial 16402), DX
reading fails. In the captured sensor responses examined (three reads,
including one during an active DX scan), the DX-related fields were exactly
zero while other leading bytes were nonzero. [CONFIRMED] for those captured
responses. A fault somewhere in the DX path on that unit is the suspected
explanation, not a proven failure mode. [INFERRED]

Narrowed 2026-08-18: during the client's Film Track Test the engine polls
PICL register `0x93` (4 bytes, one per detector) every ~3 ms while running
the transport. On this unit all four values sit near 248 with no film and
each drops to roughly 110–135 while film covers it, in two pairs a few
seconds apart. So none of the four is electrically dead, and light of some
kind is reaching all of them. None shows the fast clear/dark alternation a
barcode would produce, and the controller has never returned a type-3 entry.

Two things that reading does **not** establish. Which light those four
detectors are responding to is unknown. The imaging illuminant has its own IR
channel, used for Digital ICE, shining across the film path towards the CCD;
a DX detector near that path would see it as background whether or not the DX
emitter is lit. Film passing between them attenuates that background exactly
as it would the DX emitter's own light, so the swing does not by itself show
that the DX emitter works. And whether these four are the DX barcode readers
at all is unconfirmed; they drop in two pairs seconds apart, which suggests
sensors spread along the film path rather than four readers at one point.

So the fault may lie anywhere in the path, and **a dead or weak emitter is a
leading candidate**: it would produce exactly this picture, with the
detectors still reporting film presence from ambient light while no barcode
modulation ever reaches them. All four failing the same way points to a shared
element rather than four independent faults, and the emitter is the shared
element. Optics, gain and the controller's decoder remain open too.
[CONFIRMED readings; the cause INFERRED and not narrowed]

Two measurements would separate these: reading register `0x93` with the lamp
off (light still present means a working DX emitter; darkness means the swing
above came from the lamp), and identifying which of the four are the DX
readers, by blocking them one at a time or running a very high contrast edge
code past them.

## Follow-up work

Product/generation, frame numbering, and the whole host-side read sequence are
now established and reproduced from replayed replies across all six
resolution and Digital ICE configurations. What remains open:

- a capture from a unit whose DX sensor **works**. Everything above says what
  the engine requires; a real decoding controller would show what it actually
  emits (how a genuine type-3 arrives, event grouping, how "no film" and
  "decode failed" differ). The reference unit's sensor does not decode, so
  this cannot be answered here.
- **why Digital ICE doubles the divisor.** The values are confirmed (4/6/8 to
  8/12/16); the mechanism is localised to the scan-parameter marshalling, but
  the current decompiler output drops the argument at that call, so it needs a
  cleaner disassembly of that one code path.
- **the position counter's relation to film travel.** It measured about 2
  counts per image row during streaming, yet a frame's accepted pitch is close
  to its length in image rows, implying about 1. The two have not been
  reconciled.
- **what a cut through a code does.** The barcode repeats along the film edge,
  so cutting a roll into strips leaves a partial code at each end of a strip.
  Whether the controller reports such a fragment, suppresses it, or returns it
  as a code that fails parity is not known, nor what the numbering pass then
  does with a strip's first and last pictures.

The reference unit's fault is a fault of the individual scanner, not a
protocol fact, but it shaped which DX facts could be verified against live
hardware versus derived from the OEM software. Full command/reply captures of
all six configurations are at
[pakon-captures](https://github.com/alibosworth/pakon-captures).

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
