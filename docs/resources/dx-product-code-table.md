# The OEM's DX product-code table

The OEM software resolves a decoded DX code to an ISO speed through a lookup
table it ships at
`F-X35 COM SERVER\anselinstalldir\dataPathItems\common\common-ProdCodeTable.dpi`.
This page states what that table contains, so an implementation can resolve a
code the same way without the file.

_[DOCUMENTED] from the table file. Its own header says the speeds are "as
publish[ed] by PIMA and known by internal sources", with revision comments
running from September 2000 to 6 June 2006. The product-line labels below are
the file's own comments, kept verbatim including their spelling
(`FUJICOLOHR`, `Illford`), because they are how the file identifies each
block and changing them would make it harder to check this page against the
original._

**The original file is here:**
[`common-ProdCodeTable.dpi.txt`](common-ProdCodeTable.dpi.txt). It is carried
in full as the [narrow exception](../CONVENTIONS.md#scope-facts-not-artifacts)
to this repository's rule against OEM data files: the assignments are PIMA's,
the format is plain text, and a lookup table is no use to anyone in summary
form. The same table is published, with the same file, at
[pprc](https://pprc.alibosworth.com/resources/pakon-f135-dx-code-lookup-table/).

## How to read it

A decoded DX code is two numbers: **part 1**, the product code (7 bits,
0–127), and **part 2**, the generation code (4 bits, 0–15). Together they
select one cell of a 128 × 16 grid, and the value in that cell is the film's
ISO speed. See [dx-barcode.md](../dx-barcode.md) for how the two numbers are
carried in the scanner's protocol, and
[35mm-dx-edge-code](https://github.com/alibosworth/35mm-dx-edge-code) for the
structure of the barcode on the film.

Most of the grid is zero, which the OEM treats as "no entry": the scan gets a
default speed rather than a film-specific one. Only the assigned cells are
listed here.

## What it means for film sold now

The table's last revision is dated 6 June 2006, so **any film introduced
after that has no entry**, and several current films encode to cells that
were never assigned. Portra 400 encodes as 95/14, which is not in the table.
Motion-picture stock respooled into cassettes carries no DX barcode at all,
so nothing resolves. In both cases the OEM falls back to its default, which
is why a film-specific ISO cannot be assumed from a scan.

To look up what a specific film actually encodes, including modern emulsions,
[The Big Film Database](https://thebigfilmdatabase.merinorus.com) holds
community-decoded entries.

## The assigned entries

The table assigns a speed to 361 of the 2048 cells.

| Part 1 (product) | Part 2 (generation) | ISO | Product line, as the file names it |
|---|---|---|---|
| 2 | 0 | 100 | KONICA |
| 2 | 1 | 100 | KONICA |
| 2 | 2 | 100 | KONICA |
| 2 | 3 | 100 | KONICA |
| 2 | 4 | 200 | KONICA |
| 2 | 5 | 200 | KONICA |
| 2 | 6 | 200 | KONICA |
| 2 | 7 | 100 | KONICA |
| 2 | 8 | 400 | KONICA |
| 2 | 9 | 400 | KONICA |
| 2 | 10 | 1600 | KONICA |
| 2 | 11 | 3200 | KONICA |
| 2 | 12 | 100 | KONICA |
| 2 | 13 | 100 | KONICA |
| 2 | 14 | 100 | KONICA |
| 8 | 1 | 200 | FUJICOLOR SUPERIA HR |
| 8 | 4 | 100 | FUJICOLOR SUPERIA HR |
| 8 | 5 | 100 | FUJICOLOR SUPERIA HR |
| 8 | 6 | 160 | FUJICOLOR SUPERIA HR |
| 8 | 7 | 100 | FUJICOLOR SUPERIA HR |
| 8 | 9 | 200 | FUJICOLOR SUPERIA HR |
| 8 | 12 | 100 | FUJICOLOR SUPERIA HR |
| 8 | 13 | 200 | FUJICOLOR SUPERIA HR |
| 8 | 14 | 100 | FUJICOLOR SUPERIA HR |
| 8 | 15 | 100 | FUJICOLOR SUPERIA HR |
| 10 | 2 | 400 | FUJICOLOHR REALA |
| 10 | 3 | 200 | FUJICOLOHR REALA |
| 10 | 4 | 100 | FUJICOLOHR REALA |
| 10 | 5 | 100 | FUJICOLOHR REALA |
| 10 | 6 | 100 | FUJICOLOHR REALA |
| 10 | 7 | 100 | FUJICOLOHR REALA |
| 10 | 8 | 1600 | FUJICOLOHR REALA |
| 10 | 9 | 400 | FUJICOLOHR REALA |
| 10 | 10 | 400 | FUJICOLOHR REALA |
| 10 | 11 | 200 | FUJICOLOHR REALA |
| 10 | 12 | 100 | FUJICOLOHR REALA |
| 10 | 13 | 100 | FUJICOLOHR REALA |
| 10 | 14 | 100 | FUJICOLOHR REALA |
| 10 | 15 | 100 | FUJICOLOHR REALA |
| 12 | 2 | 400 | FUJICOLOR SUPER |
| 12 | 3 | 200 | FUJICOLOR SUPER |
| 12 | 4 | 100 | FUJICOLOR SUPER |
| 12 | 6 | 160 | FUJICOLOR SUPER |
| 12 | 7 | 100 | FUJICOLOR SUPER |
| 12 | 9 | 800 | FUJICOLOR SUPER |
| 12 | 10 | 400 | FUJICOLOR SUPER |
| 12 | 11 | 200 | FUJICOLOR SUPER |
| 12 | 12 | 100 | FUJICOLOR SUPER |
| 12 | 13 | 100 | FUJICOLOR SUPER |
| 12 | 14 | 160 | FUJICOLOR SUPER |
| 12 | 15 | 400 | FUJICOLOR SUPER |
| 17 | 0 | 100 | AGFACOLOR OPTIMA HDC |
| 17 | 1 | 100 | AGFACOLOR OPTIMA HDC |
| 17 | 2 | 100 | AGFACOLOR OPTIMA HDC |
| 17 | 3 | 50 | AGFACOLOR OPTIMA HDC |
| 17 | 4 | 200 | AGFACOLOR OPTIMA HDC |
| 17 | 6 | 200 | AGFACOLOR OPTIMA HDC |
| 17 | 7 | 400 | AGFACOLOR OPTIMA HDC |
| 17 | 8 | 400 | AGFACOLOR OPTIMA HDC |
| 17 | 10 | 1000 | AGFACOLOR OPTIMA HDC |
| 17 | 11 | 200 | AGFACOLOR OPTIMA HDC |
| 17 | 12 | 100 | AGFACOLOR OPTIMA HDC |
| 17 | 13 | 200 | AGFACOLOR OPTIMA HDC |
| 18 | 5 | 400 | SCOTCH |
| 18 | 6 | 100 | SCOTCH |
| 18 | 7 | 100 | SCOTCH |
| 18 | 9 | 200 | SCOTCH |
| 18 | 10 | 400 | SCOTCH |
| 26 | 0 | 400 | KONICA CENTURIA |
| 26 | 1 | 800 | KONICA CENTURIA |
| 26 | 2 | 100 | KONICA CENTURIA |
| 26 | 3 | 200 | KONICA CENTURIA |
| 26 | 4 | 1600 | KONICA CENTURIA |
| 26 | 5 | 400 | KONICA CENTURIA |
| 26 | 6 | 800 | KONICA CENTURIA |
| 28 | 3 | 400 | KONICA COLOR VX |
| 28 | 4 | 400 | KONICA COLOR VX |
| 28 | 12 | 100 | KONICA COLOR VX |
| 28 | 13 | 200 | KONICA COLOR VX |
| 31 | 1 | 200 | AGFACOLOR TURA HR |
| 33 | 0 | 800 | FUJI NEXIA APS |
| 33 | 1 | 400 | FUJI NEXIA APS |
| 33 | 2 | 400 | FUJI NEXIA APS |
| 33 | 3 | 200 | FUJI NEXIA APS |
| 33 | 4 | 100 | FUJI NEXIA APS |
| 33 | 5 | 200 | FUJI NEXIA APS |
| 33 | 6 | 100 | FUJI NEXIA APS |
| 33 | 7 | 100 | FUJI NEXIA APS |
| 33 | 8 | 800 | FUJI NEXIA APS |
| 33 | 9 | 400 | FUJI NEXIA APS |
| 33 | 10 | 400 | FUJI NEXIA APS |
| 33 | 11 | 200 | FUJI NEXIA APS |
| 33 | 12 | 100 | FUJI NEXIA APS |
| 33 | 13 | 200 | FUJI NEXIA APS |
| 33 | 15 | 400 | FUJI NEXIA APS |
| 35 | 0 | 1600 | FUJI SUPERIA 135 |
| 35 | 1 | 800 | FUJI SUPERIA 135 |
| 35 | 2 | 400 | FUJI SUPERIA 135 |
| 35 | 3 | 200 | FUJI SUPERIA 135 |
| 35 | 4 | 160 | FUJI SUPERIA 135 |
| 35 | 5 | 100 | FUJI SUPERIA 135 |
| 35 | 6 | 100 | FUJI SUPERIA 135 |
| 35 | 7 | 100 | FUJI SUPERIA 135 |
| 35 | 8 | 1600 | FUJI SUPERIA 135 |
| 35 | 9 | 800 | FUJI SUPERIA 135 |
| 35 | 10 | 400 | FUJI SUPERIA 135 |
| 35 | 11 | 400 | FUJI SUPERIA 135 |
| 35 | 12 | 100 | FUJI SUPERIA 135 |
| 35 | 13 | 100 | FUJI SUPERIA 135 |
| 35 | 14 | 100 | FUJI SUPERIA 135 |
| 35 | 15 | 400 | FUJI SUPERIA 135 |
| 36 | 0 | 800 | FUJICOLOR REALA ACE SUPER NPZ |
| 36 | 1 | 400 | FUJICOLOR REALA ACE SUPER NPZ |
| 36 | 2 | 400 | FUJICOLOR REALA ACE SUPER NPZ |
| 36 | 3 | 200 | FUJICOLOR REALA ACE SUPER NPZ |
| 36 | 4 | 100 | FUJICOLOR REALA ACE SUPER NPZ |
| 36 | 5 | 800 | FUJICOLOR REALA ACE SUPER NPZ |
| 36 | 6 | 100 | FUJICOLOR REALA ACE SUPER NPZ |
| 36 | 7 | 100 | FUJICOLOR REALA ACE SUPER NPZ |
| 36 | 8 | 400 | FUJICOLOR REALA ACE SUPER NPZ |
| 36 | 9 | 800 | FUJICOLOR REALA ACE SUPER NPZ |
| 36 | 10 | 400 | FUJICOLOR REALA ACE SUPER NPZ |
| 36 | 12 | 100 | FUJICOLOR REALA ACE SUPER NPZ |
| 36 | 13 | 100 | FUJICOLOR REALA ACE SUPER NPZ |
| 36 | 14 | 100 | FUJICOLOR REALA ACE SUPER NPZ |
| 36 | 15 | 100 | FUJICOLOR REALA ACE SUPER NPZ |
| 37 | 2 | 400 | FUJI NEXIA APS |
| 37 | 10 | 400 | FUJI NEXIA APS |
| 40 | 0 | 100 | KONICA |
| 40 | 1 | 100 | KONICA |
| 40 | 2 | 100 | KONICA |
| 40 | 3 | 50 | KONICA |
| 40 | 4 | 200 | KONICA |
| 40 | 5 | 200 | KONICA |
| 40 | 6 | 400 | KONICA |
| 40 | 7 | 400 | KONICA |
| 40 | 8 | 400 | KONICA |
| 40 | 9 | 200 | KONICA |
| 40 | 10 | 200 | KONICA |
| 40 | 11 | 200 | KONICA |
| 40 | 12 | 100 | KONICA |
| 40 | 13 | 200 | KONICA |
| 40 | 14 | 100 | KONICA |
| 43 | 1 | 800 | KODAK MAX/SUPRA 800 |
| 43 | 2 | 800 | KODAK MAX/SUPRA 800 |
| 43 | 3 | 800 | KODAK MAX/SUPRA 800 |
| 43 | 4 | 800 | KODAK MAX/SUPRA 800 |
| 43 | 5 | 800 | KODAK MAX/SUPRA 800 |
| 43 | 6 | 800 | KODAK MAX/SUPRA 800 |
| 43 | 7 | 800 | KODAK MAX/SUPRA 800 |
| 44 | 1 | 100 | AGFACOLOR IX240 FUTURA/II |
| 44 | 4 | 200 | AGFACOLOR IX240 FUTURA/II |
| 44 | 5 | 100 | AGFACOLOR IX240 FUTURA/II |
| 44 | 6 | 200 | AGFACOLOR IX240 FUTURA/II |
| 44 | 9 | 400 | AGFACOLOR IX240 FUTURA/II |
| 44 | 10 | 400 | AGFACOLOR IX240 FUTURA/II |
| 45 | 4 | 200 | AGFACOLOR-PRIVATE TURA COLOR |
| 45 | 6 | 200 | AGFACOLOR-PRIVATE TURA COLOR |
| 45 | 9 | 400 | AGFACOLOR-PRIVATE TURA COLOR |
| 46 | 1 | 100 | Unlabeled in source |
| 49 | 2 | 160 | AGFACOLOR |
| 49 | 3 | 100 | AGFACOLOR |
| 49 | 4 | 200 | AGFACOLOR |
| 49 | 5 | 200 | AGFACOLOR |
| 49 | 7 | 400 | AGFACOLOR |
| 49 | 8 | 400 | AGFACOLOR |
| 49 | 9 | 400 | AGFACOLOR |
| 49 | 11 | 200 | AGFACOLOR |
| 49 | 12 | 100 | AGFACOLOR |
| 49 | 13 | 200 | AGFACOLOR |
| 50 | 0 | 400 | KONICA VX JX CENTURIA |
| 50 | 1 | 400 | KONICA VX JX CENTURIA |
| 50 | 2 | 400 | KONICA VX JX CENTURIA |
| 50 | 3 | 100 | KONICA VX JX CENTURIA |
| 50 | 4 | 200 | KONICA VX JX CENTURIA |
| 50 | 5 | 400 | KONICA VX JX CENTURIA |
| 50 | 6 | 100 | KONICA VX JX CENTURIA |
| 50 | 7 | 400 | KONICA VX JX CENTURIA |
| 50 | 8 | 400 | KONICA VX JX CENTURIA |
| 50 | 9 | 200 | KONICA VX JX CENTURIA |
| 50 | 10 | 100 | KONICA VX JX CENTURIA |
| 50 | 11 | 160 | KONICA VX JX CENTURIA |
| 50 | 12 | 100 | KONICA VX JX CENTURIA |
| 50 | 13 | 200 | KONICA VX JX CENTURIA |
| 50 | 14 | 800 | KONICA VX JX CENTURIA |
| 66 | 5 | 200 | SCOTCH |
| 66 | 6 | 100 | SCOTCH |
| 66 | 10 | 400 | SCOTCH |
| 66 | 11 | 200 | SCOTCH |
| 68 | 8 | 100 | ERA |
| 72 | 0 | 400 | KONICA JX CENTURIA APS |
| 72 | 1 | 200 | KONICA JX CENTURIA APS |
| 72 | 2 | 400 | KONICA JX CENTURIA APS |
| 72 | 3 | 400 | KONICA JX CENTURIA APS |
| 72 | 4 | 200 | KONICA JX CENTURIA APS |
| 72 | 5 | 800 | KONICA JX CENTURIA APS |
| 78 | 0 | 400 | KODAK GOLD MAX VR B&W PJ |
| 78 | 1 | 100 | KODAK GOLD MAX VR B&W PJ |
| 78 | 2 | 200 | KODAK GOLD MAX VR B&W PJ |
| 78 | 3 | 400 | KODAK GOLD MAX VR B&W PJ |
| 78 | 4 | 100 | KODAK GOLD MAX VR B&W PJ |
| 78 | 5 | 200 | KODAK GOLD MAX VR B&W PJ |
| 78 | 6 | 400 | KODAK GOLD MAX VR B&W PJ |
| 78 | 7 | 800 | KODAK GOLD MAX VR B&W PJ |
| 78 | 8 | 400 | KODAK GOLD MAX VR B&W PJ |
| 78 | 9 | 100 | KODAK GOLD MAX VR B&W PJ |
| 78 | 10 | 100 | KODAK GOLD MAX VR B&W PJ |
| 78 | 11 | 200 | KODAK GOLD MAX VR B&W PJ |
| 78 | 12 | 200 | KODAK GOLD MAX VR B&W PJ |
| 78 | 13 | 400 | KODAK GOLD MAX VR B&W PJ |
| 78 | 14 | 200 | KODAK GOLD MAX VR B&W PJ |
| 78 | 15 | 800 | KODAK GOLD MAX VR B&W PJ |
| 79 | 1 | 200 | Unlabeled in source |
| 79 | 2 | 200 | Unlabeled in source |
| 79 | 3 | 100 | Unlabeled in source |
| 79 | 4 | 400 | Unlabeled in source |
| 79 | 5 | 100 | Unlabeled in source |
| 79 | 6 | 100 | Unlabeled in source |
| 79 | 7 | 200 | Unlabeled in source |
| 79 | 8 | 400 | Unlabeled in source |
| 79 | 9 | 400 | Unlabeled in source |
| 79 | 10 | 100 | Unlabeled in source |
| 79 | 11 | 160 | Unlabeled in source |
| 79 | 12 | 100 | Unlabeled in source |
| 79 | 13 | 400 | Unlabeled in source |
| 79 | 14 | 800 | Unlabeled in source |
| 79 | 15 | 400 | Unlabeled in source |
| 80 | 6 | 100 | KODAKCOLGOLD |
| 80 | 7 | 400 | KODAKCOLGOLD |
| 80 | 8 | 200 | KODAKCOLGOLD |
| 80 | 9 | 1600 | KODAKCOLGOLD |
| 80 | 10 | 100 | KODAKCOLGOLD |
| 80 | 11 | 100 | KODAKCOLGOLD |
| 80 | 12 | 200 | KODAKCOLGOLD |
| 81 | 1 | 25 | KODAKCOLGOLD |
| 81 | 2 | 125 | KODAKCOLGOLD |
| 81 | 3 | 1000 | KODAKCOLGOLD |
| 81 | 4 | 400 | KODAKCOLGOLD |
| 81 | 5 | 400 | KODAKCOLGOLD |
| 81 | 6 | 1600 | KODAKCOLGOLD |
| 81 | 7 | 400 | KODAKCOLGOLD |
| 81 | 8 | 25 | KODAKCOLGOLD |
| 81 | 10 | 100 | KODAKCOLGOLD |
| 81 | 11 | 100 | KODAKCOLGOLD |
| 81 | 13 | 400 | KODAKCOLGOLD |
| 82 | 1 | 400 | KODAK GOLD |
| 82 | 2 | 400 | KODAK GOLD |
| 82 | 3 | 400 | KODAK GOLD |
| 82 | 4 | 400 | KODAK GOLD |
| 82 | 5 | 200 | KODAK GOLD |
| 82 | 6 | 100 | KODAK GOLD |
| 82 | 7 | 200 | KODAK GOLD |
| 82 | 8 | 400 | KODAK GOLD |
| 82 | 9 | 200 | KODAK GOLD |
| 82 | 10 | 400 | KODAK GOLD |
| 82 | 11 | 400 | KODAK GOLD |
| 82 | 13 | 200 | KODAK GOLD |
| 82 | 14 | 100 | KODAK GOLD |
| 83 | 2 | 1000 | KODAK GOLD ROYAL GOLD |
| 83 | 4 | 100 | KODAK GOLD ROYAL GOLD |
| 83 | 5 | 200 | KODAK GOLD ROYAL GOLD |
| 83 | 6 | 400 | KODAK GOLD ROYAL GOLD |
| 83 | 8 | 100 | KODAK GOLD ROYAL GOLD |
| 83 | 9 | 100 | KODAK GOLD ROYAL GOLD |
| 83 | 10 | 400 | KODAK GOLD ROYAL GOLD |
| 83 | 11 | 200 | KODAK GOLD ROYAL GOLD |
| 83 | 12 | 100 | KODAK GOLD ROYAL GOLD |
| 83 | 13 | 200 | KODAK GOLD ROYAL GOLD |
| 83 | 14 | 800 | KODAK GOLD ROYAL GOLD |
| 85 | 1 | 100 | IMATION HP-FERRANIA |
| 85 | 2 | 200 | IMATION HP-FERRANIA |
| 85 | 4 | 400 | IMATION HP-FERRANIA |
| 86 | 1 | 100 | IMATION IX240 |
| 86 | 2 | 200 | IMATION IX240 |
| 86 | 4 | 400 | IMATION IX240 |
| 86 | 12 | 200 | IMATION IX240 |
| 86 | 14 | 400 | IMATION IX240 |
| 87 | 1 | 100 | IMATION-FERRANIA SOLARIS |
| 87 | 2 | 200 | IMATION-FERRANIA SOLARIS |
| 87 | 4 | 400 | IMATION-FERRANIA SOLARIS |
| 87 | 5 | 400 | IMATION-FERRANIA SOLARIS |
| 87 | 8 | 800 | IMATION-FERRANIA SOLARIS |
| 87 | 9 | 800 | IMATION-FERRANIA SOLARIS |
| 90 | 8 | 100 | Unlabeled in source |
| 90 | 10 | 400 | Unlabeled in source |
| 90 | 12 | 400 | Unlabeled in source |
| 91 | 1 | 100 | KODAK ADVANTIX |
| 91 | 2 | 200 | KODAK ADVANTIX |
| 91 | 3 | 400 | KODAK ADVANTIX |
| 91 | 5 | 100 | KODAK ADVANTIX |
| 91 | 6 | 200 | KODAK ADVANTIX |
| 91 | 7 | 400 | KODAK ADVANTIX |
| 91 | 8 | 200 | KODAK ADVANTIX |
| 91 | 9 | 200 | KODAK ADVANTIX |
| 91 | 10 | 200 | KODAK ADVANTIX |
| 91 | 11 | 400 | KODAK ADVANTIX |
| 91 | 12 | 100 | KODAK ADVANTIX |
| 94 | 1 | 400 | KODAK ADVANTIX B&W |
| 95 | 1 | 200 | KODAK ROYAL GOLD 135 |
| 95 | 2 | 400 | KODAK ROYAL GOLD 135 |
| 95 | 3 | 400 | KODAK ROYAL GOLD 135 |
| 96 | 1 | 400 | KODAK GOLD 400 GEN 9 |
| 96 | 2 | 400 | KODAK GOLD 400 GEN 9 |
| 96 | 3 | 400 | KODAK GOLD 400 GEN 9 |
| 98 | 1 | 100 | FUDA |
| 98 | 2 | 100 | FUDA |
| 98 | 3 | 100 | FUDA |
| 100 | 1 | 100 | LUCKYCOLOR |
| 100 | 2 | 100 | LUCKYCOLOR |
| 100 | 3 | 200 | LUCKYCOLOR |
| 100 | 4 | 100 | LUCKYCOLOR |
| 100 | 5 | 400 | LUCKYCOLOR |
| 100 | 6 | 200 | LUCKYCOLOR |
| 100 | 8 | 400 | LUCKYCOLOR |
| 100 | 9 | 100 | LUCKYCOLOR |
| 100 | 10 | 100 | LUCKYCOLOR |
| 100 | 11 | 200 | LUCKYCOLOR |
| 100 | 13 | 400 | LUCKYCOLOR |
| 110 | 1 | 400 | Illford |
| 110 | 2 | 400 | Illford |
| 110 | 3 | 400 | Illford |
| 110 | 4 | 400 | Illford |
| 112 | 3 | 400 | Unlabeled in source |
| 112 | 4 | 160 | Unlabeled in source |
| 112 | 6 | 400 | Unlabeled in source |
| 112 | 7 | 400 | Unlabeled in source |
| 112 | 8 | 400 | Unlabeled in source |
| 112 | 9 | 100 | Unlabeled in source |
| 112 | 11 | 400 | Unlabeled in source |
| 112 | 12 | 100 | Unlabeled in source |
| 113 | 1 | 100 | AGFACOLOR HDC/VISTA |
| 113 | 2 | 100 | AGFACOLOR HDC/VISTA |
| 113 | 3 | 200 | AGFACOLOR HDC/VISTA |
| 113 | 4 | 200 | AGFACOLOR HDC/VISTA |
| 113 | 5 | 100 | AGFACOLOR HDC/VISTA |
| 113 | 6 | 200 | AGFACOLOR HDC/VISTA |
| 113 | 7 | 800 | AGFACOLOR HDC/VISTA |
| 113 | 8 | 400 | AGFACOLOR HDC/VISTA |
| 113 | 9 | 400 | AGFACOLOR HDC/VISTA |
| 113 | 10 | 400 | AGFACOLOR HDC/VISTA |
| 115 | 1 | 100 | AGFA-Private label-PERUTZ-KIRKLAND-TURA HR |
| 115 | 2 | 800 | AGFA-Private label-PERUTZ-KIRKLAND-TURA HR |
| 115 | 3 | 200 | AGFA-Private label-PERUTZ-KIRKLAND-TURA HR |
| 115 | 4 | 200 | AGFA-Private label-PERUTZ-KIRKLAND-TURA HR |
| 115 | 5 | 200 | AGFA-Private label-PERUTZ-KIRKLAND-TURA HR |
| 115 | 6 | 200 | AGFA-Private label-PERUTZ-KIRKLAND-TURA HR |
| 115 | 7 | 200 | AGFA-Private label-PERUTZ-KIRKLAND-TURA HR |
| 115 | 8 | 400 | AGFA-Private label-PERUTZ-KIRKLAND-TURA HR |
| 115 | 9 | 400 | AGFA-Private label-PERUTZ-KIRKLAND-TURA HR |
| 115 | 10 | 400 | AGFA-Private label-PERUTZ-KIRKLAND-TURA HR |
| 115 | 11 | 200 | AGFA-Private label-PERUTZ-KIRKLAND-TURA HR |
| 115 | 12 | 100 | AGFA-Private label-PERUTZ-KIRKLAND-TURA HR |
| 120 | 1 | 100 | ORWO |
| 121 | 1 | 200 | ORWO |
| 121 | 6 | 400 | ORWO |
| 122 | 1 | 100 | ORWO |
| 122 | 2 | 200 | ORWO |
| 122 | 3 | 100 | ORWO |
| 122 | 4 | 400 | ORWO |
| 122 | 5 | 200 | ORWO |
| 122 | 7 | 400 | ORWO |

## Open questions

- The file gives one speed per cell and a product-line label per block; it
  does not name individual emulsions, so a cell cannot be turned back into a
  specific film without an outside source.
- Whether the engines for the F-235 and F-335 ship the same table, or a
  differently revised one, has not been checked.
