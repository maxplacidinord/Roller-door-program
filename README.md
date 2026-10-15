# Roller door PLC program

A demonstration roller-door control program written in the **NORDCON Structured Text dialect** for execution on a compatible NORD drive PLC.

The program controls the automatic opening and closing of a roller door and includes position control, homing, torque-based jam detection, automatic jam recovery, position restoration after power loss, configurable operating parameters, and a PAR-5 parameter-box interface.

> [!IMPORTANT]
> This project is intended for demonstration, development, and training purposes. It is not a safety-rated door controller and must not be used as the sole means of protecting people or equipment.

## Drive compatibility

The reference program runs on an **SK 5xxP** drive. `Roller Door - 200E.nstx` is the port to
the **SK 200E (NORDAC FLEX)**: the control logic is unchanged, but the marker-1 reference
pulse used by autohoming, the `MC_Home` mode value and the digital-input map follow the SK 2xxE.
The port runs on a **default SK 2xxE** (`DIN1`…`DIN4`). A standard device has no `DIN5`/`DIN6`,
and `DIN2`/`DIN3` belong to the HTL encoder, so only `DIN1` (open command) and `DIN4` (limit
sensor) carry signals — the close command comes from the PAR-5 only (see the input table
below). See `AGENTS.md` for the exact differences.

Both files are written for the drive's **8128-byte PLC program budget** (BU 0550 §4.1.1). The
parameter-row display and the preset loader were reduced to fit that budget — see
`AGENTS.md` § *PLC memory* for what was cut.

## Project information

| Item | Value |
|---|---|
| Project | Roller Door PLC Program |
| Author | Max Placidi |
| Version | 1.0 |
| Date | 4 August 2026 |
| Language | NORDCON Structured Text |
| Interface | NORD PAR-5 parameter box |

## Features

- Automatic opening, top dwell, and closing sequence
- Absolute position control
- Automatic homing and travel-length measurement
- Position restoration after drive or PLC restart
- Independent running, jam-clearing, and homing frequencies
- Torque-current jam detection in both directions
- Direction-dependent jam recovery
- Maximum jam-attempt monitoring
- Top and bottom travel-limit monitoring
- Parameter storage through `P355` and `P356`
- Four-page PAR-5 parameter editor
- Accelerated value adjustment when a PAR-5 key is held
- Fault reset from the PAR-5
- Manual open and close commands from digital inputs or the PAR-5

## Operating sequence

During normal operation, the program runs through the following sequence:

1. The door waits in the opening state.
2. An open command moves the door towards `EndPos`.
3. When the upper position is reached, the drive is disabled and the `RestTime` timer begins.
4. After the timer expires, the door closes towards `StartPos`.
5. A new open command during closing immediately reverses the requested direction.
6. When the lower position is reached, the program prepares for the next opening cycle.

The movement command is issued through `FB_MoveAbs`, while drive power is controlled through `FB_Power`.

## State machine

| State | Name | Description |
|---:|---|---|
| `0` | Homing | Measures the available travel and establishes the position reference. |
| `1` | Opening | Moves the door towards `EndPos` at `RunFreq`. |
| `2` | Reached top | Holds the door open for `RestTime` before closing. |
| `3` | Closing | Moves the door towards `StartPos` at `RunFreq`. |
| `4` | Clear closing jam | Moves upwards at `ClearFreq` after a jam is detected while closing. |
| `5` | Clear opening jam | Removes power for `JamTime`, then retries the opening movement. |
| `6` | Clear recovery jam | Handles a further jam while the door is moving upwards during a jam-clear sequence. |

## Digital inputs and controls

The program uses the following input assignments:

| Signal | SK 5xxP | SK 200E (port) | Bit | Function |
|---|---|---|---|---|
| Open command | `DI1` | `DIN1` | `0` | Moves the door towards `EndPos` |
| Close command | `DI2` | *(none — PAR-5 only)* | — | Forced close command; also used as a manual reference command during homing |
| Limit sensor | `DI5` / `DI6` | `DIN4` | `3` | Travel limit sensor; shuts the drive down on overrun |

