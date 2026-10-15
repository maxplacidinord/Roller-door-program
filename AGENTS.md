# Roller door PLC program — working notes

A demonstration roller-door control program in the **NORDCON Structured Text (ST)**
dialect for a NORD **SK 5xxP** drive PLC. Author: Max Placidi.
See `README.md` for the functional description and the PAR-5 interface.

## Files (which is which)

| File | Role |
|---|---|
| `Roller Door - 500P - FINAL.nstx` | **Current working final** (header Version 1.5). Edit this one. |
| `Roller Door - All features 107% PLC memory.nstx` | Full-feature **reference snapshot**. Over the PLC memory limit (107%), so it cannot be commissioned, but kept for its feature set and its richer comments. |
| `Roller Door - 500P - v1.3 floor reset fix.nstx` | Previous fixed revision (Version 1.3). |
| `Roller Door - 500P - optimised CASE presets draft.nstx`, `... - further optimised draft.nstx`, `5xxP - memory reduced.nstx` | Optimization/draft snapshots, historical only. |
| `Roller Door - 200E.nstx` | Port to the **SK 200E (NORDAC FLEX)**. See "SK 200E port" below. |
| `Roller Door - 180E.nstx`, `Roller Door - 2xxE.nstx` | Earlier ports; **removed** - they were straight copies of the 500P program, not real ports. Recoverable from git (`3f15c6f`, `79c71f3`). |

Every version is committed to git, so historical states can be recovered with
`git log` / `git show` rather than by copying files around.

## The `.nstx` file format

`.nstx` is a NORDCON **PLC program file for the ST language** (IL uses `*.awlx`/`*.awl`;
the encrypted, editor-locked variants are `*.nsts`/`*.awls`). BU 0550 §4.2.5.
The file is **XML**, saved by NORDCON:

```
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<NordCon>
  <VersionsInfo System="win" Version="3.0.3"/>
  <PLCFile>
    <VersionsInfo Version="1.0" Date="dd.MM.yyyy HH:mm:ss"/>
    <Program Language="ST"><![CDATA[ ...the ST source... ]]></Program>
    <Checksum>28-byte hex</Checksum>
  </PLCFile>
</NordCon>
```

Encoding and line endings: **UTF-8 with BOM**, **CRLF**. One long single line for the
XML wrapper; the ST source in the CDATA block uses normal CRLF newlines.

### Layout of the ST source (inside the CDATA)

The editor-managed sections always appear in this order; only `@Program` holds the
hand-written code, the rest is generated/managed by NORDCON:

```
(* @Header *)      HEADER   ... END_HEADER   (see below)
(* @Configuration *)   --- editor only ---
(* @Breakpoints *)      --- editor only ---
(* @Watchpoints *)      --- editor only ---
(* @Variablen declarations *)  declaration + VAR_ACCESS + VAR sections
(* @Displayformat *)    --- auto-generated variable dump ---
(* @Debuglist *)        --- editor only ---
(* @Watchlist *)        --- editor only ---
(* @Program *)          the actual program
```

The `HEADER ... END_HEADER` pseudo block at the top is written by NORDCON:

```
HEADER
  VERSION = 2
  VENDOR  = NORD DRIVESYSTEMS
  DATE    = dd/mm/yyyy HH:mm:ss      <- last save/compile
  DATA    = <compiled program, hex>
  PLCDISPLAY = <hex of the PAR-5 start-up display text>
  STMODE  = ON
  PLCTYPE = 42
END_HEADER
```

- `DATE`/`DATA` and the trailing `<Checksum>` are rewritten by NORDCON on every save.
- `<Checksum>` is 28 bytes of hex (SHA-224-sized). It is **not** a plain hash of the
  source text (tested against SHA-224/SHA-256 of the source, with and without BOM and
  with CRLF/LF normalisation - no match). It tracks the `HEADER` `DATA` field, i.e. the
  compiled image, so it changes when NORDCON re-saves even if the code is unchanged.
- **Consequence:** editing the ST source outside NORDCON leaves `DATE`, `DATA` and
  `<Checksum>` stale. Open and compile/save in NORDCON to regenerate them before
  downloading. Hand-editing here is fine for source work; the file still parses as XML.

## PLC memory

- The PLC budget is program memory + marker memory (variables + function-block
  instances), with a dynamic boundary. "PLC memory %" in the filenames is the
  **resource consumption shown in the NORDCON message window** after compiling.
- SK 5xxP: **8128 bytes** for markers, functions and the PLC program
  (SK 38xP-FDS: 16256 bytes). BU 0550 §4.1.1. (The §4.1.2.1 worked example uses a
  generic 4032-byte total; the per-series table is the one that applies here.)
- Comments, blank lines and indentation are **not compiled**, so they do not consume
  program memory. The 107% snapshot was over the limit because of its logic, not its
  comments. Restoring richer comments therefore does not change the memory figure -
  confirm in NORDCON after compiling.

