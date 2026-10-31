# Calibration

Do these in order. Later steps depend on earlier ones. Every value you measure goes into the
constants at the top of `src/main.ino`. Keep the robot on blocks (wheels off the floor) until
the motor directions are verified.

Serial monitor: 115200 baud. Commands: `h` prints the list.

## 1. Motor direction and trim

1. Command `t`. The test drives LEFT forward, LEFT back, RIGHT forward, RIGHT back, 700 ms each.
2. If a wheel moves backward when it should go forward, set `LEFT_INVERT` or `RIGHT_INVERT` to `true`.
3. If the **left** command moves the right wheel, the A/B motor leads are swapped: swap them at the driver
   (do not try to fix this in software).
4. On the floor, command `p` (straight drive, `AV_DRIVE_SPEED`). If the robot veers, reduce
   the faster side's `LEFT_SCALE` or `RIGHT_SCALE` (for example 0.95). The trim applies to every motion.

Motors may not be matched; the trim is for that. Re-check it when the battery is low.

## 2. IR sensors (hardware first)

The sketch learns *which level means line*. It cannot repair a bad threshold.

1. **Height.** Mount the sensors at a fixed height above the track; the right height depends on your
   modules (**USER MUST MEASURE**: start at the module's datasheet height and adjust until the LED
   switches cleanly). Keep it constant across all sensors and after bumps.
2. **Potentiometer.** On each 3-pin module, turn the pot until its LED changes cleanly between the
   black line and the white floor. IR4 is the 4-pin module: use its DO pin.
3. **Spacing.** 17 mm pitch, 20 mm line: you should see 1 or 2 sensors on the line when aligned.
4. **Order.** Command `i` and move a black strip under the bar from left to right: the `active=` bits
   must change in the order IR1..IR7 (IR1 is unused). If the order is reversed, see
   "Reversing the steering direction" in `CONTROL_ALGORITHM.md`.
5. **Each sensor independently.** Cover each one with the line and the floor; confirm the raw bit
   toggles exactly once. Command `d` while running prints a warning if a sensor toggles more than
   `IR_TOGGLE_WARN_PER_SEC = 80` times per second (noise).
6. **IR1.** With the bar over line and floor, confirm GPIO14 stays at a constant floor level. If
   not, see `CODE_VS_BRIEF_MISMATCHES.md` M1.
7. **Centre.** GPIO17 is IR4, the centre/reference sensor. It must be directly above the centreline
   of the robot.
8. **Polarity.** Put IR4 on the line and IR2/IR6 on the floor, then press `c`. Expect `CALIBRATION OK`
   and a "Line level per sensor" string. The expected log shows IR4 differs from IR2/IR6.
   There is no `BLACK_STATE` constant: if you need to force polarity, edit the initial values of
   `g_lineHigh[]` (`true` means the pin is HIGH on the line) and do not rely on `calibrate()`.

## 3. ToF

1. Wiring: SDA GPIO21, SCL GPIO22. The start-up message must say `VL53L0X OK`.
2. Command `v`: it prints 15 readings over 3 s. Hold a flat card at 100, 200, 300, 400 mm
   (measure with a ruler from the sensor window). Readings should agree within a few mm and
   `fresh=1`.
