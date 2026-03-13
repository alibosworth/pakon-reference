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
