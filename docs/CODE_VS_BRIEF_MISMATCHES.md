# Code vs. hardware description: recorded mismatches

This repository documents the sketch in `src/main.ino`, which is the uploaded
final sketch. The hardware description given to the documentation project
differs from what the sketch does in several places. None of these were
fixed silently. This page lists each one, what the code really does, and what
to do about it.

Only four constants were edited in `src/main.ino`, each tagged `CHANGED (repo)`:

| Constant | Uploaded value | Value in this repo |
|---|---|---|
| `ROBOT_LENGTH_MM` | 160 | 250 |
| `ROBOT_WIDTH_MM` | 180 | 190 |
| `OBSTACLE_STOP_MM` | 0 | 300 |
| `OBSTACLE_EMERGENCY_MM` | 70 | 250 |

No logic was changed. `GEOMETRY_MEASURED` is still `false`.

---

## M1. IR1 (GPIO14) is defective but the sketch still scans it

**Brief:** IR1 is defective and intentionally unused; six sensors are active.

**Code:** `IR_PIN[7] = {14, 13, 16, 17, 18, 19, 23}`. GPIO14 is configured as
an input in `setup()` and is read in:

- `readSensors()`: it contributes to `g_mask`, `g_cnt` and `g_pos` with weight `-3`.
- `calibrate()`: it takes part in the background vote (`st[0] + st[1] + st[5]`).
- `AV_SEARCH_LINE`: `g_cnt >= LINE_CONFIRM_MIN_SENSORS` counts it as a line hit.
- `followLine()`: the optional finish test uses `g_cnt == 7`. It is disabled
  (`ENABLE_FINISH_STOP = false`), and it could never trigger with a dead sensor.

**Consequences:**

- If the defective IR1 sits constantly at the *floor* level, it is harmless.
- If it sits constantly at the *line* level, or flickers, it will bias the
  position estimate and can make SEARCH_LINE declare "line found" at once.
- The calibration vote becomes a vote among two real sensors plus one dead one.

**What to do:** run the `i` serial command with the robot over both floor and
line and confirm GPIO14 reads a constant floor level. If it does not, apply the
optional patch below. The patch is **not** in `src/main.ino` and is untested.

### Optional patch (not applied, untested)

```cpp
// 1) next to IR_PIN:
static const bool IR_ENABLED[7] = {false, true, true, true, true, true, true}; // IR1 defective

// 2) readSensors(): first line inside the for-loop
for (uint8_t i = 0; i < 7; i++) {
  if (!IR_ENABLED[i]) continue;               // skip the defective sensor
  bool isHigh = (digitalRead(IR_PIN[i]) == HIGH);
  ...

// 3) calibrate(): replace the 3-sensor vote
//    uint8_t votes = st[0] + st[1] + st[5];
//    bool bgA = (votes >= 2);
//    if (st[0] != bgA || st[1] != bgA || st[5] != bgA) Serial.println(...);
bool bgA = st[1];                              // IR2 as the background reference
if (st[1] != st[5])
  Serial.println(F("  WARNING: IR2/IR6 disagree - one may be near the line or badly adjusted"));
```

The weights are not changed by the patch (`IR2 = -2 ... IR7 = +3`).

## M2. Sensor weights and the location of "centre"

**Brief:** conceptual weights `IR2 = -2.5, IR3 = -1.5, IR4 = -0.5, IR5 = +0.5,
IR6 = +1.5, IR7 = +2.5`, centre between IR4 and IR5.

**Code:** `W[7] = {-3, -2, -1, 0, +1, +2, +3}` for IR1..IR7. IR4 has weight 0
and is the reference: `calibrate()` requires IR4 on the line, and
`centerAligned()` requires IR4 active and `|pos| <= 0.5`. The brief itself also
says IR4 (GPIO17) is the centre/reference sensor, which contradicts a centre
between IR4 and IR5.

**This repository documents the code's weights.** With IR1 unused, the
active weights are IR2 = -2, IR3 = -1, IR4 = 0, IR5 = +1, IR6 = +2, IR7 = +3.
The usable range is therefore asymmetric: 2 spacings to the left of IR4 and
3 to the right. See `docs/CONTROL_ALGORITHM.md`.

## M3. Steering sign and derivative scaling

**Brief template:** `V_left = V_base - u`, `V_right = V_base + u`, derivative
`(e_now - e_prev) / dt`.

**Code:**

```
err  = g_pos                 (positive = line to the RIGHT of IR4)
corr = KP*err + KD*g_dFilt
l    = base + corr           (left wheel)
r    = base - corr           (right wheel)
```

So the code uses `V_left = base + corr`, `V_right = base - corr`. The
derivative is the difference per control tick (nominal 4 ms), low-pass
filtered with `g_dFilt = 0.5*g_dFilt + 0.5*dRaw`, and is **not** divided by dt.
`KD = 70` therefore means 70 per unit-of-position-change per tick. In
per-second terms that is `KD * 0.004 = 0.28`.

## M4. No `BLACK_STATE` constant

**Brief:** "invert `BLACK_STATE` or equivalent".