3. Point the sensor at the real 100 x 100 mm box. Check the box face is high enough to be seen
   (**USER MUST MEASURE** box height versus the sensor's height and field of view).
4. Check the readings are stable at the box's colour and material.

## 4. Geometry

Measure with a ruler and write the numbers in the table. Then set `GEOMETRY_MEASURED = true`.

| Constant | How to measure | Your value |
|---|---|---|
| `ROBOT_LENGTH_MM` | front-most to rear-most point | |
| `ROBOT_WIDTH_MM` | widest extent including sensor bar and wheels | |
| `TOF_SENSOR_TO_ROTATION_CENTER_MM` | sensor window to the wheel-axle line, along the robot | |
| `OBSTACLE_WIDTH_MM` | the actual box | |
| `SAFETY_MARGIN_MM` | your choice (reference 20) | |

If the sensor is behind the axle or the axle is off-centre, read `CODE_VS_BRIEF_MISMATCHES.md` M7.
Press `g`: it prints the derived corner radius, stop distance, trigger, clear, emergency, `D_right`,
`D_forward` and the two times. The reference robot gives:

```
corner radius = 158 mm
min safe turn dist = 268 mm
stop = 300 | trigger = 400 | clear = 440 | emergency = 250
D_right = 165 mm -> RIGHT_SHIFT_MS = 825 (calculated)
D_forward = 545 mm -> FORWARD_PASS_MS = 2725 (calculated)
```

## 5. Drive speed (`DRIVE_MM_PER_SEC`)

Calibrate at the **same PWM value the detour uses**: `AV_DRIVE_SPEED = 100`.

1. Floor the robot on the competition floor surface with a fresh battery.
2. Command `f` (drives `g_rightShiftMs` at `AV_DRIVE_SPEED`) or `p` (drives `g_forwardPassMs`; prefer this,
   the longer run reduces the effect of start-up and braking).
3. Measure the distance travelled (mm). `V_actual = distance_mm / (run_time_ms / 1000)`.
4. Repeat at least 3 times; use the average, or a lower value to be conservative.
5. Put the result in `DRIVE_MM_PER_SEC` and recompute (`g` shows new times).

Example: `p` runs 2725 ms; the robot travels 500 mm: `V = 500 / 2.725 = 183.5 mm/s`. Set
`DRIVE_MM_PER_SEC = 183`, then `D_forward = 545` gives `545000 / 183 = 2978 ms`.

`timedRun()` does not use the soft start, so the first few ms are not representative; the longer
`p` run reduces that error. Battery voltage changes speed: measure with the battery state you race with.

Instead of `DRIVE_MM_PER_SEC` you may type measured times into `RIGHT_SHIFT_MS` and `FORWARD_PASS_MS`
(non-zero values override the calculation).

## 6. Turn times

`TURN_RIGHT_MS` and `TURN_LEFT_MS` (placeholder 420 ms, at `AV_TURN_SPEED = 130`).

1. Draw a reference line on the floor and align the robot's centreline with it.
2. Command `r`. Measure the actual rotation (a printed protractor or a square corner works well).
3. If it turns too far, reduce `TURN_RIGHT_MS`; too little, increase. Repeat until 90 degrees.
4. Do the same for `l` and `TURN_LEFT_MS`.
5. Repeat three times; the result should be repeatable within a few degrees.
6. Re-check after changing `AV_TURN_SPEED`, the battery or the floor.

Left and right can differ because the motors are not matched; that is why there are two
constants and `LEFT_SCALE` / `RIGHT_SCALE`. Change the scales if the robot drifts in straight driving;
change the turn times if only the spin angle is wrong. First guess:
`t_90 = pi * T / (4 v)` (see `MATHEMATICS.md` section 9). `ALIGN_TURN` reuses `TURN_RIGHT_MS`.

## 7. Detour test

1. Put the robot at the stop position: sensor 300 mm from the box face, on the line.
2. Calibrate (`c`), then command `o`. It runs TURN_RIGHT -> ... -> RESUME once and stops.
3. Watch for: right shift clears the box; no contact on the second turn; robot reaches the line;
   IR4 ends centred.
4. If the robot clips the box after PASS_OBSTACLE, increase `FORWARD_PASS_MS`
   (`CODE_VS_BRIEF_MISMATCHES.md` M7).

## Calibration checklist

```
[ ] Motor direction verified
[ ] Left/right motor scaling verified
[ ] IR sensor order verified
[ ] IR polarity verified
[ ] Sensor height verified
[ ] Center sensor verified (GPIO17 = IR4)
[ ] IR1 / GPIO14 verified constant (or patch applied)
[ ] ToF wiring verified
[ ] ToF readings verified
[ ] Robot length measured
[ ] Robot width measured
[ ] ToF-to-rotation-center distance measured
[ ] Obstacle width measured
[ ] Safety margin selected
[ ] Stop distance calculated
[ ] Trigger distance calculated
[ ] Drive velocity measured
[ ] Right shift distance calculated
[ ] Forward pass distance calculated
[ ] Turn time calibrated
[ ] Line reacquisition tested
[ ] Full obstacle test completed
```
