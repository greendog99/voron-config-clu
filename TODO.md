# CLU config TODO

Found in a config and log review on 2026-11-13. Work through it roughly top to bottom.
Check off an item once the fix is committed and verified on the printer.

## 🔴 Bugs: things that break or could crash

- [x] **`G28 X` / `G28 Y` leaves the printer in relative mode (G91).** `_HOME_X` and `_HOME_Y` switch to G91 and never switch back. Only the Z branch sets G90 again (`macros/homing_override.cfg:37-75`). Wrap the homing_override body in `SAVE_GCODE_STATE` / `RESTORE_GCODE_STATE`.
- [x] **Homing one axis marks every axis as homed.** `SET_KINEMATIC_POSITION` defaults to `SET_HOMED=xyz`, so after `G28 X` the toolhead reports `xyz` and `_CG28` skips homing. When Z is already homed, it also overwrites the real Z position with 1 (`macros/homing_override.cfg`).
- [x] **RESUME after M600 calls `CLEAN_NOZZLE`, which doesn't exist** (`macros/M600_pause_resume_cancel.cfg:69`). That macro is in `clean_nozzle.cfg-disabled`. Replace the call with `AFC_BRUSH`.
- [x] **`SHUTDOWN` calls `BACKUP_CONFIG`, which doesn't exist** (`macros/misc_controls.cfg:10`). It shows up 6× in klippy.log as an unknown command. Either define it with `[gcode_shell_command]`, since that extension is installed, or remove the call.
- [x] **M701/M702 call `LOAD_FILAMENT` / `UNLOAD_FILAMENT`, which don't exist** (`macros/marlin_compatibility.cfg`). Map them to AFC commands (e.g. `TOOL_UNLOAD`) or delete them.
- [x] **The `QUAD_GANTRY_LEVEL` wrapper truncates horizontal_move_z to an integer** (`macros/homing_leveling.cfg:5`). `FAST_QGL`/`COND_QGL` pass `2.5`, which becomes `2`. The wrapper's defaults (10 / 3 / 0.01) also silently override `quad_gantry_level.cfg` (8 / 3 / 0.009). Use `|float` and default to the configured values, or pass through only the params that were given.
- [x] **PAUSE retracts without checking temperature** (`M600_pause_resume_cancel.cfg`). Pausing with a cold nozzle raises "Extrude below minimum temp" and the pause aborts partway. The Z hop also doesn't clamp to `position_max`.
- [x] **A stale bed mesh gets reused.** `SETUP_PRINTER` does a full mesh only when no mesh is loaded, and an adaptive mesh only when `MESH=YES`. Otherwise it keeps the previous print's mesh, which may be an adaptive mesh of a different area. Cartographer scanning is fast, so always run `BED_MESH_CALIBRATE ADAPTIVE=1` (or use `MESH_CONDITIONALLY`, which is defined but never called).

## 🟠 Hardware / tuning

- [ ] **Reflash the Octopus as a plain USB MCU.** It is currently a USB-to-CAN bridge (`clu.cfg` `canbus_uuid`) with nothing on the physical CAN bus. klippy.log shows `bus_state=passive` and about 10M `tx_error`. After reflashing, switch `[mcu]` to `serial: /dev/serial/by-id/...` and remove the `can0` interface setup (`/etc/network/interfaces.d/can0`, `/etc/systemd/network/10-can.link`, `/etc/systemd/network/25-can.network`).
- [ ] **Input shaper is disabled** (`hardware/input_shaper.cfg` is fully commented out). The old values were from the previous toolhead anyway. Run `SHAPER_CALIBRATE` with the Nitehawk ADXL, save the results, then raise `max_accel` above 3200 (`clu.cfg`).
- [ ] **Check the hotend thermistor type.** `[extruder] sensor_type: Generic 3950` (`hardware/nitehawk-36.cfg`) is unusual for a hotend. Confirm the hotend model and set the right type (e.g. `ATC Semitec 104NT-4-R025H42G` or `104GT-2`), then re-run `PID_CALIBRATE HEATER=extruder TARGET=245`.
- [ ] **Re-check the Jabberwocky extruder settings**: `rotation_distance: 35.2` combined with `gear_ratio: 44:8, 25:17`. Confirm against the Jabberwocky docs and run an e-steps calibration (100mm extrude).
- [ ] **Re-tune pressure advance** for the new toolhead (`pressure_advance: 0.07` is probably from the old one).
- [ ] **Part fan tach check doesn't run.** `PREFLIGHT_CHECK`/`PCF_CHECK` (`macros/fan_tacho.cfg`) read `printer.fan.rpm`, but `[fan]` has no `tachometer_pin`, so the check is silently skipped. It also isn't called from `PRINT_START`. Wire up and configure the tach pin, or delete the file.
- [ ] **AFC `afc_bowden_length: 1600`** (`AFC/AFC_Turtle_1.cfg`) seems long. Confirm with `AFC_CALIBRATION` and measure the actual tube.
- [ ] Add PETG and other materials you use to `default_material_temps` (`AFC/AFC.cfg`).
- [ ] **The case LED pin is `PB6`** (`hardware/leds.cfg`), which is the BLTouch servo header (`BLT_CTL`), not the NeoPixel header (`PB0`). Fine if that's how it's wired; add a comment saying so.

