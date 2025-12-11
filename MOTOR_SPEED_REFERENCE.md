# Motor Speed Control Reference

This document contains technical information about motor speed control for the FLY-LLL PLUS Buffer and extruder synchronization.

## Buffer Motor Speed Control

### Overview
The buffer firmware controls motor speed through a chain of calculations that convert RPM to TMC2209 driver register values.

### Key Components

#### SPEED Variable
- **Location**: Line 107 in `buffer.h`
- **Default Value**: **260 RPM**
- **Type**: Static variable
- **Purpose**: Holds the target motor speed in RPM

#### VACTRUAL_VALUE Calculation
- **Formula**: `VACTRUAL_VALUE = SPEED * Move_Divide_NUM * 200 / 60 / 0.715`
- **Location**: Lines 109, 179, 864 in `buffer.h`
- **Purpose**: Converts RPM to the TMC2209 VACTUAL register value

**Formula Breakdown:**
- `SPEED`: Motor speed in RPM (default: 260)
- `Move_Divide_NUM`: Microsteps = **64**
- `200`: Steps per revolution (typical for stepper motors)
- `60`: Conversion factor (seconds to minutes)
- `0.715`: TMC2209-specific conversion factor

**Example Calculation (Default Speed):**
```
VACTRUAL_VALUE = 260 * 64 * 200 / 60 / 0.715
VACTRUAL_VALUE = 3,328,000 / 42.9
VACTRUAL_VALUE ≈ 77,576
```

#### TMC2209 Driver Control
- **Location**: Lines 399, 424, 552, 574 in `buffer.h`
- **Method**: `driver.VACTUAL(VACTRUAL_VALUE)`
- **Usage**: Sets motor velocity in forward, backward, and button control paths
- **Initialization**: Calculated at initialization (line 179) and when speed changes (line 864)

## Extruder Motor Speed

### Current Configuration
- **Status**: ✅ **Configured**
- **Variable**: `variable_extruder_motor_rpm: 79` (at default 7.0 mm/s unload/load speed)

### Extruder Configuration
```ini
[extruder]
rotation_distance: 48.02976  # mm per full extruder gear rotation
gear_ratio: 9:1             # Motor rotates 9 times per extruder gear rotation
microsteps: 16
```

### Calculation Method

For a geared extruder, the motor rotation distance must account for the gear ratio:

1. **Calculate Motor Rotation Distance:**
   ```
   Motor rotation_distance = Extruder rotation_distance / Gear ratio
   Motor rotation_distance = 48.02976 / 9 = 5.33664 mm per motor rotation
   ```

2. **Calculate Motor RPM at Extrusion Speed:**
   ```
   Motor RPM = (Extrusion Speed in mm/s) / (Motor rotation_distance in mm) * 60
   ```

### Current Calculation (Default Speed: 7.0 mm/s)

```
Motor rotation_distance = 48.02976 / 9 = 5.33664 mm/rotation
Motor RPM = (7.0 mm/s) / (5.33664 mm/rotation) * 60
Motor RPM = 1.311 rotations/s * 60
Motor RPM ≈ 78.7 RPM → 79 RPM (rounded)
```

### RPM at Different Speeds

| Extrusion Speed | Motor RPM |
|----------------|-----------|
| 5.0 mm/s       | ~56 RPM   |
| 7.0 mm/s       | ~79 RPM   |
| 10.0 mm/s      | ~112 RPM  |
| 15.0 mm/s      | ~169 RPM  |

**Note**: The motor RPM varies with extrusion speed. The configured value (79 RPM) is based on the default unload/load speed of 7.0 mm/s.

## Motor Synchronization

### Goal
Synchronize buffer and extruder motor speeds during unload operations to:
- Prevent buffer from pulling on filament still engaged in hotend
- Ensure smooth, coordinated retraction
- Optimize timing for simultaneous operations

### Current Implementation
- **Phased simultaneous operation**: Three-phase retraction sequence
  1. **Phase 1**: Initial retraction (20-30mm) to disengage filament from hotend
  2. **Phase 2**: Start buffer retraction after calculated delay
  3. **Phase 3**: Continue remaining hotend retraction while buffer runs simultaneously
- **Dynamic delay calculation**: Delay is calculated based on:
  - Hotend path length (`variable_hotend_path_length`)
  - Extrusion speed
  - Motor acceleration time (0.3s)
  - Safety margin (0.2s)
- **Fallback**: Uses `variable_buffer_retract_delay` as minimum delay if calculated value is lower

### Synchronization Strategy

The system now:
1. Calculates initial retraction distance (25% of hotend path, 20-30mm range)
2. Calculates delay dynamically: `(initial_retract_time + motor_accel_time + safety_margin)`
3. Performs true simultaneous operation: buffer and hotend retract together after initial phase
4. Adapts to different hotend configurations via `variable_hotend_path_length`

## Configuration Variables

### In `mellow_buffer_macros.cfg`

```ini
variable_buffer_motor_rpm: 260                   # Buffer motor speed in RPM
variable_extruder_motor_rpm: 79                  # Extruder motor speed in RPM (at 7.0 mm/s, with gear_ratio 9:1)
variable_hotend_path_length: 100.0              # Distance from nozzle tip to extruder in mm (used for timing calculations)
variable_buffer_retract_delay: 2.0               # Minimum delay (seconds) - actual delay calculated dynamically
```

### Speed Ratio
- **Buffer RPM**: 260 RPM
- **Extruder RPM**: 79 RPM (at 7.0 mm/s)
- **Speed Ratio**: 260 / 79 ≈ **3.29:1** (buffer is ~3.3x faster than extruder)

## Notes

- **Buffer firmware speed changes**: If you modify the `SPEED` variable in buffer firmware, update `variable_buffer_motor_rpm` accordingly
- **Extruder speed varies**: Extruder RPM depends on extrusion speed, which varies during operations
- **Hotend path length**: Measure the distance from your nozzle tip to the extruder entrance and set `variable_hotend_path_length` accordingly
  - This is critical for proper timing calculations
  - Common values: 50-80mm (direct drive), 100-150mm (Bowden)
- **Dynamic timing**: The system automatically calculates optimal delay based on your hotend path and speed
- **Simultaneous operation**: After initial disengagement, buffer and hotend retract simultaneously for efficiency

## References

- **Buffer Firmware Repository**: [Buffer Firmware](https://github.com/patofoto/Buffer)
- **Klipper Extruder Configuration**: [Klipper Extruder Documentation](https://www.klipper3d.org/Config_Reference.html#extruder)
- **TMC2209 Datasheet**: For VACTUAL register details