A default SK 200E only has `DIN1`…`DIN4`, and `DIN2`/`DIN3` belong to the HTL encoder, so
just `DIN1` and `DIN4` are free. The port therefore uses `DIN1` for the open command and
`DIN4` for one limit sensor that stops the drive (E023 overrun flag) when it is triggered
while the door is running. That sensor also provides the homing reference in both
directions, because the sensor is only read during homing and the direction of travel
identifies which end has been reached: a rising edge while homing upwards is the upper
reference, while homing downwards it is the lower one.

There is no spare input left for the close command, so it is issued from the PAR-5 down key
(as in the original) or by the automatic top-dwell timer.

PAR-5 commands provide equivalent manual control:

| PAR-5 control | Function |
|---|---|
| Up | Open the door or increase the selected parameter |
| Down | Close the door or decrease the selected parameter |
| OK | Select or save a parameter; used with the ON key to start homing |
| Left / Right | Change between the operating display and parameter pages |
| Fault-reset key | Clear PLC error flags and reset the motion sequence |
| ON + OK | Initiate homing while the drive is stopped |

> [!NOTE]
> The source header also refers to `DI5` as a test jam signal, while the executable code used `DI5` as the upper travel sensor. In the SK 200E port a single limit sensor on `DIN4` is read for both directions, and homing uses the direction of travel to tell them apart.

## Parameter configuration

### Automatic parameter overwrites

The program writes the following parameters during initialisation (RAM only, once per PLC
start):

| Parameter | Value | Purpose |
|---|---:|---|
| `P420[-01..-04]` | `0` | Switch off the drive's own digital-input functions (SK 200E port) |
| `P480[-09]` | `23` | Marker 1 response: reference point (reached) |
| `P481[-09]` | `40` | Marker 1 trigger: output via PLC (SK 5xxP) |
| `P481[-07]` | `40` | Marker 1 trigger: output via PLC (SK 200E port) |
| `P600` | `2` | Position control on (linear ramp, setpoint frequency) |
| `P553[-01..-03]` | `1`, `21`, `22` | Motion-control setpoint routing |
| `P351` | `0` | Control word and main setpoint from the PLC (SK 200E port) |

The Structured Text parameter function blocks use zero-based array indexes, so `[-09]` is
passed to the function block as index `8`, `[-07]` as index `6` and `[-01]` as index `0`.
On the SK 500P these parameters set up the marker-1 reference-point pulse used by homing:
the PLC raises marker 1 through `_39_PLC_set_digital_output_bus.8`, and `P480[-09] = 23`
makes the drive treat that as "reference point reached". On the SK 200E the same pulse is
raised through `_0_Set_digital_output.7` (marker 1 = BusIO Out Bit 6) and the trigger
parameter moves to `P481[-07]`.

