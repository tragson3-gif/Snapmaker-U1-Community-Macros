# Snapmaker U1 Custom Macro Package

This package contains the four final configuration files developed for the Snapmaker U1:

1. `my_nozzle_clean.cfg`
2. `my_chamber_control.cfg`
3. `flow_calibration_flags.cfg`
4. `my_flow_calibration.cfg`

The four `.cfg` files are direct copies of the supplied final files; only the `.crdownload` extension was changed to `.cfg`.

---

## IMPORTANT — Enable Advanced Mode First

On the Snapmaker U1 touchscreen, open the **Maintenance** menu and **enable Advanced Mode**.

Advanced Mode is required before using the printer's advanced configuration / Klipper-side access used to install and work with these macros.

After Advanced Mode is enabled, open the U1 configuration interface you have been using and place the four `.cfg` files with the printer's other configuration files.

Add an include line for each file to the main configuration file that loads your custom configs:

```ini
[include my_nozzle_clean.cfg]
[include my_chamber_control.cfg]
[include flow_calibration_flags.cfg]
[include my_flow_calibration.cfg]
```

Save the configuration and perform a firmware/Klipper restart so the new macros are registered.

Do not replace Snapmaker's stock configuration files with these files. They are intended to be separate included configuration files.

---

## IMPORTANT — Expected "Top Hat Disconnected" UI Warning

When the custom chamber-temperature automation comes online, the U1 touchscreen may report that the **Top Hat disconnected**.

On the tested setup, this is an expected UI side effect of the custom chamber-control macros coming online; it does not by itself mean that the Top Hat physically disconnected. If chamber temperature continues to report and the custom chamber/exhaust controls operate normally, dismiss the warning.

This package deliberately uses `SPEED=0.001` when a Top Cover fan is intended to be stopped. Do **not** casually change those commands to `SPEED=0`; the final chamber-control file was written this way specifically to avoid Top Hat control problems observed during testing.

---

## OrcaSlicer Integration — Chamber Temperature

The chamber controller can be started from **Machine Start G-code** and then given a filament-specific target from each filament profile.

### Filament profile

In OrcaSlicer, open the filament profile and place the chamber-target command in that filament's **Filament start G-code**.

Example for a filament that should run at 42 C:

```gcode
SET_CHAMBER_TARGET TARGET=42 RANGE=10
```

Use the chamber target appropriate for that filament profile. This lets each filament profile carry its own desired chamber temperature instead of requiring you to edit the printer profile every time material changes.

`SET_CHAMBER_TARGET` changes the target while preserving the adaptive controller's learned baseline.

### Machine Start G-code

Start/cancel the chamber automation before `PRINT_START`, but keep the flagged flow-calibration call **immediately after `PRINT_START`**:

```gcode
CANCEL_POST_PRINT_CLEANUP
START_CHAMBER_COOLING TARGET=40 RANGE=10

PRINT_START

CALIBRATE_FLAGGED_TOOLS

DEFECT_DETECTION_START
SET_PRINT_STATS_INFO TOTAL_LAYER={total_layer_count} CURRENT_LAYER=0
TIMELAPSE_START
```

The `TARGET=40` in `START_CHAMBER_COOLING` is the initial/default target. A filament profile can subsequently change it with `SET_CHAMBER_TARGET`.

Do **not** add `TEMP0`, `TEMP1`, `TEMP2`, or `TEMP3` parameters to `CALIBRATE_FLAGGED_TOOLS`. The final flag macro uses Snapmaker's native:

```gcode
FLOW_CALIBRATE FORCE=1
```

so the U1 determines the loaded filament, calibration parameters, and calibration temperature itself.

### Machine End G-code

Add the post-print cleanup call to OrcaSlicer's **Machine End G-code** after the normal print-ending operation:

```gcode
START_POST_PRINT_CLEANUP
```

The cleanup sequence performs the programmed recirculation/exhaust cycle. The next print begins with `CANCEL_POST_PRINT_CLEANUP`, so a delayed cleanup stage from the previous print cannot continue into the next job.

---

# 1. my_nozzle_clean.cfg

## Purpose

Provides:

```text
CLEAN_ALL_NOZZLES
```

This performs a factory-style cleaning sequence on all four U1 toolheads without running extruder-offset calibration.

