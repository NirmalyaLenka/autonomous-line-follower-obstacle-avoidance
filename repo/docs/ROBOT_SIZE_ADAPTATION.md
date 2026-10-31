# Adapting the software to a different robot

## How to recalculate the robot for your own size

The reference values (L = 250 mm, W = 190 mm, ...) are **not universal**. If your chassis,
motors, wheels or battery differ, work through this page from top to bottom.

1. Measure the geometry (table below). Edit the constants in `src/main.ino`.
2. Press `g`: it prints every derived distance.
3. Choose `OBSTACLE_STOP_MM` (or leave 0 for the calculated value).
4. Measure `DRIVE_MM_PER_SEC` at `AV_DRIVE_SPEED` on your floor (`CALIBRATION.md` section 5).
5. Calibrate the turn times (`CALIBRATION.md` section 6).
6. Run the detour test `o`, then the full course.

**Calculate the required physical distance first. Then convert it into a time using the
measured travel velocity of your robot.** The shipped times (825 ms, 2725 ms) are only
`distance / velocity` for the reference robot.

## Formulas

```
R_corner   = ceil( sqrt( (ROBOT_LENGTH_MM/2)^2 + (ROBOT_WIDTH_MM/2)^2 ) )
D_min_safe = TOF_SENSOR_TO_ROTATION_CENTER_MM + R_corner + SAFETY_MARGIN_MM
g_stopEffMm = max(OBSTACLE_STOP_MM, D_min_safe + STOP_BRAKE_ALLOWANCE_MM)
g_triggerEffMm = max(OBSTACLE_TRIGGER_MM, g_stopEffMm + MIN_APPROACH_ZONE_MM)
g_clearEffMm   = max(OBSTACLE_CLEAR_MM, g_triggerEffMm + 40)
D_right    = OBSTACLE_WIDTH_MM/2 + ROBOT_WIDTH_MM/2 + SAFETY_MARGIN_MM
D_forward  = g_stopEffMm + OBSTACLE_WIDTH_MM + ROBOT_LENGTH_MM/2 + SAFETY_MARGIN_MM
RIGHT_SHIFT_MS  = D_right   / DRIVE_MM_PER_SEC * 1000
FORWARD_PASS_MS = D_forward / DRIVE_MM_PER_SEC * 1000
```

The two limits of the model (sign of the sensor offset, no offset term in `D_forward`) are in
`CODE_VS_BRIEF_MISMATCHES.md` M7. Read it before trusting the numbers.

## Parameter table

| Parameter | Meaning | Current value | What happens if changed | How to measure |
|---|---|---:|---|---|
| `ROBOT_LENGTH_MM` | longest extent, front to rear | 250 | longer: larger `R_corner`, longer stop, longer pass | ruler, front-most to rear-most point |
| `ROBOT_WIDTH_MM` | widest extent incl. sensor bar | 190 | wider: larger `R_corner` and `D_right` | ruler, widest point (wheels or bar) |
| `TOF_SENSOR_TO_ROTATION_CENTER_MM` | sensor to axle line | 90 | enters `D_min_safe` additively | ruler along the robot's centreline |
| `OBSTACLE_WIDTH_MM` | box width (also used for depth) | 100 | wider box: bigger `D_right`, `D_forward` | ruler on the actual box |
| `SAFETY_MARGIN_MM` | clearance added everywhere | 20 | more margin: safer, slower detour | your choice; 20 is the reference |
| `DRIVE_MM_PER_SEC` | speed at `AV_DRIVE_SPEED` | 200 (placeholder) | scales both detour times inversely | measure (`CALIBRATION.md` section 5) |
| `OBSTACLE_STOP_MM` | requested stop distance (0 = calculated) | 300 | raised to the calculated floor if lower | choose >= `D_min_safe + STOP_BRAKE_ALLOWANCE_MM` |
| `OBSTACLE_TRIGGER_MM` | slow-approach start | 350 (effective 400) | raised to stop + 100 if lower | choose >= stop + `MIN_APPROACH_ZONE_MM` |
| `OBSTACLE_CLEAR_MM` | obstacle-gone threshold | 400 (effective 440) | raised to trigger + 40 if lower | choose above trigger |
| `OBSTACLE_EMERGENCY_MM` | dangerously close | 250 (default was 70) | halts in SHIFT/PASS/SEARCH; forced to stop-20 if >= stop | choose below stop; mind walls (M8) |

## Effect of changing the dimensions

| Quantity | Effect |
|---|---|
| Turning envelope | `R_corner` grows with both L and W |
| Stopping distance | `D_min_safe` and `g_stopEffMm` grow with `R_corner` and the sensor offset |
| Lateral clearance | `D_right` grows with W and with the box width |
| Forward pass | `D_forward` grows with the stop distance and L/2 |
| Detection timing | larger stop and trigger distances mean the sensor sees the box earlier and the approach zone is longer |
| Reacquisition | a longer detour means more cumulated drift error in a time-based path; line search starts from further away |
| Reliability | all detour moves are open loop, so errors grow with distance. Shorter robots give shorter detours |

## Worked example: a 300 x 220 mm robot (same sensor offset 90, same box, same 200 mm/s)

```
R_corner    = ceil(sqrt(150^2 + 110^2)) = ceil(186.01) = 187 mm
D_min_safe  = 90 + 187 + 20 = 297 mm
g_stopEffMm = max(300, 297 + 20) = 317 mm    (the requested 300 is raised)
g_triggerEffMm = max(350, 417) = 417 mm
g_clearEffMm   = 457 mm
D_right   = 50 + 110 + 20 = 180 mm  ->  900 ms
D_forward = 317 + 100 + 150 + 20 = 587 mm -> 2935 ms
```

The turn times and `DRIVE_MM_PER_SEC` would also have to be re-measured.

## Faster motors / different wheels / different battery

The detour times depend on the real speed at `AV_DRIVE_SPEED`, so re-measure it
whenever any of these change.

| Change | Effect |
|---|---|
| Wheel diameter | distance per revolution `C = pi * D`; speed `V = RPM * pi * D / 60` (D in metres gives m/s) |
| Gear ratio | changes RPM and torque; changes speed and turn times |
| Motor type | different speed per PWM; deadband changes minimum usable PWM |
| Battery | speed falls as voltage falls; calibrate at your racing battery state |
| PWM frequency | `PWM_FREQ` (20 kHz); the TB6612FNG accepts higher, but changing it changes motor behaviour at low duty |
| Motor driver | other drivers have different brake/standby behaviour; `motorsStop(true)` assumes TB6612 short brake |

Examples with the reference geometry (165 mm shift, 545 mm pass): at 300 mm/s the times
are 550 ms and 1816 ms; at 150 mm/s they are 1100 ms and 3633 ms. Faster travel also lengthens
the braking distance (`STOP_BRAKE_ALLOWANCE_MM = 20` is empirical for the reference robot: recheck it), and
changes the turn times and the line-following gains.

The formula `V = RPM * pi * D / 60` is a first guess only: loaded RPM is lower than no-load RPM and
PWM speed is not linear. Measure.

## Checklist after changing the robot

```
[ ] geometry measured, constants edited, GEOMETRY_MEASURED = true
[ ] g output reviewed
[ ] DRIVE_MM_PER_SEC re-measured
[ ] TURN_RIGHT_MS / TURN_LEFT_MS recalibrated
[ ] Kp / Kd / BASE_SPEED re-tuned
[ ] detour test (o) passed three times in a row
[ ] full-track test passed
```
