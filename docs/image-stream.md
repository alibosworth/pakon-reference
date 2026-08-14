# The bulk image stream

Pixel data does not travel through the command protocol. The CCD data streams
out a dedicated bulk IN endpoint straight to the host, in fixed-size chunks,
while the command channel only controls timing and parameters.

_Reconstructed from USB scan captures and from the OEM's exported raw files,
March 2026._

## Transport

The image endpoint delivers the stream in **20480-byte bulk chunks**.
[DOCUMENTED] every image packet in the scan captures is this size. The host
reads continuously into a ring buffer while a second thread writes to disk; the
scan ends on a device-signalled end-of-roll, not a fixed byte count.

## Sample format

Samples are **16-bit little-endian**, and the bit depth differs by stage:

- **The raw USB stream is 16-bit pre-calibration data.** Captured film-scan
  streams contain values across the full 16-bit range (observed up to 65534).
  [CONFIRMED] from capture analysis.
- **The processed buffer and the OEM raw export are 14-bit** (0–16383, never
  exceeding 0x3FFF): calibration/processing maps the stream into the 14-bit
  domain that later stages expect. [CONFIRMED] the export is bit-identical to
  the processed buffer. The 16,384-entry inversion LUT corroborates the 14-bit
  domain of that later stage (see [color-pipeline.md](color-pipeline.md)); it
  establishes the processing input domain, not the USB transport depth.

## Orientation

The CCD is a **vertical** line of sensor elements, spanning the film's
image-height dimension (~24 mm). The film moves **horizontally** past it, so
each sensor readout captures one vertical column of the image, and successive
readouts, as the film advances, build up the horizontal dimension. So the
scanner geometry maps the CCD axis to image height and the travel axis to image
width.

The 90° rotation that appears in the pipeline is a **storage artifact**, not a
physical one: each readout (the "scan line") is streamed and stored as a *row*,
which lands the travel axis running vertically and the CCD axis horizontally,
rotated 90° from the physical scene. The OEM raw export undoes this, presenting
width = travel direction and height = CCD direction. [CONFIRMED] from the
raw-export geometry.

## Row layout

Each scan line is a fixed number of 16-bit samples. The visible image is
**per-pixel interleaved R, G, B** (consecutive samples cycle through the
three channels), not planar within the stream. [DOCUMENTED] from the OEM's line
processor, which de-interleaves period-3 (`R[0], G[0], B[0], R[1], ...`).

When the infrared (Digital ICE) channel is enabled, the row carries a **fourth
channel separately** from the RGB: the IR samples come from a different region
of the line, not interleaved. [DOCUMENTED] from the line processor's 4-channel
path, which reads the IR lane from a separate offset.

The per-row sample count and pixel width depend on the scan resolution; higher
resolutions produce wider rows. [DOCUMENTED] the exported raw is 3000×2000 at
the highest resolution and narrower at lower ones.

### Per-resolution strides (measured on hardware)

[CONFIRMED on hardware, August 2026] measured directly on an F-135+ across its
scan modes. The visible width is `stride / 3` when there is no IR lane and
`stride / 4` when the IR lane is present:

| Scan mode | Samples per row | Layout | Visible width |
|---|---|---|---|
| Highest, no IR | 6000 | RGB | 2000 px |
| Medium, no IR | 4500 | RGB | 1500 px |
| Lowest, no IR | 3000 | RGB | 1000 px |
| Lowest, IR on | 4000 | RGB + IR block | 1000 px |

Measured in the pakon-macos project by Jorge Rangel: https://github.com/jorshhh/pakon-macos.
An 8000-sample row (2000 px × 4, RGB + IR) is observed on the F-135 with IR
on, and is the expected, but not yet measured, F-135+ highest-resolution
IR-on layout. [INFERRED for the F-135+ case]

So the IR lane is present only when Digital ICE is enabled, and the stride is
not a fixed constant; it must be derived from the scan settings (or measured),
not hardcoded.

## The exported raw file

The OEM's "save raw with header" writes a 16-byte header
(`SiPlanarFileHeader`) followed by **planar** pixel data (all R, then all G,
then all B). Note that this on-disk layout is planar even though the USB stream is
interleaved; the OEM de-interleaves before saving. [CONFIRMED] from binary
analysis of exported files.

Header (16 bytes, little-endian): header size (16), width, height, and bits per
pixel (48 = 3×16-bit). In the tested save configuration (colour correction
off), the exported pixels are 14-bit values with no colour processing applied,
just rotated, framed per exposure, and written planar. [CONFIRMED for that
configuration] The IR channel is not included even when Digital ICE was on.
With colour correction enabled, the OEM configuration files indicate the
export instead contains 12-bit RPD-rendered data, so the unprocessed claim is
scoped to the tested configuration, not to every planar export. [DOCUMENTED]

## Marker bit for row origin

The USB stream can begin mid-row (bulk chunking is not row-aligned), so the
triplet phase floats and, uncorrected, the channel assignment rotates and the
colour cast flips between scans. The scanner marks each scan line by setting the
least-significant bit of one fixed word per line; finding that word recovers the
true row origin and makes the channel assignment phase-independent. The
`R, G, B` channel identity stated above is the **marker-aligned** order: before
row-origin recovery, an arbitrarily chunked capture has a floating phase, and
phase-zero readings historically produced a spurious `B, R, G` assignment. This
marker-bit behaviour is a property of the scanner firmware; using it for
row-origin recovery was first shown to this author by Stefan Dierauf. It was
applied in a decoder (the pakon-macos project by Jorge Rangel:
https://github.com/jorshhh/pakon-macos) and [CONFIRMED on hardware,
August 2026] phase-invariant across many simulated capture starts.

## Open questions

- The F-135+ highest-resolution IR-on stride (expected 8000) is unmeasured.
- The calibration-phase (pre-film) stream structure is not documented here.
- F-235/F-335 stream geometry is entirely unknown.
