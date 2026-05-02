# Scanner family differences

What differs across the F-135, F-135+, F-235, and F-335. Same host software
stack (TLX/TLA), same FX2 bridge chip and cold identity; the differences are in
the controller boards behind the bridge.

_From the OEM firmware inventory and the per-model firmware readmes
(`ReadmeF135.txt`, `ReadmeF235.txt`, `ReadmeF335.txt` in
`F-X35 COM SERVER\Config\Firmware\`), and the PICL Plus disassembly, May 2026.
OEM file paths are relative to a standard Windows install; "the readmes" below
refers to those three files._

## At a glance

| | F-135 | F-135+ | F-235 | F-335 |
|---|---|---|---|---|
| USB cold | `0f05:f235` | `0f05:f235` | `0f05:f235` | `0f05:f235` |
| USB warm | `0f05:f135` | `0f05:f135` | `0f05:35f2` | `0f05:f335` |
| FX2 firmware | `Pakon7.hex` | `Pakon7.hex` (same) | `Pakon5.hex` | `Pakon8.hex` |
| Personality | `F235_AA07` | `F235_AA07` | `F235_AA05` | `F235_AA08` |
| Light/CCD PIC | PICL (`PL`, PIC16) | PICL+ (`NL`, PIC18) | separate CCD board | separate CCD board |
| Motor PIC | PICM (`PM`, PIC16) | PICM+ (`NM`, PIC18) | separate motor board | separate motor board |
| Illumination | lamp | lamp | lamp (`LP` board) | **LED** (`LQ` board) |
| CCD cooling (TEC) | no | **yes** | — | — |
| DX reading | in PICL | in PICL+ | dedicated `DX` board | dedicated `DY` board |
| APS (IX240) film | no | no | yes (`AP` board) | yes (`AP` board) |

[DOCUMENTED] from the OEM firmware readmes (`ReadmeF135/235/335.txt`)
and the driver INF unless noted. PIC firmware lives in
`F-X35 COM SERVER\Config\Firmware\` under the prefixes shown.

## Shared vs. distinct

- **Shared across the whole family:** the FX2 bridge chip, the cold identity
  `0f05:f235`, the personality-driven firmware selection, and the host
  software. [DOCUMENTED]
- **F-135 vs F-135+:** the same FX2 firmware and USB identity (they are
  indistinguishable over USB), but different controller boards. The Plus PICs
  are PIC18 (vs PIC16), sit at bus addresses `0x40`/`0x44` (vs `0x20`/`0x24`),
  add a thermoelectric CCD cooler (TEC, commands `0xD0`/`0xD1`), and integrate
  DX reading into PICL+. [DOCUMENTED]/[INFERRED] as noted in
  [ppb-protocol.md](ppb-protocol.md) and
  [usb-identity-and-firmware.md](usb-identity-and-firmware.md).
- **F-235 / F-335:** a different, multi-board architecture with dedicated motor,
  DX, lamp/LED, CCD, and APS boards each with their own firmware. The F-335
  uses LED illumination rather than a lamp. Their bus addressing, command
  dialects, and image-stream geometry are **not documented here** (no captures
  of these models were available). [DOCUMENTED] board makeup from the
  firmware readmes (`ReadmeF235.txt`, `ReadmeF335.txt`) only.

## Firmware prefixes

Each board type has a two-letter firmware prefix in
`F-X35 COM SERVER\Config\Firmware\`:

| Prefix | Board | Model(s) |
|---|---|---|
| `PL` / `PM` | PICL / PICM | F-135 |
| `NL` / `NM` | PICL+ / PICM+ | F-135+ |
| `MC` / `DX` / `LP` / `CD` / `AP` | motor / DX / lamp / CCD / APS | F-235 |
| `MD` / `LQ` / `DY` / `CE` / `AP` | motor / LED / DX / CCD / APS | F-335 |

[DOCUMENTED] from the firmware readmes (`ReadmeF135/235/335.txt`). The
latest F-135+ images are `NL050A.HEX` (PICL
Plus) and `NM0506.HEX` (PICM Plus); the F-135 equivalents are `PL`/`PM` series.

## Porting to an undocumented model

The transport layer very likely carries over to the F-235/F-335 (same FX2
bridge, same driver family, personality-selected firmware). The unknowns are
the multi-board PPB addressing, the command dialects, and the image geometry.
The way to resolve them is a USB or driver-level capture of the OEM software
driving the real machine, the same approach that produced the F-135/F-135+
documentation here. [INFERRED]
