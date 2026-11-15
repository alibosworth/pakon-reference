# Conventions

How this reference is written, so that every statement can be trusted and
traced.

## Confidence markers

Every non-obvious fact carries one:

- **[CONFIRMED]**: verified directly: observed on real hardware, read from a
  capture, or proven by running code against a scanner. The strongest claim.
- **[DOCUMENTED]**: stated in original Kodak/Pakon material (firmware
  readmes, INF files, service documentation) or read directly from driver
  source, but not independently verified here.
- **[INFERRED]**: a reasoned conclusion from evidence, not a direct
  observation. May be wrong; the reasoning is given so it can be checked.
- **[SPECULATIVE]**: a hypothesis worth recording but not yet supported.
  Used sparingly.

When a fact is later upgraded (e.g. an inference confirmed on hardware), the
marker changes and the date/means of confirmation is noted inline.

## Sourcing

A fact should say how it is known when that is not obvious: "from a scan
capture", "observed live on an F-135+", "from `ReadmeF135.txt`", "from TLA.dll
decompilation for interoperability", "cross-confirmed by libpakon". The goal
is that a reader can weigh and re-verify each claim.

When the source is one of the OEM engine DLLs, name the file, because the
engines are per-model builds: `TLB.dll` is the F-135/F-135+ engine,
`TLA.dll` the F-235's, `TLC.dll` the F-335's (see
[scanner-family.md](scanner-family.md#the-oem-host-software-stack)). A
behaviour read from `TLA.dll` is an F-235 fact until it is checked in
`TLB.dll`; "TLA" inside class or registry names is the SDK's name, not the
build's.

Cite OEM binaries with their PE file version so a claim can be re-checked
against the same build, e.g. "TLB.dll 3.1.0.28". The final OEM release
ships every engine, the `tlx.dll` façade and `TLXClientDemo.exe` at
3.1.0.28, `PakonIMAu.dll` at 3.0.0.22, `DMLDICELib.dll` at 1.0.0.1. Where a
byte-level claim depends on it (an address, an embedded constant), add the
first characters of the file's SHA-256; note that installed copies of the
engine DLLs are sometimes patched by tooling, so hash the file you actually
analysed.

## Scope: facts, not artifacts

This repository documents facts about the hardware and protocol. It does
**not** contain, and pull requests must not add:

- Kodak/Pakon software, DLLs, or executables
- firmware images or firmware bytes (`.hex`, `.bin`, or inline)
- OEM data files (LUTs, ICC profiles, calibration tables, personality blobs)
- decompiler output or disassembly listings
- scanned OEM documentation (service manuals, etc.)

Stating a *derived fact* ("the ColNeg inversion LUT is exactly
`out = 3500·log10(16383/in)`") is the point of the repository. Shipping the
file that fact was derived from is not. Numeric constants that constitute an
interoperability fact (register addresses, a formula's coefficients, a
protocol opcode) are facts, not artifacts, and belong here.

One narrow exception, under [Resources](resources/dx-product-code-table.md):
the OEM's DX product-code table is carried in full, both reformatted and as
the original text file. The rule above exists to keep Kodak's own work out of
this repository, and this table is not that: its own header says the speeds
are those "publish[ed] by PIMA", the industry body that assigns DX codes, so
the content is a transcription of a public standard's assignments rather than
anything the OEM devised. It is plain text, it contains no code, no
calibration data particular to a scanner, and nothing that could damage a
unit. A reference that told you a DX code resolves to an ISO speed but not
which speed would be failing at the one job this page has. Anything else in
the artifact list stays out.

## Scope: what things do, not how the OEM coded it

The dividing line for what belongs here is **WHAT vs HOW**, behaviour and
algorithm versus code structure:

- **In scope, behaviour and algorithm:** that a black-box reimplementation
  would want to reproduce: how the scanner behaves and its protocol; the
  processing maths; and the OEM software's *processing choices that affect the
  output* (for example, whether the inversion is tuned by film type or speed,
  how per-film contrast classes are selected, how roll-adaptive scene balance
  works, the order of the processing stages).
- **In scope, a parameter's exact value when it materially changes the
  output:**: a matrix coefficient, a per-film adjustment, a LUT shape. These are
  the difference between matching the OEM result and not.
- **Out of scope, code-structure artifacts with no external effect:** internal
  function/class names, flag bitmasks, enum values, purely-internal buffer
  sizes. These describe how Kodak's *program* was written, not what the scanner
  or the processing does. Cite them as the *source* of a behavioural fact; do
  not document them as facts.

The test: could someone reproduce this behaviour by observing inputs and
outputs, without reading Kodak's code? If yes, it is a documentable fact. If it
is only visible by reading the code (a name, a bitmask, an enum number), it is
a source, not a fact.

### The "A note about the Pakon implementation" callout

Facts about what the OEM host software *does* (as opposed to what the scanner
hardware does) are real and useful (a reimplementation that wants
OEM-matching results needs them) but they must never be mistaken for hardware
or protocol facts. Quarantine them in a clearly labelled callout:

> **A note about the Pakon implementation.** The OEM software does X …

so the reader always knows they are reading an observation about Kodak's
processing choices, not a property of the scanner. Apply the confidence markers
as usual; behavioural claims about the OEM software are frequently [INFERRED]
and should be marked so honestly (a plausible-sounding tuning rule may be
wrong until verified).

## Writing

The reference follows the writing rules of ASD-STE100 (Simplified Technical
English). A short sentence with one meaning is hard to misread, and some pages
tell people what not to send to a scanner. The STE dictionary is not used: the
ASD issues the specification only on request, so contributors cannot be
expected to check every word against it. These rules apply to new and edited
text. Older pages are brought into line when they are next changed.

### Sentences

- Write one fact or one instruction in each sentence.
- Keep an instruction to 20 words or fewer, and a description to 25 or fewer.
  A longer sentence usually holds two facts. Split it.
- Write full sentences, with their articles and verbs. "Not a stop." is a
  fragment. "This command does not stop the motor." is a sentence.
- Use the active voice when the actor is known. "The host sends `0x92`" is
  clearer than "`0x92` is sent".
- Write an instruction as an imperative: "Read the EEPROM before you change it."
- Write a sequence of steps as a numbered list, not a chain of arrows.

### Terms

- Use one word for one thing, and one meaning for each word. If a page
  defines a term, use it only with that meaning. Do not change words for
  variety.
- Use the name the reference already uses for a command or a register. If a
  new source gives a better name, change it everywhere in the same change,
  and say so in the changelog.
- Say which thing does an action. "The scanner" and "the OEM software" are
  different actors. A sentence about when a scan ends must say which of them
  ends it.

### Numbers and claims

- Give the value, not a comparison. "2083333 Hz" is better than "2.5× faster".
  If the ratio matters, give both.
- Make the words match the evidence. The confidence marker gives the level,
  and the sentence must not claim more than the marker does.
- Do not use words that add emphasis and no information: "clearly",
  "critical", "simply", "robust".

### Warnings

- Put a warning before the step it applies to, not after.
- Say what happens if the reader ignores it: "Do not reset the FIFOs during a
  scan. The bus stops responding until you turn the scanner off and on."

### Punctuation

- Do not use em dashes. Use a colon before an explanation. Otherwise use a
  comma, a full stop or parentheses.
- Do not start every bullet with a bold label.

### Corrections and the changelog

- A page states the current fact. Do not leave notes such as "this page used
  to say" on the page. The history goes in the [changelog](CHANGELOG.md).
- A note stays on the page only when a reader may have acted on the old
  claim, and the old claim was unsafe. An example is a command documented as a
  stop that turns the motor in reverse.
- A changelog entry says what the page said before, what it says now, and why
  the old claim was wrong. One line is not enough for a reversal.

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
them (see the [project timeline](index.md#provenance-and-timeline)). Commit dates track that research
history, not the transcription date; this is a redacted public view of a
private repository whose real commit history cannot be published.