For each tool it uses Snapmaker's factory extruder-offset preparation actions to:

- Heat the nozzle.
- Run the automatic purge/clean/cut/discard routine.
- Present the nozzle in the factory manual-cleaning position.
- Allow a manual-cleaning interval.
- Cool the nozzle.
- Continue to the next tool.

It does **not** call `PROBE_CALIBRATE` or an extruder-offset calibration command.

## Basic use

```gcode
CLEAN_ALL_NOZZLES
```

Default temperature is 270 C for all four tools.

## Individual temperatures

```gcode
CLEAN_ALL_NOZZLES T0=230 T1=230 T2=270 T3=270
```

The final cooling temperature defaults to 65 C and can also be supplied:

```gcode
CLEAN_ALL_NOZZLES T0=230 T1=230 T2=270 T3=270 COOL=65
```

The current file includes a 10-second fallback manual-cleaning wait for each tool.

---

# 2. my_chamber_control.cfg

## Purpose

Adds adaptive control of the U1 Top Cover exhaust fan using the cavity/chamber temperature sensor.

Primary commands:

```text
START_CHAMBER_COOLING
SET_CHAMBER_TARGET
STOP_CHAMBER_COOLING
START_POST_PRINT_CLEANUP
CANCEL_POST_PRINT_CLEANUP
```

The adaptive controller learns the approximate exhaust-fan baseline needed for the current print rather than using a fixed exhaust speed.

The file also contains a post-print air-cleaning cycle:

1. Internal recirculation at 100% for 10 minutes.
2. Internal recirculation off and exhaust at 80% for 2 minutes.
3. Both fans stop.

### Important U1 fan behavior

The configuration intentionally uses:

```text
SPEED=0.001
```

to physically stop the Top Cover fans. The file specifically warns not to use `SPEED=0`, because that causes Top Hat problems on this setup.

## Start adaptive chamber control

Example:

```gcode
START_CHAMBER_COOLING TARGET=40 RANGE=10
```

`TARGET` is the desired chamber temperature.

`RANGE` controls the strength of the temperature correction. With `RANGE=10`, a 1 C temperature error changes exhaust demand by approximately 5 percentage points.

## Change target while running

```gcode
SET_CHAMBER_TARGET TARGET=42 RANGE=10
```

This changes the target without resetting the learned baseline.

## Stop chamber control

```gcode
STOP_CHAMBER_COOLING
```

## Recommended print-start integration

See the **OrcaSlicer Integration — Chamber Temperature** section above for the complete Machine Start, Filament Start, and Machine End setup.

A new print should cancel any unfinished post-print cleanup before starting chamber control:

```gcode
CANCEL_POST_PRINT_CLEANUP
START_CHAMBER_COOLING TARGET=40 RANGE=10

PRINT_START
```

Keep the remainder of your normal Snapmaker/Orca start sequence after this.

## Recommended print-end integration

Call:

```gcode
START_POST_PRINT_CLEANUP
```

from the print-end sequence after the normal printing operation has finished.

The next print's `CANCEL_POST_PRINT_CLEANUP` prevents an old delayed cleanup stage from continuing into the new print.

---

# 3. flow_calibration_flags.cfg

## Purpose

This is the automatic per-tool flow-calibration request system.

Instead of relying on Snapmaker/Orca's print-specific flow-calibration check box, each physical U1 tool has its own independent flag.

When a filament roll changes, flag only that channel. The next print will force native Snapmaker flow calibration for that flagged tool, then clear its flag.

This is particularly useful with the print queue because calibration is associated with the filament/tool that changed rather than being requested again for every queued print.

## Flag an individual tool

```gcode
FLOW_CAL_T0
FLOW_CAL_T1
FLOW_CAL_T2
FLOW_CAL_T3
```

Example: if only the rolls in T0 and T3 changed:

```gcode
FLOW_CAL_T0
FLOW_CAL_T3
```

T1 and T2 remain untouched.

## Flag all tools

```gcode
FLOW_CAL_ALL
```

## Check pending flags

```gcode
FLOW_CAL_STATUS
```

Example response:

```text
FLOW: Pending calibration: T0=1 T1=0 T2=0 T3=1
```

## Clear all flags manually

```gcode
FLOW_CAL_CLEAR_ALL
```

## Print-start integration

The key command is:

```gcode
CALIBRATE_FLAGGED_TOOLS
```

Place it **immediately after `PRINT_START`** in OrcaSlicer's Machine Start G-code:

```gcode
PRINT_START

CALIBRATE_FLAGGED_TOOLS

DEFECT_DETECTION_START
SET_PRINT_STATS_INFO TOTAL_LAYER={total_layer_count} CURRENT_LAYER=0
TIMELAPSE_START
```

If chamber control is also installed, a combined start sequence can begin like this:

```gcode
CANCEL_POST_PRINT_CLEANUP
START_CHAMBER_COOLING TARGET=40 RANGE=10

PRINT_START

CALIBRATE_FLAGGED_TOOLS

DEFECT_DETECTION_START
SET_PRINT_STATS_INFO TOTAL_LAYER={total_layer_count} CURRENT_LAYER=0
TIMELAPSE_START
```

Do **not** pass `TEMP0`, `TEMP1`, `TEMP2`, or `TEMP3` to `CALIBRATE_FLAGGED_TOOLS`.

The final macro uses:

```gcode
FLOW_CALIBRATE FORCE=1
```

Snapmaker's native flow-calibration routine therefore determines the loaded filament, calibration parameters, and calibration temperature.

The flag is cleared after the corresponding `FLOW_CALIBRATE FORCE=1` command completes.

### Why these flags are independent

These custom flags intentionally do not modify Snapmaker's own `flow_calibrate` / `flow_calib_extruders` print-preference data. This keeps the custom workflow separate from Snapmaker's UI preference mechanism and avoids depending on that internal representation.

---

# 4. my_flow_calibration.cfg

## Purpose

Provides direct/manual flow-calibration macros for individual tools and all four tools.

The file explicitly selects the physical tool before calling Snapmaker's native `FLOW_CALIBRATE`.

Tool mapping documented by the file:

```text
T0 -> extruder
T1 -> extruder1
T2 -> extruder2
T3 -> extruder3
```

## Calibrate one tool manually

```gcode
CALIBRATE_FLOW_T0
CALIBRATE_FLOW_T1
CALIBRATE_FLOW_T2
CALIBRATE_FLOW_T3
```

Each defaults to 220 C.

A temperature can be supplied manually:

```gcode
CALIBRATE_FLOW_T0 TEMP=240
CALIBRATE_FLOW_T3 TEMP=255
```

## Calibrate all four manually

```gcode
CALIBRATE_ALL_FLOW
```

Default temperature: 220 C.

Or:

```gcode
CALIBRATE_ALL_FLOW TEMP=240
```

This manual file is separate from the automatic flag system. The automatic system in `flow_calibration_flags.cfg` uses `FLOW_CALIBRATE FORCE=1` and does not require manually supplied temperatures.

---

# Suggested Complete Installation Order

1. On the U1 touchscreen, open **Maintenance** and enable **Advanced Mode**.
2. Open the U1 advanced configuration interface.
3. Back up the existing printer configuration before changing it.
4. Add these four `.cfg` files alongside the printer's configuration files.
5. Add the four `[include ...]` lines to the configuration that loads custom files.
6. Save and restart the firmware/Klipper configuration.
7. Confirm the new macros appear in the macro list or console.
8. Add `CALIBRATE_FLAGGED_TOOLS` immediately after `PRINT_START` in Orca Machine Start G-code.
9. If using adaptive chamber control, add `CANCEL_POST_PRINT_CLEANUP` and `START_CHAMBER_COOLING ...` to print start and `START_POST_PRINT_CLEANUP` to print end.
10. Test with a single calibration flag first:

```gcode
FLOW_CAL_CLEAR_ALL
FLOW_CAL_T0
FLOW_CAL_STATUS
```

11. Send a print and confirm T0 performs native flow calibration and its flag clears.
12. After verification, use `FLOW_CAL_T0` through `FLOW_CAL_T3` whenever the corresponding filament roll changes.

---

# Files

- `my_nozzle_clean.cfg` — factory-style four-nozzle cleaning without offset calibration.
- `my_chamber_control.cfg` — adaptive chamber exhaust control plus post-print filtration/exhaust.
- `flow_calibration_flags.cfg` — independent per-tool next-print flow-calibration flags.
- `my_flow_calibration.cfg` — direct/manual per-tool and all-tool flow calibration commands.