**Code:** polarity is learned at start-up by `calibrate()` into `g_lineHigh[7]`.
There is no compile-time constant. See `docs/CALIBRATION.md` for how to
override it manually.

## M5. Geometry defaults and "measured" status

The uploaded sketch had `ROBOT_LENGTH_MM = 160`, `ROBOT_WIDTH_MM = 180`,
`OBSTACLE_STOP_MM = 0`, `OBSTACLE_EMERGENCY_MM = 70`. With those values the
derived stop distance is 251 mm, D_right 160 mm, D_forward 451 mm. To
reproduce the reference robot (stop = 300 mm) `OBSTACLE_STOP_MM` must be set to
300 explicitly: leaving it at 0 with L = 250, W = 190 gives 288 mm
(`g_minSafeStopMm + STOP_BRAKE_ALLOWANCE_MM`).

The reference dimensions are described as approximate, and
`TOF_SENSOR_TO_ROTATION_CENTER_MM = 90`, `DRIVE_MM_PER_SEC = 200`,
`TURN_RIGHT_MS = TURN_LEFT_MS = 420` are labelled `MEASURE` in the sketch.
`GEOMETRY_MEASURED` stays `false` until the builder verifies all of them
with a ruler; the start-up warning is intentional.

## M6. Meaning of `ROBOT_WIDTH_MM`

The brief calls W the "wheel span/width". In the code the comment defines it as
the **widest extent including the sensor bar**. The sensor bar is 180 mm
wide per the code header. If any part of your robot is wider than the wheel
span, use the larger number: it sets both `R_corner` and `D_right`.

## M7. Limits of the implemented geometry model

These are properties of the formulas in `computeGeometry()`, not edits.

1. **Sign of the ToF offset.** The code computes
   `g_minSafeStopMm = TOF_SENSOR_TO_ROTATION_CENTER_MM + R_corner + SAFETY_MARGIN_MM`.
   The exact condition for rotating without touching the box is
   `d_sensor_to_box + s >= R_corner + M`, where `s` is the signed distance from
   the rotation centre to the sensor (`s > 0` if the sensor is *ahead* of the axle).
   The code's formula is exact if the sensor is **behind** the axle (`s = -90`).
   For the usual front-mounted sensor (`s = +90`) it is conservative: the robot stops
   about `2 x 90 = 180 mm` farther away than strictly necessary (tight value
   `158 + 20 - 90 = 88 mm`). Conservative is safe, but it lengthens the whole detour.
   `ALLOW_STOP_BELOW_CALCULATED_MINIMUM` exists for builders who want the tighter value.
2. **`D_forward` has no sensor-offset term.** `D_forward = D_stop + B + L/2 + M`.
   For a sensor 90 mm ahead of the axle, 545 mm leaves the rotation centre only
   55 mm beyond the box's far face. A worst-case full-disk check (not in the code):
   to rotate at the end of the pass without the swept disk of radius `R_corner + M`
   touching the box's far-right corner, the centre needs about 136 mm of longitudinal
   clearance, which gives roughly 626 mm of travel for a sensor ahead of the axle
   and 446 mm for a sensor behind it. The 545 mm reference lies between the two.
   If the robot clips the box when it turns toward the line after PASS_OBSTACLE,
   increase `FORWARD_PASS_MS` by hand. **USER MUST VERIFY on the track.**
3. **Rotation centre assumed at the middle of the bounding rectangle.**
   `R_corner = sqrt((L/2)^2 + (W/2)^2)` assumes the axle is mid-length. If the
   axle is off-centre, use `sqrt(max(front, rear)^2 + (W/2)^2)` and enter an
   equivalent larger `ROBOT_LENGTH_MM` (`2 * max(front, rear)`).
4. The detour assumes the box is centred on the line and the right side is free.
   Nothing in the code checks either.

## M8. Emergency distance of 250 mm

`OBSTACLE_EMERGENCY_MM = 250` is a reference-configuration value (the sketch
default was 70). `emergencyNow()` halts the robot in SHIFT_RIGHT, PASS_OBSTACLE and
SEARCH_LINE if a valid reading at or below 250 mm persists for 2 readings.
Anything within 250 mm in front of the robot during the detour counts: a wall or an arena
boundary, not only the box. Lower it if that is a problem on your arena.

## M9. Mermaid diagram

The example diagram in the brief draws `BACKUP_TO_SAFE --> TURN_RIGHT`.
In the code BACKUP returns to STOP, which re-checks the distance, and only then
goes to TURN_RIGHT. APPROACH can also return to FOLLOW_LINE (clear or timeout),
and several states can end in HALT. `diagrams/state_machine.md` follows the code.

## M10. Items the project information does not state

Marked `USER MUST MEASURE` or `USER MUST VERIFY` throughout the docs: the
"Total 360" buck converter's exact module, output voltage and current limit;
the battery capacity; the BO motors' rated voltage; the supply voltage and output
levels of the IR modules (ESP32 GPIOs are not 5 V tolerant); the ToF breakout's
supply range; `DRIVE_MM_PER_SEC`; `TURN_RIGHT_MS`; `TURN_LEFT_MS`.

## M11. Gyro

A gyro was available during development. The sketch does not use one. It is
documented as not required.
