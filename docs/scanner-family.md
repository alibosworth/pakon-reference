# Scanner family differences

What differs across the F-135, F-135+, F-235, and F-335. Same host software
API (the TLX COM interface), same FX2 bridge chip and cold identity; the
differences are in the controller boards behind the bridge, and in which
per-model scanner-control library the host software loads (see
[The OEM host software stack](#the-oem-host-software-stack)).

_From the OEM firmware inventory and the per-model firmware readmes
(`ReadmeF135.txt`, `ReadmeF235.txt`, `ReadmeF335.txt` in
`F-X35 COM SERVER\Config\Firmware\`), and the PICL Plus disassembly, May 2026.
The host-software section is from the OEM's F235 SDK manuals (June 2004) and
the string tables of the shipped COM server DLLs, August 2026. OEM file paths are
relative to a standard Windows install; "the readmes" below refers to those
three files._

## The OEM host software stack

The OEM host software is a COM server, installed as `F-X35 COM SERVER`, that
the OEM applications (PSI, IQ) and the sample client `TLXClientDemo` drive.
It is layered: one COM façade over three per-model engine builds.

Two things about it are not what the names suggest, and both have caught
projects out:

- **The letters A/B/C do not follow the model numbers 135/235/335.** The
  135 line is served by `TLB.dll`, the F-235 by `TLA.dll`, the F-335 by
  `TLC.dll`. The letters follow build order: TLA was the original F235 SDK
  engine, TLB and TLC were added later.
- **The F-135 and the F-135+ share one engine, `TLB.dll`, even though their
  controller boards differ.** The per-generation differences (bus addresses
  `0x20`/`0x24` vs `0x40`/`0x44`, the TEC, the LED-current ceilings) are
  handled inside that one build, keyed on which controller addresses answer
  the start-up probes.

| Layer | File | Role |
|---|---|---|
| Façade | `tlx.dll` | The one COM class clients bind (`Tlx.TLXMain`). Selects and wraps a per-model engine at runtime. |
| Engine, 135 line | `TLB.dll` | Scanner control + imaging for the **F-135 and F-135+**: one build that knows both controller address sets (PICL/PICM at `0x20`/`0x24`, PICL+/PICM+ at `0x40`/`0x44`). |
| Engine, F-235 | `TLA.dll` | Scanner control for the **F-235** family (F235plus/F235C): multi-board architecture with APS reader, dedicated DX board, lamp, CCD and motor boards; MOF reading. |
| Engine, F-335 | `TLC.dll` | Scanner control for the **F-335** family: TLA's twin with **LED** illumination in place of the lamp. |
| Image processing | `PakonIMAu.dll` (Ansel), `DMLDICELib.dll` (Digital ICE) | Loaded by the engines. |

[CONFIRMED] from the shipped binaries (August 2026; `tlx.dll`, `TLA.dll`,
`TLB.dll`, `TLC.dll` all file version 3.1.0.28, the final OEM release). The
TLA/TLB/TLC = F-235/F-135/F-335 file mapping was stated by the pakon-mac
project (https://github.com/gazzdingo/pakon-mac, `docs/05-source-material.md`)
before it was established here; it is confirmed here independently from the
DLLs' own string tables, and extended to the F-135+ (one 135-line build). The engines
share their SDK type library and error tables (every build carries the full model
enum `F135`/`F135_PLUS`/`F235`/`F235C`/`F335`/`F335C` and the same
`EC_*` codes), so those do not tell the builds apart. What does is each
build's **bus-address name table** and **kernel device**:

- `TLB.dll` names its bus participants `AD_HOST`, `AD_PICL`, `AD_PICM`,
  `AD_PICL_PLUS`, `AD_PICM_PLUS` (and `AD_BOOT_*` bootloader twins), and
  opens `\\.\Pakon135`. It is the only engine that knows the PICL/PICM
  architecture, and it knows both generations of it.
- `TLA.dll` names `AD_HOST`, `AD_APS_READ`, `AD_DX_READ`, `AD_CCD1`,
  `AD_LAMP`, `AD_MOTOR` (plus `AD_BOOT_*`), and carries the APS
  cartridge and MOF machinery: the F-235's per-function board layout.
- `TLC.dll` names the same set with `AD_LED_LAMP` in place of `AD_LAMP`,
  and opens `\\.\Pakonx35` (the FX35 driver family): the F-335, whose
  illumination is LED.
- `TLXClientDemo` makes exactly one `CoCreateInstance`, of `Tlx.TLXMain`;
  `tlx.dll` holds wrapper classes and error codes for all three engines
  (`CTLAWrapper`/`CTLBWrapper`/`CTLCWrapper`,
  `EC_CoCreateInstanceTLA/TLB/TLC`), so engine selection is tlx.dll's job.
  The PakonClient project (Henri Toivonen,
  https://github.com/eatfrog/PakonClient, `docs/tlx-lowlevel.md`) recovered
  the dispatch from `tlx.dll` itself: it probes `\\.\Pakon135` and, if
  present, creates CLSID `{52B5538B-7926-40AD-9DBE-810228E147AD}`; it
  probes `\\.\PakonX35` and creates `{6449DE65-60A9-4A45-A3A1-337F5E6B41E0}`.
  Every engine registers a class *named* `TLAMain`, so the name does not
  identify the server; the CLSID bytes do. `{52B5538B-…}` is embedded in
  `TLB.dll` and in no other engine; `{6449DE65-…}` in `TLC.dll` only
  (checked on TLB.dll 3.1.0.28 sha256 5866ec56…, TLC.dll 3.1.0.28
  9af71627…, TLA.dll 3.1.0.28 bbcf954e…; the CLSID constant sits in
  `tlx.dll` 3.1.0.28 at 0x10021770 and is loaded by the `Pakon135` probe
  routine at 0x10005bcc). That is the façade's own routing: `Pakon135` (F-135/F-135+) to TLB,
  `PakonX35` (F-335) to TLC.
- Cross-confirmed on hardware, both generations: an OEM PPB debug trace of
  the OEM software driving an F-135+ (serial 16402, April 2026) names the
  controllers `AD_PICL_PLUS`/`AD_PICM_PLUS`, identifiers present only in
  `TLB.dll`; and an independent project (pakon-tlx-macos, by Pablo Navarro,
  https://github.com/pablonavarrob/pakon-tlx-macos) runs the unmodified OEM
  stack against a physical original F-135 with `tlx.dll` and `TLB.dll`
  only; `TLA.dll` and `TLC.dll` are never loaded.

**Naming history (read this before trusting a bare "TLA").** Pakon shipped
the F235 with a software development kit for writing your own scanning
application against its COM interface, documented in two manuals: the
"F235 COM" Programmer's User Guide (Pakon document 124579-I) and Programmer's
Reference Manual (124580-I), both for TLA version 0.0.30.2, dated June 2004.
(They circulate with the PakonClient project's docs, see the credits.) In
that SDK, "TLA" is the name of the *entire F235 SDK*: one `TLA.DLL` in
`C:\Program Files\Pakon\TLA COM Server`, sample client `TLAClientDemo`,
main class `TLAMain`. [DOCUMENTED] The later multi-model release kept that
F235 engine as `TLA.dll`, added `TLB.dll` for the 135 line and `TLC.dll`
for the F-335, put the `tlx.dll` façade in front, and renamed the demo
`TLXClientDemo`, while every build kept the original internal class names
(`CiTLAMain`, `CLSID_TLAMain`; even the 135-line engine's registry flags
carry names like `TlaControlLeds`). [INFERRED from the string tables and
registry layout; corroborated by the builds' own copyright strings: TLA
2001–2002, TLB 2003–2004, TLC 2004, tlx 2005] So "TLA" in an internal name
means "the SDK", not "the F-235 build".

**Consequence for readers:** a fact sourced to `TLB.dll` is about the
F-135/F-135+; a fact sourced to `TLA.dll` is about the **F-235** and must
not be assumed to hold on the 135 line (and vice versa), even where the two
engines share code. Where an earlier passage of this reference cites
"TLA.dll" for a 135-line fact, read it as an attribution to be re-checked
against `TLB.dll`; the PPB and command-set facts here were established
from captures of 135-line hardware and stand on that evidence.

The selection mechanism is the same probe for all three: `tlx.dll` tries to
open a kernel device by name and creates the engine registered for it.
`\\.\Pakon135` selects TLB, `\\.\PakonX35` selects TLC, and
`\\.\Loopback` selects TLA (CLSID `{8DDDFE2E-EF6D-46A6-821B-4D3EE6380B3B}`,
embedded in `TLA.dll` only; probe routine at 0x10003db6). Which device name
exists is fixed by which OEM kernel driver build is installed: per the FX35
driver source, the F135 build publishes `PAKON135`, the F335 build
`PAKONX35`, and the F235 build `LOOPBACK`. So the engine is chosen by the
installed driver, before anything is read from the scanner. [CONFIRMED]

Open: whether `TLA.dll` also serves the original (non-plus) F235, or only
the F235plus/F235C the 2004 SDK documents; and how the 135-line engine's
per-generation behaviour differs inside `TLB.dll` (e.g. the two LED-current
ceiling tables it carries, one per controller address set; pakon-tlx-macos,
`docs/PROTOCOL.md`).

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
  software's COM API (`tlx.dll`); the per-model engine DLL beneath it
  differs (see above). [DOCUMENTED]
- **F-135 vs F-135+:** the same FX2 firmware and USB identity (they are
  indistinguishable over USB), but different controller boards. The Plus PICs
  are PIC18 (vs PIC16), sit at bus addresses `0x40`/`0x44` (vs `0x20`/`0x24`),
  add a thermoelectric CCD cooler (TEC, commands `0xD0`/`0xD1`), and integrate
  DX reading into PICL+. On the host side both are driven by the same
  engine build, `TLB.dll`, which carries both address sets (see
  [above](#the-oem-host-software-stack)). [CONFIRMED on hardware, August 2026]: the shared FX2
  firmware, the inverted presence probes, and the Plus init (TEC, per-channel
  exposure) were all verified by the pakon-macos project by Jorge Rangel: https://github.com/jorshhh/pakon-macos,
  driving a real F-135+; see
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
