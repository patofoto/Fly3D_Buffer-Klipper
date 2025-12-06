# Klipper Configuration Files for FLY-LLL PLUS Buffer

This repository contains Klipper configuration files for integrating the FLY-LLL PLUS Buffer with your 3D printer.

## Prerequisites

**⚠️ Firmware Required:** This Klipper configuration requires the FLY-LLL PLUS Buffer to be flashed with firmware v1.1.x or later.

- **Firmware Repository:** [Buffer Firmware](https://github.com/patofoto/Buffer)
- **Firmware Installation:** See the [firmware repository README](https://github.com/patofoto/Buffer) for firmware compilation and flashing instructions

## Files

- **`mellow_buffer_klipper.cfg`** - Basic buffer configuration with pin definitions, sensor setup, and manual feed/retract macros
- **`mellow_buffer_macros.cfg`** - Advanced standalone integration with automatic filament unload and buffer retraction
- **`mellow_buffer_user_settings.cfg`** - User-configurable variables (park positions, temperatures, speeds, etc.)

## Installation

### Quick Start

1. Copy the configuration files to your Klipper `printer_data/config/` folder:
   ```bash
   cp mellow_buffer_klipper.cfg ~/printer_data/config/
   cp mellow_buffer_macros.cfg ~/printer_data/config/
   cp mellow_buffer_user_settings.cfg ~/printer_data/config/
   ```

2. Add includes to your `printer.cfg`:
   ```ini
   [include ./mellow_buffer_user_settings.cfg]
   [include ./mellow_buffer_klipper.cfg]
   [include ./mellow_buffer_macros.cfg]
   ```

3. **Important**: Update the pin assignments in `mellow_buffer_klipper.cfg` to match your printer's GPIO pins. The current pins (PA8, PC9, PF4, PF1, PF0) are for a Voron 2.4 with a specific MCU configuration.

4. Restart Klipper to load the new configuration.

### Installation via Moonraker Update Manager (Recommended)

For automatic updates, you can use Moonraker Update Manager to track the `standalone-buffer` branch and receive updates automatically through your Mainsail/Fluidd interface.

#### Initial Setup

1. **Clone this repository to your config folder:**
   ```bash
   cd ~/printer_data/config
   git clone https://github.com/patofoto/Fly3D_Buffer-Klipper.git Fly3D_Buffer
   ```

2. **Add the update manager entry to your `moonraker.conf`:**
   ```ini
   [update_manager Fly3D_Buffer]
   type: git_repo
   path: ~/printer_data/config/Fly3D_Buffer
   origin: https://github.com/patofoto/Fly3D_Buffer-Klipper.git
   primary_branch: main
   is_system_service: False
   managed_services: klipper
   ```

3. **Copy the user settings file to your config directory (outside tracked folder):**
   ```bash
   cp ~/printer_data/config/Fly3D_Buffer/mellow_buffer_user_settings.cfg ~/printer_data/config/
   ```
   This allows you to edit variables without Moonraker making them read-only.

4. **Update your `printer.cfg` includes:**
   ```ini
   [include ./mellow_buffer_user_settings.cfg]  # Your editable copy (outside tracked folder)
   [include ./Fly3D_Buffer/mellow_buffer_klipper.cfg]
   [include ./Fly3D_Buffer/mellow_buffer_macros.cfg]
   ```

5. **Restart Moonraker:**
   ```bash
   sudo systemctl restart moonraker
   ```

#### Using Moonraker Update Manager

Once configured, you can manage updates through your Mainsail or Fluidd interface:

- **Check for updates**: Navigate to the "Updates" section in Mainsail/Fluidd
- **View available updates**: You'll see "Fly3D_Buffer" listed if updates are available
- **Apply updates**: Click "Update" to pull the latest changes from the `standalone-buffer` branch
- **Automatic restart**: Klipper will automatically restart after updates are applied

#### Benefits

- **Automatic updates**: Get notified when new versions are available
- **Easy management**: Update through the web interface without SSH access
- **Version tracking**: Moonraker tracks the repository and branch automatically
- **Safe updates**: Klipper restarts automatically after updates

**Note**: If you customize pin assignments or other settings in `mellow_buffer_klipper.cfg`, be aware that updates may overwrite your changes. Consider keeping a backup or using a separate config file for customizations.

### Pin Configuration

Before using these configs, you **must** update the pin assignments in `mellow_buffer_klipper.cfg` to match your printer's GPIO configuration:

- **`_Feed_Button`** (output_pin): Pin that controls buffer forward feeding
- **`_Retract_Button`** (output_pin): Pin that controls buffer reverse retraction
- **`filament_sensor`** (filament_switch_sensor): Pin for filament runout detection
- **Trigger buttons** (gcode_button): Optional pins for manual trigger buttons

**Critical**: Choose pins that default to **HIGH** on boot to prevent unwanted motor movement during Klipper startup. The buffer firmware uses active-low logic (LOW = pressed, HIGH = released).

### Features

#### Basic Configuration (`mellow_buffer_klipper.cfg`)
- Filament runout detection with pause on runout
- Manual buffer feeding macro (`Buffer_Feeding`)
- Manual buffer retraction macro (`Buffer_Retraction`)
- Pin definitions for buffer control

#### Standalone Integration (`mellow_buffer_macros.cfg`)
- **`BUFFER_UNLOAD_FILAMENT`** - Complete filament unload macro that:
  - Auto-homes printer if not homed
  - Parks toolhead at safe position
  - Manages hotend temperature
  - Unloads filament from hotend
  - Retracts buffer until runout sensor detects no filament
  - Optional cooldown after unload

- **`Buffer_Retract_Until_Runout`** - Helper macro for buffer retraction with sensor polling

### Usage

#### Basic Manual Control
```gcode
Buffer_Feeding      # Feed filament for 10 seconds
Buffer_Retraction   # Retract filament for 10 seconds
```

#### Advanced Unload
```gcode
BUFFER_UNLOAD_FILAMENT                    # Standard unload
BUFFER_UNLOAD_FILAMENT TEMP=230          # Unload at 230°C
BUFFER_UNLOAD_FILAMENT TEMP=230 COOL=Yes # Unload and cool down
```

### User Settings Configuration

All user-configurable variables are in `mellow_buffer_user_settings.cfg`. This file should be **copied outside the tracked folder** when using Moonraker Update Manager, as Moonraker makes tracked files read-only.

#### Editing Variables with Moonraker

1. **Copy the settings file to your config directory:**
   ```bash
   cp ~/printer_data/config/Fly3D_Buffer/mellow_buffer_user_settings.cfg ~/printer_data/config/
   ```

2. **Edit your copy:**
   ```bash
   nano ~/printer_data/config/mellow_buffer_user_settings.cfg
   ```

3. **Update `printer.cfg` to include your copy:**
   ```ini
   [include ./mellow_buffer_user_settings.cfg]  # Your editable copy
   [include ./Fly3D_Buffer/mellow_buffer_klipper.cfg]
   [include ./Fly3D_Buffer/mellow_buffer_macros.cfg]
   ```

#### Available Variables

Edit `mellow_buffer_user_settings.cfg` to customize:
```ini
variable_park_x: 325.0               # X parking position
variable_park_y: 348.0               # Y parking position
variable_park_min_z: 10.0            # Minimum Z height
variable_unload_purge_length: 25.0   # Purge length (mm)
variable_unload_temp: 250            # Unload temperature (°C)
variable_unload_speed: 7.0           # Unload speed (mm/s)
variable_hotend_path_length: 100.0   # Distance from nozzle to extruder (mm)
variable_buffer_pulse_interval: 5.0  # Buffer pulse interval (mm)
variable_buffer_pulse_duration: 0.3  # Buffer pulse duration (seconds)
# ... and more
```

### Customization

#### Adjust Buffer Retraction Parameters

Modify the `Buffer_Retract_Until_Runout` call in `BUFFER_UNLOAD_FILAMENT`:
```ini
Buffer_Retract_Until_Runout TIMEOUT=60 POLL=0.5
```

- `TIMEOUT`: Maximum retraction time in seconds (safety limit, default: 60)
- `POLL`: Poll interval in seconds (default: 0.5)

**Note**: Retraction continues until the runout sensor detects no filament, or the timeout is reached (safety limit).

### Troubleshooting

#### Motor moves during Klipper startup
- **Cause**: Pin defaults to LOW on boot
- **Fix**: Change to a pin that defaults HIGH (check your MCU datasheet)

#### Buffer retraction doesn't stop
- **Cause**: Runout sensor not detecting filament absence
- **Fix**: Check sensor wiring and pin assignment in `mellow_buffer_klipper.cfg`

#### Filament not unloading completely
- **Cause**: Unload length too short (auto-calculated from `hotend_path_length`)
- **Fix**: Increase `variable_hotend_path_length` in `mellow_buffer_user_settings.cfg` or use `UNLOAD_LENGTH` parameter

### Requirements

- Klipper firmware
- FLY-LLL PLUS Buffer with firmware v1.1.x or later
- Properly configured GPIO pins on your printer's MCU
- Filament runout sensor connected to buffer board

### Support

- **Firmware Issues:** See the [Buffer Firmware repository](https://github.com/patofoto/Buffer) for firmware-related problems
- **Klipper Issues:** Consult the [Klipper documentation](https://www.klipper3d.org/)
- **This Repository:** For issues with these Klipper configuration files, open an issue in this repository

