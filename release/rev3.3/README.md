# Burnwing Rev 3.0 Production Files

Fabrication and assembly output for the Burnwing Rev 3.0 board, generated from commit `caff272` and carried by the `rev3.3` tag (`df94088`).

Burnwing is a dual-channel thermal knife release mechanism designed for CubeSats and small satellites. It ships as a unified panel carrying two sections joined by mouse bites: a **Flight Model (FM)** and a break-away **Engineering Model (EM)** test fixture.

---

## Features

- **Dual-Purpose Panel**: The break-away Engineering Model (EM) test fixture allows full benchtop testing and can be separated via mouse bites prior to flight integration.
- **USB-C PD Powered**: Selectable voltage and current limits for straightforward lab testing, or powered via Molex PicoBlade / Pico-Lock connectors.
- **Comprehensive Protection**: Integrated ESD protection, over-temperature protection, over-current limiting, and reverse polarity protection.

## Notable Improvements (Rev 2 → Rev 3)

- **Reversible 6-Pin Cable Pinout**: Enables the use of unmodified COTS Molex wire harnesses without polarity orientation errors.
- **High-Side Control Architecture**: Switched from low-side switching (NCV8402ADDR2G) to an Infineon BTS7030-2EPA high-side smart power switch. This keeps the burn elements unenergized until triggered and supports both 3.3V and 5V logic inputs.

---

## Production Files & Release Assets

| File | Description |
|---|---|
| `GERBER-Burnwing-rev3.3.zip` | 9 Gerber layers, PTH/NPTH Excellon drill files with maps, and the `.gbrjob` |
| `BOM-Burnwing-rev3.3.csv` | SMT Bill of Materials (29 line items, 48 placements, every line carrying an LCSC code) |
| `CPL-Burnwing-rev3.3.csv` | Component Placement List (48 placements, including hybrid through-hole parts) |
| `Burnwing-Schematic-rev3.3.pdf` | Complete circuit schematic |
| `Burnwing-rev3.3.step` | 3D CAD assembly model (DNP components excluded) |

---

## Board Specifications

- **Layers**: 2 layers, 1.6 mm thickness
- **Finish**: ENIG (Electroless Nickel Immersion Gold), black solder mask
- **Supply Voltage**: 5.0 V to 15.0 V nominal (12.0 V flight baseline, 18.0 V absolute maximum)
- **Vias**: All 71 vias are untented for thermal and probing accessibility

### DIP switch settings

The voltage legends (5 V / 12 V) are exact. The current legends read 1 A and 2 A; the values actually requested over USB-PD are **1.25 A and 2.25 A**, because the current-select resistor decodes to the nearest tabulated step. This is a ceiling negotiated with the supply rather than a current the board draws, and the board's own requirement is 1.0 A per channel, so neither setting under-supplies it.

---

## Read Before Ordering Assembly

**The CPL includes through-hole footprints deliberately.** `J2`, the USB-C receptacle, is a hybrid part declared `through_hole`. An SMD-only position export drops it and lists 47 parts instead of the correct 48. Do not re-export the position file as SMD-only.

**Two parts must be consigned or hand-fitted, and must come off the uploaded assembly BOM.** Neither is orderable through the assembly house:

| Ref | Part | Reason |
|---|---|---|
| U1 | HUSB238_002DD (C7471904) | Zero supplier stock in the DFN-10 package. Stock held by CubeSource |
| R12, R13 | RR01J36RTB | Burn elements, not an LCSC part. Marked do-not-populate; fitted by CubeSource |

**U2 stock is thin.** The BTS7030-2EPA (C534836) is the flight high-side driver, has no second source in the design, and stood at 433 units. Worth confirming before committing to a run.

---

## Verification & Status

- **ERC**: 0 violations.
- **DRC**: 0 violations beyond the Q1 library note below, 0 unconnected pads, 0 schematic-parity issues.
- **BOM & CPL Parity**: Schematic, board, and the JLCPCB plugin database agree on every part number, so a BOM taken from any of the three orders the same parts. 48-placement CPL.

Eight checks carry reduced severity in the project's rule set and are therefore not represented in those totals: hole-to-copper clearance, solder-mask bridging, courtyard overlap, missing courtyard, silkscreen over copper, silkscreen to board edge, non-plated hole inside courtyard, and footprint filter match.

### Known items

- The Q1 footprint on the board declares the SMD footprint type while the `POWERDI3333` library copy does not, so DRC reports one `lib_footprint_mismatch`. The board is the correct side and Q1 appears in the position file; the warning clears when `(attr smd)` is added to the library footprint. Fixed on `main` in `85b867c` (2026-08-21); the published rev3.3 files predate that commit and still carry the warning.
- `R14` and `R19` are 0 Ω jumpers on the engineering half. The schematic and board specify `C100044`; the JLCPCB plugin database holds `C141796` for the same two parts. Both are 0 Ω 0603, and the flight-half jumpers are unaffected.
- `R33` is a do-not-populate 0 Ω position whose land pattern is 0402 while the assigned part is an 0603 device. Not fitted, so no build is affected.
