# Mathematics

Every formula used by the firmware or needed to recalibrate it, in one place.
Variable names match `src/main.ino`. "Reference" values are for the reference
robot (L = 250 mm, W = 190 mm). Values marked *placeholder* come from the sketch
and must be measured on your robot.

Formulas are written as plain text so they render everywhere.

---

## 1. Time-of-flight distance

The VL53L0X emits pulsed infrared light and measures the round-trip time of
the reflection:

```
d = c * dt / 2
```

- `d`: distance to the target
- `c`: speed of light, about 299 792 458 m/s
- `dt`: round-trip flight time
- `/ 2`: the light travels out and back

The ESP32 code never computes this. The sensor's internal processing and the
Adafruit library return `RangeMilliMeter` and `RangeStatus`; the sketch only
validates the result (`RangeStatus != 4` and 15 mm to 1500 mm) and compares it
with thresholds.

## 2. Line position

Each sensor `i` gives `s_i = 1` if it sees the line, else 0. With weights `w_i`
(code: `W[]`, IR1..IR7 = -3, -2, -1, 0, +1, +2, +3):

```
pos = sum(w_i * s_i) / sum(s_i)          (only if sum(s_i) > 0)
```

Active sensors on the reference robot (IR1 unused): IR2 = -2, IR3 = -1,
IR4 = 0, IR5 = +1, IR6 = +2, IR7 = +3. With a 17 mm pitch one position unit is
17 mm. If the line is under IR4 only, `pos = 0`; under IR4 and IR5, `pos = +0.5`.

Error (code): `err = pos`. The target is 0, positive means the line is to the right.

## 3. PD control

Continuous form:

```
u(t) = Kp * e(t) + Kd * de/dt
```

Code (discrete, per control tick):

```
dRaw   = err - g_lastErr
g_dFilt = 0.5 * g_dFilt + 0.5 * dRaw
corr   = KP * err + KD * g_dFilt          KP = 32, KD = 70
```

The derivative is a per-tick difference. To convert to continuous units use
`Kd_continuous = KD * dt = 70 * 0.004 s = 0.28`.

Filter: `y[n] = 0.5 y[n-1] + 0.5 x[n]` has a time constant of about
`-dt / ln(0.5) = 5.8 ms`. The sum of the filtered samples equals the sum of
`dRaw`, so a step change `Δe` contributes a total of `KD * Δe` regardless of
the filter. The filter only spreads it over several ticks.

Control period: `LOOP_US = 4000 us = 4 ms`, so `f = 1 / 0.004 = 250 Hz`.

## 4. Differential drive and saturation

```
base  = max(BASE_SPEED - SLOWDOWN*|err|, MIN_SPEED)
base  = base * ramp                      (start ramp)
base  = min(base, cap)                   (approach / post-detour caps)
V_left  = base + corr
V_right = base - corr
V = constrain(V, -MAX_REVERSE, 255)      MAX_REVERSE = 110
```

Start ramp: `ramp = 0.5 + 0.5 * t / START_RAMP_MS` for `t < 400 ms`, then 1.0.
It scales `base` only, not `corr`.

Steady state (derivative = 0, ramp = 1, no cap):

| err | base | V_left | V_right |
|---:|---:|---:|---:|
| 0 | 140 | 140 | 140 |
| +0.5 | 133 | 149 | 117 |
| +1 | 126 | 158 | 94 |
| +2 | 112 | 176 | 48 |
| +3 | 98 | 194 | 2 |
| -2 | 112 | 48 | 176 |

`MIN_SPEED = 75` is never reached by the proportional term because
`140 - 14 * 3 = 98 > 75`. Wheel reversal comes from derivative kicks: a jump
from 0 to +3 in one tick gives `dFilt = 1.5`, `corr = 96 + 105 = 201`,
`V_right = 98 - 201 = -103`, which is inside the `-110` limit.

Why PWM: the TB6612FNG switches the motor supply on and off. The average motor
voltage is approximately `duty/255 * V_supply` (minus driver drops). The sketch
uses 20 kHz, 8-bit resolution (`PWM_FREQ`, `PWM_BITS`), so duty `0..255` maps to the
speed values used everywhere.

## 5. Safety geometry

```
R_corner = ceil( sqrt( (L/2)^2 + (W/2)^2 ) )
```

Reference: `sqrt(125^2 + 95^2) = sqrt(24650) = 157.003`, and `ceilf()` gives 158 mm.

```
D_min_safe = TOF_SENSOR_TO_ROTATION_CENTER_MM + R_corner + SAFETY_MARGIN_MM
           = 90 + 158 + 20 = 268 mm            (g_minSafeStopMm, g_turnMinMm)
```

```
g_stopEffMm    = max(OBSTACLE_STOP_MM, D_min_safe + STOP_BRAKE_ALLOWANCE_MM)
               = max(300, 268 + 20) = 300 mm
g_triggerEffMm = max(OBSTACLE_TRIGGER_MM, g_stopEffMm + MIN_APPROACH_ZONE_MM)
               = max(350, 400) = 400 mm
g_clearEffMm   = max(OBSTACLE_CLEAR_MM, g_triggerEffMm + 40)
               = max(400, 440) = 440 mm
g_emergencyEffMm = OBSTACLE_EMERGENCY_MM = 250 mm
                   (forced to g_stopEffMm - 20 if it is >= g_stopEffMm)
```

