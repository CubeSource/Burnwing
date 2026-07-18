# Burnwing rev 4 — Replace HUSB238 with CH32X035 PD-capable MCU

**Date:** 2026-07-18
**Status:** Draft — pending review
**Board:** cubesource/Burnwing (base: rev 3.2)

## Goal

Replace U1 (HUSB238, WDFN-10) with a CH32X035F8U6 MCU (QFN20, LCSC C42442062, ~$0.53). The MCU performs USB-PD sink negotiation in firmware and adds current telemetry, USB serial control, and an I2C slave interface — while leaving every existing hardware burn path untouched. This lays the foundation for a future fully self-contained release device (RTC, storage cap, SCR dump) without implementing those features this revision.

## Architecture decisions (agreed)

1. **Chip-on-board**, not the WeAct core board module. CH32X035F8U6 stays viable in QFN20 — 17 of 19 GPIOs used, two ADC-capable spares.
2. **Parallel burn path.** External `BURN_CTRL_1/2` lines and dev-kit buttons keep their direct hardware route to the BTS7030-2EPA inputs. MCU GPIOs are diode-ORed onto the same nets and can only *add* drive, never veto. A blank/dead MCU leaves a fully working 5V board (USB-C supplies vSafe5V without negotiation). Auto-cutoff therefore applies to MCU-initiated burns only; external paths remain pure hardware by design.
3. **Voltage selection: cycle button + 4 indicator LEDs + solder-jumper hard-set** (CH224-trigger-board UX, replacing the rev-3 DIP switch). Voltages: 5 / 9 / 12 / 15 V. Button steps through them; selection persists in MCU flash; firmware falls back to the highest offered voltage ≤ selection (worst case 5V) and inhibits renegotiation while any burn channel is conducting.
4. **LED pins double as jumper straps.** LEDs are active-low (pin sinks from 3.3V); each solder-jumper pad ties its pin to GND. Read as inputs at boot: closed jumper = forced voltage, and that LED is permanently lit. No extra pins.
5. **I2C slave on the SWD pins (PC18/PC19), shared with debug.** On QFN20 the I2C peripheral only routes to PC16/17 (USB — would kill CDC) or PC18/19 (SWD). Qwiic connector (JST-SH: GND/3V3/SDA/SCL) and SWD test pads share these nets. Firmware holds the pins as SWD for a ~2s boot window before remapping to I2C, so a debugger can always attach at reset; unplug Qwiic while debugging.
6. **Programming:** SWD test pads (WCH-Link) + USB bootloader via the USB-C data lines. Runtime "jump to bootloader" command over CDC.
7. **Command-sense taps kept dedicated** (pins are available): 10k from BURN_CTRL/button nets to two GPIOs so firmware observes externally-commanded burns. Note: IS current sensing already detects any conducting burn regardless of source; the taps distinguish "commanded" from "conducting".

## Hardware changes

### Removed
- U1 HUSB238 (LCSC C7471904) and its CC/CFG resistor network (CH32X035 provides Rd internally; CC1/CC2 route straight to MCU).
- SW1 DIP switch (replaced by cycle button).

### Added
- CH32X035F8U6, QFN-20-EP 3x3 (C42442062) on the 3.3V rail (existing TPS70933 LDO; TPS709 input tolerates the full 5–15V VBUS range).
- Momentary pushbutton (voltage cycle).
- 4× indicator LED + resistor (active-low to 3.3V) with parallel solder-jumper pads to GND.
- Qwiic JST-SH 4-pin connector (GND/3V3/SDA/SCL) + 2× I2C pull-ups to 3.3V, shared nets with SWD pads.
- SWD test pads (SWDIO/SWCLK; net-shared with Qwiic).
- 2× diode-OR (small-signal diode + existing pull-downs) from MCU burn-drive GPIOs into BTS7030 IN0/IN1 nets.
- 2× IS sense resistor + RC filter on BTS7030 IS pins into MCU ADC.
- 2× 10k command-sense taps from BURN_CTRL/button nets to MCU GPIOs.

### Draft pin map (verify against AF/remap tables in schematic phase)

| Function | Pins | Notes |
|---|---|---|
| PD CC1/CC2 | PC14 / PC15 | fixed silicon |
| USB D−/D+ | PC16 / PC17 | CDC + factory bootloader |
| SWDIO/SWCLK ↔ I2C SDA/SCL | PC18 / PC19 | boot window ~2s as SWD, then remap; SDA/SCL assignment per RM |
| Burn drive 1/2 | PB3, PB11 | Hi-Z at reset; pull-downs guarantee off through power-up/reset/flash |
| IS sense 1/2 | PA0, PA1 | ADC A0/A1 |
| Command taps 1/2 | PA2, PA3 | |
| Cycle button | PB12 | |
| Voltage LEDs / straps (5/9/12/15) | PA4–PA7 | active-low, jumper pads to GND |
| Spare | PB0, PB1 | ADC-capable |

## Firmware v1 scope (agreed: "PD + telemetry")

Base: WCH `02-PD_Sink` example (WeActStudio.CH32X035CoreBoard repo) + WCH PD stack.

- Boot → read jumper straps, else flash-stored selection → negotiate; fall back to highest offered ≤ selected; request current from the source's advertised PDO (RDO current field — supersedes HUSB238 CFG-strap current selection). LED shows active voltage; blink patterns for negotiating/failed states.
- USB CDC console: PD status (source PDO list, active contract), live IS current telemetry, burn commands (fire / timed pulse), jump-to-bootloader.
- Auto-cutoff on wire-break (IS current collapse) for MCU-initiated burns; renegotiation inhibited while any channel conducts.
- I2C slave (7-bit address 0x42, firmware-changeable via register write persisted to flash): register map mirroring CDC functionality; burn commands require two-step **arm → fire** with magic byte inside a short arm window.
- Out of scope this rev (future roadmap): RTC scheduling, storage-cap trickle charge, SCR dump path, standalone drop-off logic.

## Unchanged

BTS7030-2EPA switches and direct control paths, dev-kit buttons and their current-limit resistors, Zener clamps, PicoBlade/Pico-Lock connectors, snap-off dev section, TPS70933 LDO, USB-C connectors.

**Prerequisite housekeeping:** rev 3.2 still carries the outstanding D5/D6 footprint reconciliation and the mislabeled Pico-Lock J2→J4 fix (see `burnwing-schematic-reconciliation.md`); land those before or with this rev.

## Validation plan

1. Scope BTS7030 IN pins through MCU power-up, reset, and flashing — prove no glitch can fire a channel.
2. PD negotiation against ≥3 PD chargers plus a dumb 5V supply (must degrade to working 5V board).
3. USB CDC and PD coexistence on the single port; bootloader entry.
4. Debugger attach inside the boot window with Qwiic disconnected; I2C slave function after remap.
5. Live burn of a real wire under telemetry: confirm current profile capture, wire-break detection, auto-cutoff.
6. Jumper-strap boot reads and LED indication for all four voltages.

## Open details for the schematic phase

- Exact I2C remap register value and SDA/SCL orientation on PC18/PC19 (CH32X035 reference manual).
- Bootloader entry mechanism (replicate WeAct core board's BOOT arrangement).
- IS sense-resistor value from the BTS7030-2EPA kILIS ratio scaled to 3.3V ADC range; RC corner.
- Diode-OR part selection and confirmation existing pull-down values hold nets low against diode leakage.
- LED strap circuit: confirm logic thresholds with chosen LED Vf and internal pull-ups.
- JLCPCB availability pass on new parts (button, JST-SH, LEDs); CH32X035F8U6 stock was 1451 at C42442062 on 2026-07-18.