The SK 200E port also switches the drive's own input functions off (`P420[-01..-04] = 0`).
`DIN2`/`DIN3` are shared with the HTL encoder, and on the SK 2xxE encoder evaluation is
always active, so those inputs must not carry digital functions (BU 0210 §3.4) — otherwise
every encoder pulse would also trigger "Enable left" and "Fixed frequency 1". `DIN1` and
`DIN4` are read by the PLC only, so their factory functions ("Enable right", "Fixed
frequency 2") are switched off as well.

### Settings the program cannot make

These are device settings, not program values — check them before the first run:

| Parameter | Value | Note |
|---|---|---|
| `P350` | `1` | PLC functionality on. With `P350 = 0` the internal PLC does not run, so the program cannot write it itself. |
| `P301` | *encoder pulses* | `Incremental encoder`: pulses per revolution of the HTL encoder (factory setting = `6`, i.e. `4096` pulses). |
| `P604` | `0` | `Encoder type` = incremental (factory setting). |
| encoder track 0 | *not wired* | Track 0 would occupy `DIN1`, which is the open-command input (BU 0210 §3.4). |

Because the program writes its parameters to RAM, a drive restart with the PLC stopped
reverts to the device's stored values; use NORDCON or the ParameterBox to save them
permanently if the drive is also to run without the PLC program.

### Adjustable parameters: P355

`P355` contains 16-bit operating values.

| Parameter | Variable | Test value | Unit / scaling |
|---|---|---:|---|
| `P355[01]` | `RunFreq` | `2000` | NORD setpoint fraction out of `16384` |
| `P355[02]` | `ClearFreq` | `3500` | NORD setpoint fraction out of `16384` |
| `P355[03]` | `TorqueLimitUp` | `2` | Deciamps |
| `P355[04]` | `TorqueLimitDown` | `2` | Deciamps |
| `P355[05]` | `MaxJams` | `5` | Number of attempts |
| `P355[06]` | `HomeFreq` | `2000` | NORD setpoint fraction out of `16384` |
| `P355[07]` | `FloorDeadZone` | `100` | Millirevolutions |
| `P355[08]` | `TopDeadZone` | `100` | Millirevolutions |
| `P355[09]` | `HomingTorqueLimit` | `2` | Deciamps |

### Persistent values: P356

`P356` contains 32-bit position and timing values.

| Parameter | Variable | Test value | Unit / purpose |
|---|---|---:|---|
| `P356[01]` | Saved position | `100` | Last saved position in millirevolutions |
| `P356[02]` | `EndPos` | `10000` | Measured upper position in millirevolutions |
| `P356[03]` | `RestTime` | `2000` | Upper dwell time in milliseconds |
| `P356[04]` | `JamTime` | `2000` | Power-off recovery time in milliseconds |

## PAR-5 interface

The PAR-5 interface contains one operating display and four parameter pages.

### Operating display

The main page displays the current operating information, including:

- Current door position
- Current state-machine state
- Open and close control information

### Parameter page 1

- Running frequency
- Jam-clear frequency
- Upward torque-current limit
- Downward torque-current limit

### Parameter page 2

- Maximum jam attempts
- Homing frequency
- Floor dead zone
- Top dead zone

### Parameter page 3

- Homing torque-current limit
- End position
- Top dwell time
- Jam recovery time

### Parameter page 4

- Preset selector (`LoadConf`): selecting a preset number writes that preset's parameter set
  (the values shared by all presets, plus any per-door differences) and saves it

### Editing parameters

1. Use the left or right key to enter the parameter pages.
2. Use the up and down keys to select a row.
3. Press OK to enter editing mode.
4. Use up or down to change the value.
5. Hold a key to increase the adjustment rate.
6. Press OK again to save the value.

The selected parameter is marked with `>`. While editing, it is marked with `*`, and its label flashes.

Values on pages 1 and 2 are written to `P355`. Values on page 3 are written to `P355` or `P356`, depending on the selected row. Page 4 is the preset selector: pressing OK on it writes the selected preset's values to `P355`/`P356` and saves them.

## Homing

Homing is initiated from the PAR-5 by pressing the ON and OK keys while:

- The controller is in state `1`
- The motor is stationary
- The operating page is active

The homing sequence performs the following operations:

1. Enables drive power.
2. Moves in the positive direction.
3. Detects the upper endpoint from the limit sensor, a positive torque-current threshold or a manual PAR-5 reference command.
4. Records the current position as the temporary upper endpoint.
5. Moves in the negative direction.
6. Detects the lower endpoint from the limit sensor, a negative torque-current threshold, or a manual PAR-5 reference command.
7. Calculates the usable travel from the difference between the two measured positions.
8. Sets `StartPos` to zero.
9. Writes the measured `EndPos` to `P356[02]`.
10. Completes the position-reference operation through `FB_Home`.

The calculated travel is prevented from being smaller than the combined floor and top dead zones.

## Position recovery

The current position is periodically written to `P356[01]`.

During initialisation, the saved position is restored with `ResetPos` when it lies within the valid travel range:

```text
FloorDeadZone < SavedPosition < EndPos - TopDeadZone