Ordering that must hold: `clear > trigger > stop > emergency`.
`g_turnMinMm = 268 < g_stopEffMm = 300`, so a normal stop always allows the turn.

Sign caveat for the sensor offset: see `docs/CODE_VS_BRIEF_MISMATCHES.md` M7.
With `s` = signed distance from rotation centre to sensor (+ if ahead):

```
exact:  d_sensor_to_box >= R_corner + M - s
```

## 6. Detour geometry

```
D_right   = OBSTACLE_WIDTH_MM/2 + ROBOT_WIDTH_MM/2 + SAFETY_MARGIN_MM
          = 50 + 95 + 20 = 165 mm
D_forward = g_stopEffMm + OBSTACLE_WIDTH_MM + ROBOT_LENGTH_MM/2 + SAFETY_MARGIN_MM
          = 300 + 100 + 125 + 20 = 545 mm
```

All divisions are integer divisions in C (the reference values are even, so
there is no rounding). `D_forward` is a first-order estimate: see M7.

Worst-case cross-check (not in the code): the centre must end up
`sqrt((R_corner+M)^2 - (D_right - B/2)^2) = sqrt(178^2 - 115^2) = 135.9 mm`
past the box's far face to rotate freely. Travel needed = (centre-to-front-face) + B + 136, where
centre-to-front-face = `d_stop + s`. Sensor 90 mm ahead: `390 + 100 + 136 = 626 mm`.
Sensor 90 mm behind: `210 + 100 + 136 = 446 mm`.

## 7. Distance to time

```
t_ms = D_mm / V_mm_per_s * 1000
```

Code: `g_rightShiftMs = g_dRightMm * 1000 / DRIVE_MM_PER_SEC` (integer division),
and likewise for `g_forwardPassMs`. With `DRIVE_MM_PER_SEC = 200`:

```
shift = 165 / 200 * 1000 = 825 ms
pass  = 545 / 200 * 1000 = 2725 ms
```

Other speeds for the same geometry: 300 mm/s gives 550 ms and 1816 ms;
150 mm/s gives 1100 ms and 3633 ms. `DRIVE_MM_PER_SEC` is a calibration value
for the specific robot at `AV_DRIVE_SPEED = 100`. Measure it.

If `RIGHT_SHIFT_MS` or `FORWARD_PASS_MS` is non-zero, it replaces the computed time.

## 8. Wheels and motors (for first estimates)

```
C = pi * D                    distance per wheel revolution
V = RPM * pi * D / 60         (D in m gives m/s)
```

Illustrative only (not this robot): D = 65 mm, 100 RPM gives
`100 * pi * 0.065 / 60 = 0.340 m/s = 340 mm/s`. Real loaded speed is lower and
nonlinear in PWM. Always replace the estimate with a measurement.

## 9. In-place turn (first estimate only)

For an in-place spin with both wheels at linear speed `v` and track width `T`
(wheel-to-wheel distance):

```
omega = 2 v / T
t_90  = (pi/2) / omega = pi * T / (4 v)
```

Example with `T = 190 mm`, `v = 260 mm/s`: `t_90 = 0.574 s`. Slip, motor
mismatch and floor friction make this unreliable; use it to choose a first
guess for `TURN_RIGHT_MS` / `TURN_LEFT_MS` and then calibrate (see
`docs/CALIBRATION.md`).

## 10. Confirmation counters and timing

ToF readings arrive at the sensor's measurement rate (the sketch does not
change the library's default timing budget; check it with the `v` command).
A counter that must reach N therefore takes about `N` measurement periods.
Stale limit: `TOF_STALE_MS = 400 ms`.

## 11. Search sweeps (time schedule)

`sweepDir(t, firstDir, base)` has legs of length `k*base` out and `2*k*base`
back. Lost-line search (`base = 200`, `firstDir = g_lastDir`), time `t`
measured from `GAP_CROSS_MS`:

| t range (ms) | direction |
|---|---|
| 0 to 200 | first |
| 200 to 600 | opposite |
| 600 to 1200 | first |
| 1200 to 2000 | opposite |
| 2000 to 3000 | first |
| 3000 to 4200 | opposite (cut at 3500 by `SEARCH_TIMEOUT_MS`) |

Failsafe sweep (`base = 150`, first = right), limited by `FAILSAFE_SEARCH_MS = 5000`:

| t range (ms) | direction |
|---|---|
| 0 to 150 | right |
| 150 to 450 | left |
| 450 to 900 | right |
| 900 to 1500 | left |
| 1500 to 2250 | right |
| 2250 to 3150 | left |
| 3150 to 4200 | right |
| 4200 to 5000 | left (cut at 5000) |

## Summary of key equations

```
R_corner   = sqrt((L/2)^2 + (W/2)^2)
D_min      = D_sensor_offset + R_corner + D_safety
D_right    = W_obstacle/2 + W_robot/2 + D_safety
D_forward  = D_stop + W_obstacle + L_robot/2 + D_safety
t_ms       = D_mm / V_mm_per_s * 1000
u          = Kp*e + Kd*(delta e)/(delta t)
V_L        = V_base + u      (code convention)
V_R        = V_base - u
```
