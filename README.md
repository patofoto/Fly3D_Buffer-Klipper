# FLY-LLL PLUS Buffer — Klipper Macros

Klipper configuration and macros for the Mellow/Fly3D **FLY-LLL PLUS** filament buffer:

- pin and filament-sensor setup for the buffer's control signals
- standalone `BUFFER_LOAD_FILAMENT` / `BUFFER_UNLOAD_FILAMENT` macros
- hooks for [Demon Klipper Essentials Unified](https://github.com/3DPrintDemon/Demon_Klipper_Essentials_Unified), so Demon's own `LOAD_FILAMENT` / `UNLOAD_FILAMENT` work with the buffer

Developed and tested on a Voron 2.4 350 (Stealthburner, Galileo 2, Revo Voron) with about 1.35m of PTFE between the buffer and the toolhead.

## Requirements

- FLY-LLL PLUS flashed with firmware **v1.1.x or later**: [patofoto/Buffer](https://github.com/patofoto/Buffer) (fork of [Fly3DTeam/Buffer](https://github.com/Fly3DTeam/Buffer))
- A current Klipper. The sensor config uses `debounce_delay`, which older versions don't support.
- Five free MCU pins wired to the buffer board (see [Wiring](#wiring))
- Optional: Demon Klipper Essentials Unified

## How the buffer behaves

These points drive how the macros are written.

- **Two control signals act like the buffer's buttons.** *Feed* (forward) and *Retract* (reverse) are active-low: `VALUE=0` means pressed and the motor runs; `VALUE=1` means released and the motor is idle.
- **The filament sensor is the buffer's inlet switch.** "Detected" means filament has been inserted into the buffer, not that it has reached the extruder. There is no sensor at the toolhead.
- **The sensor output freezes while Retract is held.** The firmware waits in a loop until the signal is released and doesn't update the sensor meanwhile. So retraction runs in segments, with a 0.25s release between them so the sensor can update.
- **The buffer follows the extruder in one direction only.** It feeds when the extruder pulls filament. It does *not* retract when the extruder pushes filament back into the tube, so a long extruder-only retraction bunches filament up in the tube and leaves the end in the gears. Unloads therefore have the extruder release the filament *while* the buffer pulls.
- **The buffer moves filament at about 23mm/s**, so a full retraction through 1.35m of PTFE takes about a minute.
- **The firmware stops feeding after 60s of continuous feeding by default.** It then stops following the extruder until it's reset. With a long tube the first feed can take about that long, so raise the limit (see [Buffer firmware timeout](#buffer-firmware-timeout)).

## Files

| File | Purpose |
|---|---|
| `mellow_buffer_klipper.cfg` | Pins, filament sensor, manual `Buffer_Feeding` / `Buffer_Retraction` / `BUFFER_STOP` |
| `mellow_buffer_macros.cfg` | Load/unload automation and the macros called from Demon's hooks |
| `mellow_buffer_user_settings.cfg` | Template for your settings. Copy it out of the repo folder. |
| `demon_buffer_integration.cfg` | Reference snippet for Demon's hook file. **Never include it.** |
| `MOTOR_SPEED_REFERENCE.md` | Buffer and extruder motor speed calculations |

## Installation

1. **Clone the repository** into your config folder:
   ```bash
   cd ~/printer_data/config
   git clone https://github.com/patofoto/Fly3D_Buffer-Klipper.git Fly3D_Buffer
   ```

2. **Add it to Moonraker's update manager** in `moonraker.conf`, then restart Moonraker:
   ```ini
   [update_manager Fly3D_Buffer]
   type: git_repo
   path: ~/printer_data/config/Fly3D_Buffer
   origin: https://github.com/patofoto/Fly3D_Buffer-Klipper.git
   primary_branch: main
   is_system_service: False
   managed_services: klipper
   ```

3. **Copy the settings file out of the repo folder.** Moonraker keeps tracked files read-only, and updates would overwrite them.
   ```bash
   cp ~/printer_data/config/Fly3D_Buffer/mellow_buffer_user_settings.cfg ~/printer_data/config/
   ```

4. **Include the files explicitly** in `printer.cfg`:
   ```ini
   [include ./mellow_buffer_user_settings.cfg]            # your editable copy
   [include ./Fly3D_Buffer/mellow_buffer_klipper.cfg]
   [include ./Fly3D_Buffer/mellow_buffer_macros.cfg]
   ```
   Don't use `[include ./Fly3D_Buffer/*.cfg]`. That also loads the repo's own settings file, which then overrides your copy, and the Demon snippet, which then overrides Demon's hooks.

5. **Set your pins** in `mellow_buffer_klipper.cfg` (see [Wiring](#wiring)). If they differ from the defaults, copy that file out of the repo folder too and include your copy, or updates will overwrite your pins.

6. **Restart Klipper.**

### Wiring

Example for the Voron 2.4 this was built on:

| Signal | Printer pin | Buffer board pin |
|---|---|---|
| `_Feed_Button` (output) | PA8 | PB5 (forward signal) |
| `_Retract_Button` (output) | PC9 | PB6 (reverse signal) |
| `filament_sensor` (input) | PF4 | PB15 (inlet switch output) |
| `Trigger Feeding` button (input) | ^!PF1 | PA2 (short press of the buffer's forward key) |
| `Trigger Retraction` button (input) | ^!PF0 | PA3 (short press of the buffer's reverse key) |

For the two outputs, **pick pins that are HIGH at boot**; a pin that starts LOW runs the motor while Klipper starts. Keep `shutdown_value: 1` on both, so an emergency stop leaves the motor idle.

### Buffer firmware timeout

If your PTFE tube is longer than about 1.2m, raise the firmware's feed timeout, or the buffer may give up before new filament reaches the extruder. It then stops following the extruder until it's reset.

Connect the buffer's USB port to a computer, open its serial port at 115200 baud, and send each command on its own line:

```
info              # show settings, including timeout (default 60000 ms)
timeout 120000    # 120s, saved on the buffer
rt                # read the timeout back
```

## Usage with Demon Klipper Essentials Unified

Demon runs heating, purging, tip shaping and parking. This repo adds the buffer through four hooks in Demon's user file `Demon_User_Files/demon_custom_expansion_v*.cfg`. `demon_buffer_integration.cfg` has the exact lines. In short:

| Demon hook | Body |
|---|---|
| `_CUSTOM_PRE_LOAD`, `_CUSTOM_PRE_LOAD_CLEAN` | `Buffer_Assert_Filament_Detected` |
| `_CUSTOM_POST_UNLOAD`, `_CUSTOM_POST_UNLOAD_CLEAN` | `Buffer_Retract_Until_Runout TIMEOUT=90 POLL=2.0` |

Set the matching flags (`pre_load`, `post_unload`, `pre_load_clean`, `post_unload_clean`) to `True`. Demon updates can reset that file, so check the hooks after updating Demon.

Recommended settings for a Stealthburner/Galileo 2 with a Revo (100mm from nozzle tip to gears):

| File | Setting | Why |
|---|---|---|
| Demon user settings | `load_length: 80` | The fast load move stops before the melt zone |
| Demon user settings | `load_purge_length: 70` | The rest of the path plus about 50mm of purge, at 7mm/s |
| Demon user settings | `unload_length: 20` | Demon only pulls the filament out of the hot zone; the rest happens with both motors |
| Demon user settings | `max_extrude_speed: 7` | A purge speed the hotend can melt. Demon warns about values below 15, which is harmless. |
| Your buffer settings | `unload_assist_length: 90`, `unload_assist_speed: 18` | The extruder releases the filament while the buffer pulls |

**Loading:** insert filament into the buffer and wait for it to feed to the extruder gears and stop. Then run `LOAD_FILAMENT`.

**Unloading:** run `UNLOAD_FILAMENT`.
1. Demon purges, shapes the tip and pulls back a little.
2. The extruder and buffer then pull together for a few seconds.
3. The buffer continues alone until the filament leaves its inlet. You'll see `✓ Buffer: Filament ejected`.

**M600:** Demon parks and waits. Run `UNLOAD_FILAMENT`, swap the filament, then `LOAD_FILAMENT` and `RESUME`.

**After a runout:** the print pauses. `UNLOAD_FILAMENT` releases the extruder and pulls the leftover piece back out through the buffer.

## Usage without Demon

```gcode
BUFFER_LOAD_FILAMENT                      # home, park, heat, engage, advance to the nozzle, purge
BUFFER_LOAD_FILAMENT TEMP=230 SPEED=5
BUFFER_UNLOAD_FILAMENT                    # home, park, heat, purge, phased retraction, buffer retraction
BUFFER_UNLOAD_FILAMENT TEMP=230 COOL=No
```

## Commands

| Command | Description |
|---|---|
| `BUFFER_LOAD_FILAMENT` | Standalone load. Params: `TEMP`, `SPEED`, `ENGAGE_LENGTH`, `PURGE_LENGTH`, `RETRACT_LENGTH`, `COOL`, `COOL_TEMP` |
| `BUFFER_UNLOAD_FILAMENT` | Standalone unload. Params: `TEMP`, `SPEED`, `UNLOAD_LENGTH`, `COOL`, `COOL_TEMP` |
| `Buffer_Retract_Until_Runout` | Retracts in the background until the inlet switch clears. Params: `TIMEOUT` (motor-on seconds, 90), `POLL` (segment seconds, 2.0, minimum 1.0), `ASSIST` (mm), `ASSIST_SPEED` (mm/s) |
| `Buffer_Assert_Filament_Detected` | Errors if the buffer has no filament or a retraction is still running |
| `BUFFER_STOP` | Stops the motor and cancels a running retraction |
| `Buffer_Feeding` / `Buffer_Retraction` | Run the buffer forward or back for 10s |

## Settings

In your copy of `mellow_buffer_user_settings.cfg`. If you add a variable to an older copy, put it with the other `variable_` lines, above `gcode:`.

| Variable | Default | Meaning |
|---|---|---|
| `park_x`, `park_y`, `park_min_z` | 325, 348, 10 | Park position for the standalone macros |
| `load_temp`, `unload_temp` | 250 | Default temperatures (°C) |
| `load_speed`, `unload_speed` | 7.0 | Extruder speed for standalone load/unload (mm/s) |
| `engage_length` | 20 | First load move; the rest of `hotend_path_length` follows before the purge |
| `load_purge_length`, `unload_purge_length` | 50, 25 | Purge lengths (mm) |
| `load_retract_length` | 10 | Anti-ooze retraction after the load purge (mm) |
| `hotend_path_length` | 100 | Nozzle tip to extruder gears (mm) |
| `buffer_startup_delay`, `buffer_pulse_interval`, `buffer_pulse_duration` | 0.5, 5.0, 0.3 | Buffer pulsing during the standalone unload |
| `filament_tail_extra_extrude` | 10 | Extra retraction at the end of the standalone unload (mm) |
| `unload_assist_length` | 0 (off) | Extruder retraction while the buffer starts pulling (Demon unload). 90 is suggested with Demon `unload_length: 20`. |
| `unload_assist_speed` | 18 | Speed of that retraction. Keep it below the buffer's ~23mm/s. |
| `nozzle_clean_macro` | `CLEAN_NOZZLE` | Any macro, parameters allowed; empty to disable. Skipped with a warning if it doesn't exist. |
| `cooldown`, `cooldown_temp` | Yes, 150 | Cooldown after the standalone macros |

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| Buffer motor runs while Klipper starts | An output pin is LOW at boot. Use pins that start HIGH, with `shutdown_value: 1`. |
| "No filament in buffer" | Filament isn't past the buffer's inlet switch. Insert it further and let the buffer feed it. |
| "Still retracting from the last unload" | Wait for `Filament ejected`, or run `BUFFER_STOP`. |
| The extruder can't grab the filament | The tip hasn't reached the gears, or it's bent. Recut it at an angle and let the buffer feed it until it stops at the gears. |
| Extruder skips during a load or unload purge | The purge is faster than the hotend can melt. Use the Demon settings above (a short fast move, then a 7mm/s purge). |
| Unload "snaps", or the filament end stays in the extruder | The extruder pushed filament back while the buffer was idle. Use Demon `unload_length: 20` with `unload_assist_length: 90`. |
| Unload ends with "Timeout reached" but the filament came out | Your tube needs more time. Raise `TIMEOUT` in the hook line. |
| The buffer stops feeding or following the extruder | The firmware's feed timeout fired. Short-press the buffer's forward key or run `Buffer_Feeding` to reset it, and raise the timeout. |
| "No filament came back through the buffer inlet" (after a runout) | The leftover piece didn't reach the switch within 20s. Pull it out by hand. |

## Support

- Firmware: [patofoto/Buffer](https://github.com/patofoto/Buffer)
- Klipper: [klipper3d.org](https://www.klipper3d.org/)
- This repository: open an issue
