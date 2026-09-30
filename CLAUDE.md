# CLAUDE.md — Fly3D Buffer Klipper Project

## Project Overview

Klipper macros and configuration for integrating the **FLY-LLL PLUS filament buffer** with a **Voron 2.4 350mm** running Klipper firmware. The primary goal is to enable fully automated filament load/unload using the buffer motor, coordinated with the extruder.

### Primary Integration Target
Macros must be compatible with (or callable from) the load/unload macros in:
**[Demon_Klipper_Essentials_Unified](https://github.com/3DPrintDemon/Demon_Klipper_Essentials_Unified)**

---

## Printer SSH Access

The live printer is accessible via SSH for reading configs and installed macros:
- **Host:** `patofoto@10.0.10.44` (SSH key auth)
- **Config path:** `~/printer_data/config/`
- **Password:** provide in chat when needed (do not store here — file is tracked by git)

## Preferred Workflow for Printer Changes

1. **Repo-tracked files** (anything in `Fly3D_Buffer/`) — edit locally, commit, push to GitHub, Moonraker syncs automatically. Never write these directly on the printer.
2. **Printer-only files** (e.g. `Demon_User_Files/demon_custom_expansion_v*.cfg`) — these can't sync from our repo. Provide the user with the exact lines to change and let them edit in Mainsail/Fluidd. Do NOT write directly via SSH unless the user explicitly asks.
3. **SSH reads** are acceptable for inspecting live config, but prefer asking the user to paste content from Mainsail to save tokens.

---

## Hardware Context

| Component | Details |
|---|---|
| Printer | Voron 2.4 350mm |
| Firmware | Klipper |
| Buffer | FLY-LLL PLUS (Mellow/Fly3D) |
| Buffer firmware | v1.1.x+ — [patofoto/Buffer](https://github.com/patofoto/Buffer) (forked from [Fly3DTeam/Buffer](https://github.com/Fly3DTeam/Buffer)) |
| Printer Klipper config backup | [patofoto/Voron_2.4_350_Backup](https://github.com/patofoto/Voron_2.4_350_Backup/tree/main/printer_data/config) |
| Hotend | Revo Voron |
| Extruder | Stealthburner + Galileo 2 (gear ratio 9:1, rotation_distance 48.02976) |
| hotend_path_length | 100mm (Revo Voron nozzle tip → Galileo 2 gear center) |
| Buffer→Extruder PTFE | ~1345mm (buffer motor exit → extruder entrance) |

---

## Repository Files

| File | Purpose |
|---|---|
| `mellow_buffer_klipper.cfg` | Pin definitions, filament sensor, manual feed/retract macros |
| `mellow_buffer_macros.cfg` | Full load/unload automation macros |
| `mellow_buffer_user_settings.cfg` | All user-configurable variables (should be copied outside tracked folder for Moonraker) |
| `MOTOR_SPEED_REFERENCE.md` | Technical reference for buffer/extruder motor speed calculations |
| `test_extruder_unload.cfg` | Test macros for extruder unload development |

---

## Critical Pin Logic (Active-Low)

The FLY-LLL PLUS buffer firmware uses **active-low button logic**:
- `VALUE=0` (LOW) = button pressed = **motor moves**
- `VALUE=1` (HIGH) = button released = **motor idle**

**Pin requirements:**
- Use pins that default **HIGH** on boot (e.g., PA8, PC9) — prevents motor movement on startup
- Both output pins MUST have `shutdown_value:1` — so emergency stop sets pins HIGH (motor idle), not LOW (motor runs)
- Do NOT use pins that default LOW (like PC15, PE9)

**Current pin assignments (Voron 2.4 specific):**

| Signal | Printer Pin | Buffer Board Pin | Wire |
|---|---|---|---|
| _Feed_Button | PA8 | PB5 (KEY2 Forward) | White |
| _Retract_Button | PC9 | PB6 (KEY1 Reverse) | Black |
| filament_sensor | PF4 | PB15 | White |
| Trigger Feeding button | ^!PF1 | PA2 | White |
| Trigger Retraction button | ^!PF0 | PA3 | Red |

---

## Macro Architecture

### Macro Summary

| Macro | File | Purpose |
|---|---|---|
| `BUFFER_UNLOAD_FILAMENT` | macros.cfg | Full unload: home → park → heat → purge → retract hotend (phased) → buffer retract until runout |
| `BUFFER_LOAD_FILAMENT` | macros.cfg | Full load: home → park → heat → engage extruder → purge → retract |
| `Buffer_Retract_Until_Runout` | macros.cfg | Helper: runs buffer retraction, polls runout sensor, stops when filament gone |
| `Buffer_Feeding` | klipper.cfg | Manual: feeds filament 10 seconds |
| `Buffer_Retraction` | klipper.cfg | Manual: retracts filament 10 seconds |
| `BUFFER_STOP` | klipper.cfg | Emergency: releases both pins immediately |
| `_BUFFER_USER_SETTINGS` | user_settings.cfg | Variable container for all configurable values |

### Unload Phases

1. **Phase 1** — Initial retraction (40% of hotend_path_length) while extruder still grips filament
2. **Phase 2/3** — Smart pulsing: extruder retracts in segments, buffer pulses after each to take up slack
3. **Phase 4** — Extra extrude distance for filament stringing (`variable_filament_tail_extra_extrude`)
4. **Buffer_Retract_Until_Runout** — Buffer continues retracting until runout sensor confirms filament gone (60s timeout)

### Load Flow

1. Preflight checks (settings loaded, filament at sensor, hotend hot enough if mid-print) — evaluated at render time, so a failure raises before anything moves
2. Auto-home if needed → park → heat
3. Engage `engage_length`, then advance the rest of `hotend_path_length` to the nozzle — single pass at `load_speed`, no grip test (the sensor is the buffer's inlet switch, so it can't detect grip)
4. Purge → retract to prevent oozing
5. Optional nozzle clean → optional cooldown

---

## Key User Variables (`mellow_buffer_user_settings.cfg`)

```ini
variable_version: "1.1.13"
variable_park_x: 325.0               # Parking position for Voron 2.4 350
variable_park_y: 348.0
variable_park_min_z: 10.0
variable_unload_temp: 250            # Default unload temp (°C)
variable_load_temp: 250
variable_unload_speed: 7.0           # mm/s (extruder)
variable_hotend_path_length: 100.0   # Nozzle tip to extruder distance (Stealthburner/Galileo 2)
variable_buffer_startup_delay: 0.5   # Seconds before starting buffer retraction
variable_buffer_pulse_interval: 5.0  # mm of extruder retraction between buffer pulses
variable_buffer_pulse_duration: 0.3  # Seconds buffer runs per pulse
variable_filament_tail_extra_extrude: 10.0  # Extra mm for stringing handling
variable_nozzle_clean_macro: "CLEAN_NOZZLE"  # Macro to call after load/unload
variable_cooldown: "Yes"
variable_cooldown_temp: 150
```

---

## Motor Speed Reference

| Motor | Speed |
|---|---|
| Buffer motor | 260 RPM (firmware default) |
| Extruder motor | ~79 RPM at 7.0 mm/s (gear ratio 9:1) |
| Buffer:Extruder ratio | ~3.3:1 (buffer is faster) |

The buffer firmware uses `VACTUAL` register: `SPEED * 64 * 200 / 60 / 0.715 ≈ 77576` at 260 RPM.

---

## Demon Klipper Essentials Integration Notes

Integration uses Demon's `demon_custom_expansion_v*.cfg` hooks — a user-owned file that Demon's update manager never overwrites. Our macros plug into four hooks around Demon's native load/unload sequences. See `demon_buffer_integration.cfg` in this repo for the exact code to paste.

### DKEU User Variable File Locations

All user-editable DKEU files live in:
```
~/printer_data/config/Demon_User_Files/
```
This directory is never touched by Demon's Moonraker update manager. **Filenames include version numbers that change with each DKEU update** — always `ls` the directory to find current names before editing. File patterns and purposes:

| Pattern | Purpose |
|---|---|
| `demon_custom_expansion_v*.cfg` | Buffer integration hooks go here |
| `demon_user_settings_v*.cfg` | Core settings: `unload_length`, park position, auto-cool, etc. |
| `demon_user_settings_filament_variables_v*.cfg` | Per-filament PA, retraction, temps |
| `demon_user_settings_cleaner_variables_v*.cfg` | Nozzle cleaner / purge bucket settings |

### Hook mapping

| Demon hook | Buffer action | Macro called |
|---|---|---|
| `_CUSTOM_PRE_LOAD` | Check filament is in the buffer (inlet switch) before Demon engages | `Buffer_Assert_Filament_Detected` |
| `_CUSTOM_POST_UNLOAD` | Buffer retracts filament tail after Demon's unload | `Buffer_Retract_Until_Runout TIMEOUT=90 POLL=2.0` |
| `_CUSTOM_PRE_LOAD_CLEAN` | Same as PRE_LOAD (applies to LOAD_CLEAN) | `Buffer_Assert_Filament_Detected` |
| `_CUSTOM_POST_UNLOAD_CLEAN` | Same as POST_UNLOAD (applies to UNLOAD_CLEAN) | `Buffer_Retract_Until_Runout TIMEOUT=90 POLL=2.0` |

### End-to-end flows

**LOAD_FILAMENT:**
1. User inserts filament → inlet switch triggers → buffer firmware auto-feeds until its slider hits HALL pos2 (filament stopped at the extruder gears) or 60s firmware timeout
2. User calls Demon's `LOAD_FILAMENT`
3. `_CUSTOM_PRE_LOAD` → `Buffer_Assert_Filament_Detected` → aborts with clear message if no filament in the buffer (it can't tell whether feeding has finished)
4. Demon heats hotend, engages extruder, purges

**UNLOAD_FILAMENT:**
1. User calls Demon's `UNLOAD_FILAMENT`
2. Demon heats, tip-shapes, then retracts `unload_length` (125mm on this printer) — net ~157mm total, clearing the 100mm hotend path. Extruder has no filament grip when hook fires.
3. `_CUSTOM_POST_UNLOAD` → `Buffer_Retract_Until_Runout` → buffer retracts in segments until the inlet switch clears (~60s through the 1345mm PTFE, 90s max)

### Enable flags required in `_CUSTOM_EXPANSION_ACTIVE_LIST`
```ini
variable_ceal_master_enable: True   # master switch
variable_pre_load:          True
variable_post_unload:       True
variable_pre_load_clean:    True
variable_post_unload_clean: True
```

### Standalone alternative
`BUFFER_UNLOAD_FILAMENT` and `BUFFER_LOAD_FILAMENT` remain as self-contained alternatives for users without Demon (handles homing, parking, heating, and buffer coordination all in one macro).

### M600 (future)
Demon's `FIL_CHANGE_PARK` handles mid-print parking. Automatic M600 chaining (park → auto-unload → wait → auto-load) is planned after load/unload hooks are validated on the printer.

---

## Klipper Jinja2 Limitations (Critical for Development)

Klipper renders the **entire Jinja2 template before any GCode executes**. This causes several non-obvious pitfalls:

### Variables inside loops don't update
```jinja
{% set success = False %}
{% for i in range(3) %}
    {% set success = True %}   {# This does NOT persist outside this iteration #}
{% endfor %}
{# success is still False here #}
```
`BUFFER_LOAD_FILAMENT` used to have an engagement retry loop built on this pattern: every attempt ran, and the failure block fired afterwards anyway. It's now a single pass.

### No `break` in loops
Klipper's Jinja environment enables no extensions, so there is no `break`/`continue`. To poll until a condition, use a `[delayed_gcode]` that reschedules itself — see `Buffer_Retract_Until_Runout` / `_buffer_retract_poll`. `delayed_gcode` waits for the G-code mutex, so its ticks don't run until the calling top-level command has finished.

### Template renders at parse time, not execution time
Sensor reads like `printer["filament_switch_sensor filament_sensor"].filament_detected` inside a `{% if %}` block are evaluated when the template is first rendered — meaning conditional branches based on live printer state inside loops may not reflect what the printer is actually doing mid-execution. Use GCode commands (not Jinja2 logic) for real-time decisions. A nested macro call renders when its line executes, so a helper macro called after `M400` does see fresh state.

### Use `{ action_raise_error("message") }` not `ABORT`
`ABORT` is **not** a Klipper command. Unknown commands only print `Unknown command:"ABORT"` and the macro **keeps running**. The correct way to stop a macro with an error message is:
```jinja
{ action_raise_error("Error message here") }
```
It raises while the template renders, so no line of that macro runs — including `RESPOND` lines placed before it. Put all user guidance in the error text. Avoid ` #` and ` ;` inside messages: Klipper's config parser strips them as inline comments.

### Speed is in mm/min in GCode, mm/s in variables
`G1 F{speed}` expects mm/min. User variables are in mm/s. Always multiply by 60: `{% set speed = speed_mmsec * 60 %}`.

---

## Filament Sensor Behavior

The sensor is the buffer's **inlet filament switch** (firmware ENDSTOP_3 on PB7), relayed to the printer on buffer PB15 → PF4. There is no sensor at the extruder: "detected" only means filament is in the buffer.

**PB15 freezes while the retract signal is held.** The firmware's `motor_control()` sits in a `while (BACK_SIGNAL_PIN == LOW)` loop and never refreshes PB15, so holding `_Retract_Button` LOW keeps the sensor at "present" no matter where the filament is. `Buffer_Retract_Until_Runout` releases the pin for 0.25s between retraction segments so the firmware can refresh it. The same applies to `_Feed_Button` (`FRONT_SIGNAL_PIN`).

Other firmware behavior (patofoto/Buffer `lib/buffer/buffer.cpp`): auto-feed stops with an internal error after 60s of continuous forward motion, and releasing the retract signal also sets that error flag, so auto-feed stays off until a forward press or until the filament leaves the inlet switch.

The sensor has `pause_on_runout: true`. Implications:
- If the sensor triggers **during a print** (e.g., runout), Klipper calls `PAUSE` automatically — this is the desired runout behavior
- During `BUFFER_UNLOAD_FILAMENT`, the sensor going LOW is the *success* condition (filament ejected) — the macro waits for this in `Buffer_Retract_Until_Runout`
- During `BUFFER_LOAD_FILAMENT`, the sensor being HIGH is the *precondition* — if it's LOW, no filament is present and the macro aborts
- `event_delay: 2.0` and `debounce_delay: 2.0` mean state changes take up to 2 seconds to register — factor this into polling intervals

---

## Development Conventions

- All macros use Jinja2 templating standard to Klipper (no Python, no moonraker-only features)
- Variables in `_BUFFER_USER_SETTINGS` are the single source of truth — never hardcode values in macros.cfg
- Use `RESPOND MSG=` for user feedback (visible in Mainsail/Fluidd console)
- Use `SET_DISPLAY_TEXT MSG=` for display/LCD text (shorter messages)
- Always `SAVE_GCODE_STATE` / `RESTORE_GCODE_STATE` in load/unload macros
- Filament sensor name: `filament_switch_sensor filament_sensor`
- Feed pin: `_Feed_Button`, Retract pin: `_Retract_Button`
- `VALUE=0` = motor on, `VALUE=1` = motor off (active-low — this is counterintuitive, document it)

## Model Selection Guide

Use the cheapest model that can handle the task. Escalate only when needed.

| Model | Use for |
|---|---|
| **haiku** | Reading files, searching code, answering questions, simple single-line edits, fetching URLs |
| **sonnet** | Most coding tasks: writing/editing macros, debugging Jinja2, multi-file changes, planning |
| **opus** | Complex architectural decisions, hard debugging across many files, ambiguous requirements |

### Project-specific guidance

- **Exploring the codebase** (Glob, Grep, Read) → haiku
- **Editing a single macro or variable** → haiku
- **Writing a new macro with logic** (loops, conditionals, phased sequences) → sonnet
- **Debugging why a macro misbehaves at runtime** → sonnet
- **Designing integration with Demon Klipper Essentials** → sonnet
- **Major refactor across multiple .cfg files** → sonnet
- **Resolving ambiguous hardware/firmware interactions** → opus

---

## Moonraker Update Manager

- `mellow_buffer_user_settings.cfg` should be **copied outside** the tracked folder — Moonraker makes tracked files read-only
- Updates are distributed via the `main` branch
- `managed_services: klipper` causes Klipper to restart after updates

## Current Version

`v1.1.13` — Latest fix: `shutdown_value:1` on both output pins (emergency stop safety)