## 🟡 Cleanup / consistency

- [ ] **Three layers of PAUSE/RESUME/CANCEL_PRINT.** mainsail.cfg defines them, `M600_pause_resume_cancel.cfg` overrides them, and AFC then renames RESUME/PAUSE to wrap them. Because of this, `mainsail_overrides.cfg` (`_CLIENT_VARIABLE`) mostly does nothing. Pick one approach: either use Mainsail's macros plus `_CLIENT_VARIABLE`, or keep the custom ones and drop the override file. AFC already does its own Z hop (`z_hop: 5`) on top of PAUSE's 10mm.
- [x] **`[respond] default_type: error`** (`clu.cfg:27`) makes every `RESPOND`/`M117` echo show up as an error (`!!`) in the console. Change it to `echo` unless the error styling is intentional.
- [ ] **Stale `[probe] z_offset = 0` in the SAVE_CONFIG block** (left over from Tap). Remove it in a SAVE_CONFIG session, or confirm the Cartographer plugin needs it.
- [x] **LED effects use the wrong indices** (fixed: chain is 1 = nozzle left, 2 = nozzle right, 3 = logo; every toolhead effect re-mapped): `logo_leveling` uses (2,3) and `nozzle_leveling` uses (1), which is swapped (`macros/led_effects.cfg:145,152`). `nozzle_meshing` uses (1) instead of (2,3) (`:222`). Also check whether the Jabberwocky still has an SB-style 3-LED chain (`[neopixel sb_leds]`).
- [ ] PAUSE uses `STATUS_LEVELING` as its LED effect. Add a dedicated "paused" effect (there's a TODO in the file).
- [ ] Remove unused macros: `FILAMENT_CHANGE`, `PARK_CENTER`, `MESH_CONDITIONALLY` (if not adopted above), and most of `stealthburner_leds.cfg` (only `set_nozzle_leds_on` is used).
- [ ] `KAMP_Settings.cfg` only includes `Line_Purge.cfg`, so `mesh_margin`/`fuzz_amount` do nothing. The update_manager entry tracks KAMP's `dev` channel; switch to `main`/stable. Check that `tip_distance: 8` matches what the AFC cutter leaves behind.
- [x] Fix stale comments:
  - "Voron Tap" in `homing_override.cfg:65`
  - "park nozzle at front" (actually rear) in `printing.cfg:116`
  - "2 hours" (actually 1h) in `M600_pause_resume_cancel.cfg:42`
  - "FAN4" (actually FAN3) in `hardware/fans.cfg`
- [x] Delete `hardware-disabled/` (old SB2209/Tap/Stealthburner config; it's in git history) and `clean_nozzle.cfg-disabled`.
- [x] Delete `spyglass.conf`, since crowsnest is the webcam service actually running.
- [x] Prune the old `printer-2026*.cfg` SAVE_CONFIG backups (deleted the 7 from July, kept the two from Nov 12).
- [x] **`AFC/AFC.var.unit` is runtime state tracked in git** and changes after every print. Untrack it (`git rm --cached`) and add `AFC/AFC.var*` to `.gitignore`.
- [x] `[force_move]` is defined in both `clu.cfg` and `AFC/AFC_Hardware.cfg`; `[pause_resume]`, `[display_status]` and `[respond]` are also defined in mainsail.cfg. Drop the duplicates.
- [ ] Re-enable the exhaust fan on FAN4 if one is installed (`hardware/fans.cfg`).
- [x] Remove the commented-out `update_manager` blocks for `pgcode` and the old `cartographer` from `moonraker.conf`.
- [ ] **README.md is stale**: it describes a `printer/` subdirectory that doesn't exist and lists the SB2209 instead of the Nitehawk-36 + Jabberwocky.
- [ ] The Klipper checkout is "dirty" only because of extension files symlinked into `klippy/extras` (normal for led_effect/voron-klipper-extensions). Nothing to fix; noting it so it isn't chased again.
