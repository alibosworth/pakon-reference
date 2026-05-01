# The DX barcode subsystem

35mm film carries a **DX barcode** along its edge, encoding the film product
and speed. The scanners read it with a dedicated infrared sensor and use it to
pick film-specific processing. This subsystem is independent of the visible
image path.

_From reverse engineering of the OEM DX handling and the product-code table,
March 2026; product-code decoding cross-checked May 2026._

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
file header.

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

## Note on the reference hardware

On the specific F-135+ used for much of this research (serial 16402), DX
reading fails. In the captured sensor responses examined (three reads,
including one during an active DX scan), the DX-related fields were exactly
zero while other leading bytes were nonzero. [CONFIRMED] for those captured
responses. A failed DX IR sensor on that unit is the suspected explanation,
not a proven failure mode. [INFERRED]

## Follow-up work

The open DX questions all trace to the reference unit's (serial 16402)
non-functioning DX reads, so the unlock is another scanner with a working DX
sensor:

- capture the DX command exchange (start scan, pot adjustment, sensor reads)
  on a unit that decodes film successfully, to fill in the response payloads
  the failed unit could not provide;
- confirm which bytes of the 30-byte sensor response carry the decoded
  product/generation values, and how "no film" and "decode failed" differ on
  a healthy sensor;
- establish whether the DX result arrives in the polled sensor state, as an
  event, or both. This is a fault of the
individual scanner, not a protocol fact, but it shaped which DX facts could be
verified against live hardware versus derived from the OEM software.

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
