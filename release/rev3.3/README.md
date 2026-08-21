# Burnwing Rev 3.0 — production files, rev3.3

Fabrication and assembly output for the Burnwing Rev 3.0 board, generated from
commit `caff272`.

Burnwing is a dual-channel thermal knife release mechanism for CubeSats and
small satellites. It ships as one panel carrying two halves joined by mouse
bites: a flight model (FM) and a break-away engineering model (EM) test fixture.

## Contents

| File | Contents |
|---|---|
| `GERBER-Burnwing-rev3.3.zip` | 9 gerber layers, PTH and NPTH Excellon drill files with maps, and the `.gbrjob` |
| `BOM-Burnwing-rev3.3.csv` | 29 lines, 48 placements, every line carrying an LCSC code |
| `CPL-Burnwing-rev3.3.csv` | 48 placements. Includes through-hole parts — see note below |
| `Burnwing-Schematic-rev3.3.pdf` | Schematic |
| `Burnwing-rev3.3.step` | 3D assembly, DNP parts excluded |

## Board

2 layers, 1.6 mm, ENIG, black soldermask. Supply 5.0 to 15.0 V with 12 V as the
flight baseline, 18.0 V absolute maximum. All 66 vias are untented.

## Before ordering assembly

**The CPL includes through-hole footprints deliberately.** J2, the USB-C
receptacle, is a hybrid part declared `through_hole`. An SMD-only position
export drops it and lists 47 parts instead of 48.

**Two parts must be consigned or hand-fitted.** Neither is orderable through
the assembly house, and both should be removed from the uploaded assembly BOM:

| Ref | Part | Reason |
|---|---|---|
| U1 | HUSB238_002DD (C7471904) | Zero supplier stock in the DFN-10 package. Stock held by CubeSource |
| R12, R13 | RR01J36RTB | Burn elements, not an LCSC part. Marked do-not-populate; fitted by CubeSource |

**U2 stock is thin.** The BTS7030-2EPA (C534836) is the flight high-side driver,
has no second source in the design, and stood at 433 units. Worth confirming
before committing to a run.

## Known items

- The Q1 footprint on the board declares the SMD footprint type while the
  `POWERDI3333` library copy does not, so DRC reports one
  `lib_footprint_mismatch`. The board is the correct side and Q1 appears in the
  position file; the warning clears when `(attr smd)` is added to the library
  footprint.
- `R14` and `R19` are 0 Ω jumpers on the engineering half. The schematic and
  board specify `C100044`; the JLCPCB plugin database holds `C141796` for the
  same two parts. Both are 0 Ω 0603, and the flight-half jumpers are unaffected.
- `R33` is a do-not-populate 0 Ω position whose land pattern is 0402 while the
  assigned part is an 0603 device. Not fitted, so no build is affected.

## Verification at `caff272`

ERC 0 violations. DRC 0 violations beyond the Q1 library note above,
0 unconnected pads, 0 schematic-parity issues. Schematic, board, and the JLCPCB
plugin database agree on every part number, so a BOM taken from any of the three
orders the same parts.

Eight checks carry reduced severity in the project's rule set and are therefore
not represented in those totals: hole-to-copper clearance, solder-mask bridging,
courtyard overlap, missing courtyard, silkscreen over copper, silkscreen to
board edge, non-plated hole inside courtyard, and footprint filter match.
