# Conventions

How this reference is written, so that every statement can be trusted and
traced.

## Confidence markers

Every non-obvious fact carries one:

- **[CONFIRMED]** — verified directly: observed on real hardware, read from a
  capture, or proven by running code against a scanner. The strongest claim.
- **[DOCUMENTED]** — stated in original Kodak/Pakon material (firmware
  readmes, INF files, service documentation) or read directly from driver
  source, but not independently verified here.
- **[INFERRED]** — a reasoned conclusion from evidence, not a direct
  observation. May be wrong; the reasoning is given so it can be checked.
- **[SPECULATIVE]** — a hypothesis worth recording but not yet supported.
  Used sparingly.

When a fact is later upgraded (e.g. an inference confirmed on hardware), the
marker changes and the date/means of confirmation is noted inline.

## Sourcing

A fact should say how it is known when that is not obvious: "from a scan
capture", "observed live on an F-135+", "from `ReadmeF135.txt`", "from TLA.dll
decompilation for interoperability", "cross-confirmed by libpakon". The goal
is that a reader can weigh and re-verify each claim.

## Scope: facts, not artifacts

This repository documents facts about the hardware and protocol. It does
**not** contain, and pull requests must not add:

- Kodak/Pakon software, DLLs, or executables
- firmware images or firmware bytes (`.hex`, `.bin`, or inline)
- OEM data files (LUTs, ICC profiles, calibration tables, personality blobs)
- decompiler output or disassembly listings
- scanned OEM documentation (service manuals, etc.)

Stating a *derived fact* — "the ColNeg inversion LUT is exactly
`out = 3500·log10(16383/in)`" — is the point of the repository. Shipping the
file that fact was derived from is not. Numeric constants that constitute an
interoperability fact (register addresses, a formula's coefficients, a
protocol opcode) are facts, not artifacts, and belong here.

## Model names

`F-135`, `F-135+` (the "Plus" / "Hybrid"), `F-235`, `F-335`. "F-X35" refers to
the family. Where a fact is specific to one model, say so; where it is shared,
say that too.

## Numbers and encoding

Hex with a `0x` prefix for protocol/register values (`0x8A`), plain decimal
for counts and measurements. USB IDs as `vvvv:pppp` lowercase (`0f05:f135`).
Multi-byte values state their endianness. Byte sequences are space-separated
hex (`04 03 10 00 85`).

## Dates

Each document's facts are dated to when the underlying research established
them (see the timeline in the README). Commit dates track that research
history, not the transcription date; this is a redacted public view of a
private repository whose real commit history cannot be published.
