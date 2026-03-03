# CLAUDE.md — Fly3D Buffer Klipper Project

## Project Overview

Klipper macros and configuration for integrating the **FLY-LLL PLUS filament buffer** with a **Voron 2.4 350mm** running Klipper firmware. The primary goal is to enable fully automated filament load/unload using the buffer motor, coordinated with the extruder.

### Primary Integration Target
Macros must be compatible with (or callable from) the load/unload macros in:
**[Demon_Klipper_Essentials_Unified](https://github.com/3DPrintDemon/Demon_Klipper_Essentials_Unified)**

---

## Hardware Context

| Component | Details |
|---|---|
| Printer | Voron 2.4 350mm |
| Firmware | Klipper |
| Buffer | FLY-LLL PLUS (Mellow/Fly3D) |
| Buffer firmware | v1.1.x+ — [patofoto/Buffer](https://github.com/patofoto/Buffer) (forked from [Fly3DTeam/Buffer](https://github.com/Fly3DTeam/Buffer)) |
| Printer Klipper config backup | [patofoto/Voron_2.4_350_Backup](https://github.com/patofoto/Voron_2.4_350_Backup/tree/main/printer_data/config) |
| Extruder | Stealthburner + Galileo 2 (gear ratio 9:1, rotation_distance 48.02976) |

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

1. Auto-home if needed → park → heat
2. Check filament detected at sensor
3. Extruder engagement (3 retries) with retract test to confirm grip
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

Integration uses Demon's `demon_custom_expansion.cfg` hooks — a user-owned file that Demon's update manager never overwrites. Our macros plug into four hooks around Demon's native load/unload sequences. See `demon_buffer_integration.cfg` in this repo for the exact code to paste.

### Hook mapping

| Demon hook | Buffer action | Macro called |
|---|---|---|
| `_CUSTOM_PRE_LOAD` | Check filament is at extruder before Demon engages | `Buffer_Assert_Filament_Detected` |
| `_CUSTOM_POST_UNLOAD` | Retract filament tail out of buffer after Demon clears hotend | `Buffer_Retract_Until_Runout TIMEOUT=60 POLL=0.5` |
| `_CUSTOM_PRE_LOAD_CLEAN` | Same as PRE_LOAD (applies to LOAD_CLEAN) | `Buffer_Assert_Filament_Detected` |
| `_CUSTOM_POST_UNLOAD_CLEAN` | Same as POST_UNLOAD (applies to UNLOAD_CLEAN) | `Buffer_Retract_Until_Runout TIMEOUT=60 POLL=0.5` |

### End-to-end flows

**LOAD_FILAMENT:**
1. User inserts filament → buffer firmware auto-feeds via HALL sensor → filament arrives at extruder
2. User calls Demon's `LOAD_FILAMENT`
3. `_CUSTOM_PRE_LOAD` → `Buffer_Assert_Filament_Detected` → aborts with clear message if sensor not triggered yet
4. Demon heats hotend, engages extruder, purges

**UNLOAD_FILAMENT:**
1. User calls Demon's `UNLOAD_FILAMENT`
2. Demon heats hotend, retracts ~32mm net — enough to clear the extruder grip
3. `_CUSTOM_POST_UNLOAD` → `Buffer_Retract_Until_Runout` → buffer motor runs until sensor clears (60s max)

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
The `BUFFER_LOAD_FILAMENT` engagement retry loop uses this pattern — it appears to retry but `engagement_success` never actually changes. Workaround: restructure logic to avoid needing a mutable flag, or use a single-pass approach.

### No `break` in loops
Jinja2 has no `break` statement. The `Buffer_Retract_Until_Runout` macro works around this with a `{% if not runout_detected %}` guard inside the loop body, but all iterations still execute (with `G4` waits), so the loop always runs to `max_iterations`. It stops the motor early but still waits out the full timeout in practice.

### Template renders at parse time, not execution time
Sensor reads like `printer["filament_switch_sensor filament_sensor"].filament_detected` inside a `{% if %}` block are evaluated when the template is first rendered — meaning conditional branches based on live printer state inside loops may not reflect what the printer is actually doing mid-execution. Use GCode commands (not Jinja2 logic) for real-time decisions.

### Use `{ action_raise_error("message") }` not `ABORT`
`ABORT` is **not** a built-in Klipper command. The current macros call `ABORT` on error conditions which will generate a "Unknown command: ABORT" error. The correct way to stop a macro with an error message is:
```jinja
{ action_raise_error("Error message here") }
```
Or define a custom `[gcode_macro ABORT]` that calls `{ action_raise_error("Aborted") }`.

### Speed is in mm/min in GCode, mm/s in variables
`G1 F{speed}` expects mm/min. User variables are in mm/s. Always multiply by 60: `{% set speed = speed_mmsec * 60 %}`.

---

## Filament Sensor Behavior

The sensor (`filament_switch_sensor filament_sensor`) has `pause_on_runout: true`. Implications:
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
