# Roller door PLC program

A demonstration roller-door control program written in the **NORDCON Structured Text dialect** for execution on a compatible NORD drive PLC.

The program controls the automatic opening and closing of a roller door and includes position control, homing, torque-based jam detection, automatic jam recovery, position restoration after power loss, configurable operating parameters, and a PAR-5 parameter-box interface.

> [!IMPORTANT]
> This project is intended for demonstration, development, and training purposes. It is not a safety-rated door controller and must not be used as the sole means of protecting people or equipment.

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

| Input | Function |
|---|---|
| `DI1` | Open command |
| `DI2` | Forced close command; also used as a manual reference command during homing |
| `DI5` | Upper travel sensor in the current code |
| `DI6` | Lower travel sensor and lower reference during homing |

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
> The source header also refers to `DI5` as a test jam signal. The executable code uses `DI5` as the upper travel sensor, so the physical input mapping should be verified before testing.

## Parameter configuration

### Automatic parameter overwrites

The program writes the following parameters during initialisation:

| Parameter | Value | Purpose |
|---|---:|---|
| `P480[09]` | `23` | Assigns the reference-point function |
| `P481[09]` | `40` | Assigns the PLC-controlled digital output function |

The Structured Text parameter function blocks use zero-based indexes, so `[09]` is passed to the function block as index `8`.

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

- Last saved position
- Three currently unused display rows

### Editing parameters

1. Use the left or right key to enter the parameter pages.
2. Use the up and down keys to select a row.
3. Press OK to enter editing mode.
4. Use up or down to change the value.
5. Hold a key to increase the adjustment rate.
6. Press OK again to save the value.

The selected parameter is marked with `>`. While editing, it is marked with `*`, and its label flashes.

Values on pages 1 and 2 are written to `P355`. Values on page 3 are written to `P355` or `P356`, depending on the selected row. Page 4 is intended as a position-information page.

## Homing

Homing is initiated from the PAR-5 by pressing the ON and OK keys while:

- The controller is in state `1`
- The motor is stationary
- The operating page is active

The homing sequence performs the following operations:

1. Enables drive power.
2. Moves in the positive direction.
3. Detects the upper endpoint from a positive torque-current threshold or a manual PAR-5 reference command.
4. Records the current position as the temporary upper endpoint.
5. Moves in the negative direction.
6. Detects the lower endpoint using `DI6`, a negative torque-current threshold, or a manual PAR-5 reference command.
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
