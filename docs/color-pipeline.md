# The colour pipeline

How the OEM turns a raw C-41 negative scan into a positive. The core is two
linked steps (a density conversion and a colour-correction matrix) driven by
two data files. This is the piece most film-scanning tools get wrong, because
the inversion happens in density (log) space, not linear.

_From reverse engineering of the Ansel engine (`PakonIMAu.dll`) and the OEM
colour-correction data files, March 2026; the exact LUT formula and the
pipeline ordering settled by numeric analysis, July 2026. OEM file paths are
relative to a standard Windows install; this reference documents the derived
facts, not the files._

**Engine scope.** The OEM's host software has per-model engine builds
(`TLB.dll` for the F-135/F-135+, `TLA.dll` for the F-235; see
[scanner-family.md](scanner-family.md#the-oem-host-software-stack)), and the
builds differ in the colour-processing stage. The data files described on
this page ship with every build, but which stages a given build actually
applies to colour-negative data, and in what order, is engine-specific.
Analysis of the shipped binaries by the pakon-mac project (Guy Langford-Lee,
https://github.com/gazzdingo/pakon-mac) reports that the LUT→matrix
stage described below is the **F-235 engine's** (applied through the
`PakonIMAu` MMX kernel), while the **135-line engine** applies a different,
per-unit colour transform to colour-negative data and does not call the
LUT+matrix kernel at all (see [Open questions](#open-questions)). Readers
working on an F-135/F-135+ should treat this page as documenting the shared
data files and the F-235 path until the 135-line path is documented here.

## The inversion LUT, exactly

The OEM's negative→positive tone conversion is a single 14-bit (16,384-entry)
1-D lookup table (`F-X35 COM SERVER\Config\ColorCorrection\_ClientColNegLut.txt`).
Numeric analysis shows it is, exactly:

```
D(x) = 3500 · log10(16383 / x)          x = scanner linear code, 1..16383
```

with `D(0)` clamped to 16383. [CONFIRMED] the maximum residual against all
16,383 table entries is 5×10⁻⁵ (float rounding). [July 2026]

What this means precisely:

- The LUT converts **scanner-linear transmission code to optical density**,
  scaled 3500 counts per decade of density, anchored so CCD full scale
  (16383 = maximum transmission) reads density 0.
- It is a **pure log/density conversion**: there is no film-characteristic
  shaping and no tone rendering in it. Its output is a scene-log-exposure-like
  encoding, not a finished positive.
- "Inverts the negative" is true only in the sense that density is inversely
  related to transmission: a dense part of the negative (a scene highlight) maps
  to high density. [CONFIRMED by the formula]

12-bit (4096-entry) per-film-product LUT variants exist for the `FilmLut`
stage. [DOCUMENTED]

## The colour matrix

After the LUT, a 3×4 affine transform
(`Config\ColorCorrection\_ClientColNegMat.txt`) is applied in density space:

```
R' =  1.119·R − 0.101·G − 0.012·B −  82.6
G' = −0.201·R + 1.101·G + 0.117·B − 586.9
B' = −0.117·R + 0.048·G + 1.083·B − 707.8
```

[DOCUMENTED] the coefficients; [CONFIRMED July 2026] the domain and ordering:

- **Order is LUT → matrix.** The offsets are density-unit quantities: they are
  meaningless before the LUT and exact after it.
- **The 3×3 is the classical masking-equation correction in density space**:
  it removes C-41 dye unwanted-absorption crosstalk (the orange mask's colour
  cast). Its row sums are ≈ 1.01, so neutral density maps to neutral density:
  it corrects colour crosstalk without re-grading neutrals.
- **The offset column is Dmin (film-base) subtraction in density units.**
  Dividing by 3500 gives R 0.024 D, G 0.168 D, B 0.202 D, the orange mask's
  base-density signature (blue densest), measured through this scanner's
  per-channel exposure calibration. The offsets are therefore entangled with
  the scanner's exposure/gain calibration; the 3×3 is not.

## Pipeline placement

The Ansel colour-negative path (`AnsColorNegativePath`) runs many stages;
relevant to the inversion, the order is: scanner correction (gain/offset,
lens-falloff) on the raw linear data → the density LUT → the crosstalk/Dmin
matrix → output colour-space encoding. The scanner-correction and
output-encoding stages are separate from the two inversion steps above.
[DOCUMENTED]/[INFERRED] stage names are confirmed from the DLL; the exact
execution order within the engine is inferred from dependencies.

## Practical restatement for implementers

The single invariant: film-base (Dmin) compensation happens exactly once. The
OEM performs it inside the matrix stage. Its offset column subtracts the
film-base densities, on the assumption that the input is on the OEM's
calibrated linear scale (clear base near full scale). [DOCUMENTED]

For input that is not on the OEM's calibrated scale, those baked offsets do not
transfer. The film base must instead be established for the actual input (for
example, measured from the scan itself) and compensated in one place: either
normalise in the linear domain before the LUT, or derive new density-domain
offsets for the matrix. Combining linear-domain normalisation with the OEM
offset column applies the compensation twice.

The off-diagonal 3x3 terms are scale-independent in a way the offsets are not
(a global density scale commutes with the matrix, per the numeric analysis
above), so they remain applicable to re-based input. They are the part that
pure per-channel processing cannot express: the actual dye-mask correction.

Output-colourspace encoding (sRGB or a rendering profile) is a final, separate
step.

One point in this space is verified: measured linear-domain Dmin
normalisation, then the LUT, then sRGB, with the matrix omitted entirely,
reproduces the OEM positive closely on real scans. [CONFIRMED on hardware,
August 2026, in the pakon-macos project by Jorge Rangel:
https://github.com/jorshhh/pakon-macos] The full OEM matrix path has not been
reproduced outside the OEM software.

## Rendering profiles

Beyond the faithful inversion, the OEM offers "vibrant" rendering through Kodak
ICC profiles (e.g. an RPD rendering profile) plus scene-balance and highlight
handling, at `F-X35 COM SERVER\anselinstalldir\icc\`. These are a look applied
on top of the inversion, not part of it. [DOCUMENTED]

## Open questions

- The full OEM matrix path has not been reproduced outside the OEM software.
- The OEM's film-specific layer (per-film LUTs and profiles, film-class
  selection, scene balance) is not yet documented here.
- **The 135-line colour path.** The pakon-mac project's analysis of the
  shipped engines (https://github.com/gazzdingo/pakon-mac) reports that
  `TLB.dll` (F-135/F-135+) applies a per-unit second-order
  polynomial (3×10 coefficients, stored as `NegMatrix0-29` in the scanner's
  calibration EEPROM and mirrored to the registry) to colour-negative data,
  and that the density LUT + 3×4 matrix kernel documented on this page has
  no call site in that build; the psix project (Oliver Roch,
  https://github.com/veroc/psix) independently measured the OEM's F-135+
  output transfer as per-channel with no post-process 3×3, and the
  pakon-tlx-macos project (Pablo Navarro,
  https://github.com/pablonavarrob/pakon-tlx-macos) records the same 3×10
  matrix in the F-135's registry. That polynomial's exact form, where the
  negative→positive inversion then happens on the 135 line, and how the
  Ansel stages consume the result are not yet documented here. Until they
  are, the pipeline on this page is the F-235's, and 135-line
  reimplementations should not assume it.
