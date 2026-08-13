# USB identity and firmware loading

The F-X35 scanners are Cypress/Anchor **EZ-USB FX2** devices. On power-on they
come up running a bootstrap firmware and must be handed a second-stage image
over USB before they operate as a scanner. The USB identity does not name the
model: the model is a property of the internal controllers, discovered later
over the command protocol.

_Established from analysis of the OEM Windows driver and firmware loader, and
USB captures of the firmware-load sequence, March 2026; firmware file inventory
and PICL Plus disassembly added May 2026. OEM file paths below are given
relative to a standard Windows install of the Pakon software._

## Cold and warm identity

| State | USB VID:PID | Meaning |
|---|---|---|
| Cold (bootstrap) | `0f05:f235` | unloaded family bootstrap; needs firmware |
| Warm (operational) | `0f05:f135` | operational F-135-family scanner |
| Warm | `0f05:35f2` | operational F-235 |
| Warm | `0f05:f335` | operational F-335 |

[DOCUMENTED] `0f05` is the vendor ID across the family; the cold `f235` and
warm `f135` IDs are read from the driver/INF and observed in captures. A
consequence worth stating plainly: **the USB PID is set by the running
firmware, not the physical model.** The F-135 and F-135+ present identical cold
and warm IDs and cannot be distinguished from USB descriptors; the model is
only knowable from the command-protocol presence probes (see
[ppb-protocol.md](ppb-protocol.md)). [INFERRED] from the shared identity and
the personality mechanism below.

## The personality mechanism

Every family member cold-enumerates the same way. The driver reads an 8-byte
**personality** structure from the device with vendor request `0xA9`:

```
[id] [vendorId:2] [productId:2] [revision:2] [extra]
```

and selects the firmware image by the key `<productId>_<revision>`. The
revision distinguishes the model families:

| Personality key | Firmware image | Family |
|---|---|---|
| `F235_AA05` | `FX35Driver\Pakon5.hex` | F-235 |
| `F235_AA07` | `FX35Driver\Pakon7.hex` | F-135 (and F-135+) |
| `F235_AA08` | `FX35Driver\Pakon8.hex` | F-335 |

[DOCUMENTED] from the OEM INF file and the firmware loader source; the FX2
images ship in the driver install directory (`FX35Driver\`), with a shared
bootstrap stage `FX35Driver\PknInit.hex`. Note that
the F-135 and F-135+ share one key (`F235_AA07`) and therefore one FX2
firmware image: the FX2 is a plain USB-to-bus bridge, and everything
model-specific lives behind it on the internal controller boards, which carry
their own firmware and are not loaded over USB at power-on. [INFERRED] from the
shared personality and the bridge architecture. [CONFIRMED on hardware,
August 2026] a real F-135+ cold-enumerates `0f05:f235`, and the F-135's
`Pakon7.hex` image (replayed from a firmware-load capture) brings it up as
operational `0f05:f135`: the Plus and the F-135 use the same FX2 image.

## The firmware download

The load is the standard FX2 (EZ-USB) sequence, over USB control transfers:

1. hold the 8051 in reset (write the CPUCS register)
2. download a bootstrap stage to internal RAM (vendor request `0xA0`)
3. release reset; the bootstrap enables external-RAM writes
4. read the personality (`0xA9`) and select the model image
5. download the main image (internal `0xA0` + external `0xA3` RAM)
6. reset again to run it; the device re-enumerates as the operational scanner

[DOCUMENTED] from the firmware-load capture and the loader source. The CPUCS
register is at `0x7F92` on the older EZ-USB and `0xE600` on the FX2. The
bootstrap and main images are Intel HEX files in the OEM package; this
reference documents the *sequence*, not the firmware bytes (see
[CONVENTIONS.md](../CONVENTIONS.md)).

## Controller firmware

Behind the FX2 bridge, each scanner's controllers run their own firmware,
programmed at the factory and not part of the USB power-on load:

| Model | Light/CCD | Motor | Notes |
|---|---|---|---|
| F-135 | PICL (PL-series, PIC16) | PICM (PM-series, PIC16) | |
| F-135+ | PICL+ (NL-series, PIC18) | PICM+ (NM-series, PIC18) | adds TEC, integrated DX |
| F-235 | separate CCD board | separate motor board | dedicated DX board |
| F-335 | separate CCD board | separate motor board | LED illumination, dedicated DX board |

[DOCUMENTED] from the OEM firmware readmes (`F-X35 COM SERVER\Config\Firmware\`,
which holds the PIC images: `PL`/`PM` for the F-135, `NL`/`NM` for the F-135+,
and the separate-board prefixes for the larger models). The F-135+ PICL Plus
firmware (`NL050A.HEX`) disassembles as PIC18 (4-byte absolute addressing,
config registers at `0x300000`), confirming the architecture step up from the
F-135's PIC16. [CONFIRMED] by disassembly, May 2026. See
[scanner-family.md](scanner-family.md) for the full per-model breakdown.
