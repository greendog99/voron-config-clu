This is where we manage my 3D printer configuration (Klipper + Moonraker + Mainsail).
The printer is named **CLU**.

## Hardware

- Printer: Voron 2.4R2 300mm (Formbot kit). CoreXY, sensorless X/Y homing.
- Mainboard: BTT Octopus 1.1. It currently runs as a USB-CAN bridge (`[mcu] canbus_uuid` in `clu.cfg`), and a reflash to plain USB is planned.
- Toolhead: Jabberwocky on an **LDO Nitehawk-36** toolboard over USB (`[mcu nhk]`). Extruder, hotend, fans, toolhead LEDs and ADXL are in `hardware/nitehawk-36.cfg`.
- Probe: **Cartographer V3** over USB (`[mcu cartographer]`). Uses the `cartographer3d-plugin` Python package in `~/klippy-env`, not the old cartographer-klipper repo. Z homing uses `probe:z_virtual_endstop`, and the final Z is set with `CARTOGRAPHER_TOUCH_HOME` in `SETUP_PRINTER`.
- Filament changer: **BoxTurtle** with AFC-Lite board (`[mcu Turtle_1]`), 4 lanes. AFC software is in `~/AFC-Klipper-Add-On` (symlinked into `~/klipper/klippy/extras/AFC*.py`). Toolhead filament sensors are `nhk:gpio3` (tool_start) and `nhk:gpio13` (tool_end).
- Nozzle brush: mounted on the **gantry**, so it moves in Z with the nozzle. `AFC_BRUSH` works at any Z, which is why `brush_loc` Z is `-1`.
- Pi: Raspberry Pi with a BTT PiTFT50 running KlipperScreen. Webcams run on crowsnest.

## File layout

- `printer.cfg`: only includes. Load order is mainsail.cfg → mainsail_overrides.cfg → clu.cfg → `hardware/*` → `macros/*.cfg` → `AFC/*.cfg`. Later sections with the same name override earlier ones key by key.
- `clu.cfg`: `[mcu]`, `[printer]` limits, idle_timeout and misc modules.
- `hardware/`: physical config (steppers, heaters, fans, probe, QGL, LEDs, toolboard).
- `macros/`: gcode macros. `printing.cfg` has `PRINT_START` → `SETUP_PRINTER` and `PRINT_END`. `homing_override.cfg` has the sensorless homing.
- `AFC/`: BoxTurtle config. `AFC_Macro_Vars.cfg` holds the positions for the cut/brush/poop/kick/park macros. `AFC/macros/*.cfg` are upstream AFC macros: don't edit them, they get replaced on AFC updates.
- `AFC/AFC.var.*`: AFC runtime state written by the software. Don't hand-edit.
- `*.cfg-disabled`: old configs that aren't loaded (older ones are in git history).
- `printer-YYYYMMDD_HHMMSS.cfg`: automatic SAVE_CONFIG backups (gitignored).
- `TODO.md`: the running list of fixes and improvements. Keep it updated as items get done.

## Don't touch

- Anything below `#*# <---------------------- SAVE_CONFIG ---------------------->` in `printer.cfg`. It holds PID values and the Cartographer models. Change it only through Klipper (`SAVE_CONFIG`).
- `mainsail.cfg` and `AFC_menu.conf` are symlinks into other repos. Override settings in our own files instead.

## Slicer interface

Slicing is done in **OrcaSlicer**, which uploads and starts jobs on the printer through Moonraker. Start/end gcode and per-filament settings live in Orca profiles, not in this repo, so ask before assuming what the slicer sends.
The slicer calls `PRINT_START BED=<temp> EXTRUDER=<temp> TOOL=<initial tool>`, and ends with `PRINT_END`. `SETUP_PRINTER` always runs an adaptive mesh (`MESH=` is ignored if sent). Adaptive meshing needs OrcaSlicer's "Label objects" setting on; without it the mesh covers the full bed.
Tool changes are `T0`–`T3`, handled by AFC.

## Workflow rules

- **Ask before any Klipper restart** (`RESTART` / `FIRMWARE_RESTART` / service restarts). A restart kills a running print. Before suggesting one, check the printer is idle:
  `curl -s localhost:7125/printer/objects/query?print_stats | jq .result.status.print_stats.state`
- Once approved, restart via Moonraker: `curl -s -X POST localhost:7125/printer/firmware_restart` (or `/printer/restart`). Then confirm the printer is ready:
  `curl -s localhost:7125/printer/info | jq .result.state`
  If it isn't, read the tail of `~/printer_data/logs/klippy.log`.
- **Git: one small commit per fix on `main`**, made after the user confirms the change works on the printer.
- Run gcode for testing through `curl -s -X POST localhost:7125/printer/gcode/script --data-urlencode 'script=...'`, then read the result with `/printer/objects/query?...`.
- **Test moves are allowed without asking when the printer is idle** (`print_stats.state` is `standby`, `complete` or `cancelled`), e.g. homing, QGL, jogging, and running brush/park macros. Check the state immediately before each test run. Still ask before heating, extruding, or loading/unloading filament.
- Before editing a macro, grep for its callers across `macros/`, `AFC/` and `~/mainsail-config/mainsail.cfg`. PAUSE, RESUME and CANCEL_PRINT are defined in mainsail.cfg, overridden in `macros/M600_pause_resume_cancel.cfg`, and AFC wraps them at runtime (`_AFC_RENAMED_RESUME_`).

## Useful paths

- Logs: `~/printer_data/logs/` (klippy.log, moonraker.log, AFC.log; rotated copies have date suffixes).
- Klipper source: `~/klipper` (v0.13). Extras like led_effect and voron-klipper-extensions are symlinked into `klippy/extras`, so `git status` shows the checkout as "dirty". That's expected.
- KAMP: `~/Klipper-Adaptive-Meshing-Purging` (only Line_Purge is used).
- Klipper config reference: https://www.klipper3d.org/Config_Reference.html
