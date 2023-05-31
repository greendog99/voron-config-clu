# voron-config-clu
Configuration and information about my Formbot Voron 2.4R2 300✕300 printer, named CLU.

My configuration is broken up into many files:

* `printer.cfg` just includes all the other files from the `printer` subdirectory.
* `printer/printer.cfg` contains the main printer info.
* `printer/hardware/*` contains (mostly) physical printer configurations, such as stepper settings.
* `printer/macros/*` contains (mostly) gcode macros, like `PRINT_START`.
* It's (always) a work in progress.

## Major Modifications

The modifications I've made so far:

[Voron Tap R8](https://github.com/VoronDesign/Voron-Tap)

[SB2209 CANbus toolhead board](https://github.com/bigtreetech/EBB/tree/master) and removed X/Y cable chains and endstops.

GE5C Z bearing mod [mashup by lolplastics](https://www.printables.com/model/457431-voron-24-ge5c-mashup).

## Aesthetic / Quality of Life Mods

BTT PiTFT50 5" touchscreen running KlipperScreen. Mounted using [CannedBass's 45" mount](https://github.com/VoronDesign/VoronUsers/tree/master/printer_mods/CannedBass/PITFT50_45_degree_mount).

[Magnetic removable panels by Le0n](https://github.com/VoronDesign/VoronUsers/tree/master/printer_mods/Le0n/Magnetic_Panels_with_Magnet_Inserts).

[+1mm Z drive belt tensioners by ZvB](https://www.printables.com/model/356388-voron-v24-z-belt-loop-tensioners-1mm) since my short Z motor belts were fairly loose after install.

[Magnetic removable hinged doors by PALM](https://www.printables.com/model/221377-voron-24-magnetic-door-hinges).

[Magnetic door latches by Stephan C](https://www.printables.com/model/419658-magnet-door-latches-for-voron/comments/780939).

[Z axis dials](https://www.printables.com/model/242370-voron-24-z-axis-dials).