## SK 200E port (`Roller Door - 200E.nstx`)

Derived from `Roller Door - 500P - FINAL.nstx`. The control logic is the same except for
the platform differences below (the SK 2xxE input shortage also removes the close-command
input). Reference: BU 0550 (PLC) plus BU 0200 / BU 0210 (NORDAC FLEX SK 200E and its
POSICON).

| # | Topic | SK 5xxP (original) | SK 2xxE (port) |
|---|---|---|---|
| 1 | Marker 1 pulse (autohoming) | `_39_PLC_set_digital_output_bus.8` | `_0_Set_digital_output.7` (BusIO Out Bit 6 = marker 1) |
| 1 | Marker 1 parameters | `P480[-09]=23`, `P481[-09]=40` | `P480[-09]=23`, `P481[-07]=40` |
| 2 | `MC_Home` | byte `MODE` (16 = P623 = 16) | standard FB, `MODE := FALSE` (BOOL) |
| 3 | Close command | `_5_State_digital_input`.1 (DIN2) | **removed** - DIN2/DIN3 are the HTL encoder, so close is PAR-5 only |
| 4 | Limit sensor | `_5_State_digital_input`.4/.5 (DIN5/DIN6) | `.3` (DIN4) for **both** - a default SK 2xxE has no DIN5/DIN6 |

- A default SK 2xxE only provides DIN1…DIN4 (bits 0…3), and DIN2/DIN3 belong to the HTL
  encoder (BU 0200 §2.5). Only DIN1 and DIN4 are usable, so the port puts the open command
  on bit 0 (DIN1) and a single limit sensor on bit 3 (DIN4).
- The close command's digital-input term was dropped (the `_5_State_digital_input.1 OR`
  clause at the four state-machine sites); the PAR-5 down key and the automatic top-dwell
  timer still close the door.
- One sensor covers both homing references: the two edge detectors (`DI5Edge`/`DI6Edge`)
  both read bit 3, and they are only evaluated during homing, where the direction of travel
  separates them (rising edge going up = upper reference, going down = lower). Triggered
  during normal running it sets the overrun flag and shuts the drive down
  (`_110_ErrorFlags.1` and/or `.2`, i.e. E023.1 / E023.2).
- `_39_PLC_set_digital_output_bus` does **not** exist on the SK 2xxE (BU 0550 §4.5). On the
  SK 2xxE `_0_Set_digital_output` bits 1..8 map to BusIO Out Bit 0..7, and marker 1 is
  BusIO Out Bit 6 (BU 0200 P481), so the marker is written with bit 7. Autohoming is
  therefore carried over, not dropped. It is declared as `INT`, not `UINT`: the NORDCON
  variable list only offers BOOL/BYTE/INT/DINT/STRING (BU 0550 §4.2.6.1), even though the
  §4.5 process-value table lists the type as UINT.
- `P480[-09]` (marker 1 action) is `23` = reference point, unchanged. Only the P481
  trigger moves array: marker 1 is `[-07]` on the SK 2xxE, `[-09]` on the SK 5xxP.
- The byte-`MODE` `MC_Home` (BU 0550 §4.3.3.4) is SK 5xxP/31xP/35xP/38xP-FDS only. The
  SK 2xxE uses the standard `MC_Home` (§4.3.3.3), whose `MODE` is a BOOL. The marker pulse
  lasts one scan, so the reference position is applied on the following scan - confirm the
  reversal behaviour during commissioning.
- Every other function block used (`MC_Power`, `MC_MoveAbsolute`, `MC_Reset`,
  `MC_WriteParameter_16/32`, `FB_ResetPosition`, `FB_STRINGToPBOX`, `FB_DINTToPBOX`) and
  every other process value used is supported in the 2xxE column of the BU 0550 §4.3
  matrices / §4.5 tables.
- PLC memory: the SK 2xxE(-FDS) has the same **8128 bytes** as the SK 5xxP (BU 0550 §4.1.1),
  so memory is not a constraint for the port.

## Commenting conventions in `@Program`

- Section banners: `(*1. -------------------- Track Limit detection ----------------------*)`
  numbering sections 1-8, matching the header block at the top of `@Program`.
- Comments are NORDCON-green `(* ... *)`; keep explanatory notes on their own line,
  and short tags inline at the end of a code line.
- Keep the descriptive `Line 1..4` blocks for parameter pages 1-4 in the display CASE.

## Reference manuals (in the NORD reference library)

- **BU 0000 - NORDCON** - NORDCON user interface and project handling.
- **BU 0550 - PLC Functionality** - PLC languages, memory model, function blocks, faults.
  Cited pages above are §4.1.1 (PLC specification / memory), §4.1.2.1 (memory),
  §4.2.5 (open/save/encryption, file types).
