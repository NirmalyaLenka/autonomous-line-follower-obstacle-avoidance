# Control algorithm: line following

Implemented by `readSensors()`, `calibrate()` and `followLine()` in `src/main.ino`.
Equations are collected in `MATHEMATICS.md`.

## 1. Sensor array

The array is 7 positions on a 17 mm pitch (positions measured from the left edge of the
180 mm bar, from the sketch header: IR1 = 39, IR2 = 56, IR3 = 73, **IR4 = 90 (centre)**,
IR5 = 107, IR6 = 124, IR7 = 141 mm). **IR1 (GPIO14) is defective and not used on the
reference robot**, so six sensors are active (IR2..IR7). The sketch still scans GPIO14: see
`CODE_VS_BRIEF_MISMATCHES.md` M1.

With a 20 mm line and a 17 mm pitch a correctly aligned robot sees one or two sensors.

Each sensor is a digital module, so it reports only line / no line. `readSensors()`
compares each pin with the level that means "line" for that sensor (`g_lineHigh[i]`)
and builds:

- `g_mask`: bit `i` set if sensor `i` sees the line
- `g_cnt`: number of sensors seeing the line
- `g_pos`: weighted average position (kept from the last valid reading if `g_cnt == 0`)

## 2. Weights

```
W[7] = { -3, -2, -1, 0, +1, +2, +3 }        // IR1 .. IR7
```

| Sensor | GPIO | Weight | Status |
|---|---|---:|---|
| IR1 | 14 | -3 | **defective, unused** (still scanned by the sketch) |
| IR2 | 13 | -2 | active |
| IR3 | 16 | -1 | active |
| IR4 | 17 | 0 | active, **centre / reference** |
| IR5 | 18 | +1 | active |
| IR6 | 19 | +2 | active |
| IR7 | 23 | +3 | active |

Positive position means the line is to the right of IR4. Because IR1 is out, the usable
range is asymmetric: `-2` on the left, `+3` on the right.

`pos = sum(w_i * s_i) / sum(s_i)`. Examples: IR4 only gives 0; IR4+IR5 gives +0.5;
IR3+IR4+IR5 gives 0; IR7 only gives +3.

### Reversing the steering direction

If the robot steers away from the line instead of toward it, the array's left/right
orientation does not match the motor mapping. Fix it in one place only:

1. Reverse the order of `IR_PIN[]` so that the first entry is the sensor that is physically
   on the **left**, or
2. Swap which motor is wired to the "A" and "B" channels (and re-check `LEFT_INVERT` /
   `RIGHT_INVERT`).

Do **not** flip only the sign of `W[]`, `KP` or the `l`/`r` formulas. The lost-line pivot
(`g_lastDir`), the sharp-edge test (`|g_lastPos| >= 2`) and `ALIGN_FINE` all use the same
convention (positive = right = left wheel forward, right wheel backward), and they would
then turn the wrong way while the PD loop turns the right way.

## 3. Polarity calibration

`calibrate()` runs at start (and with the `c` command), with the robot **stationary, IR4 on
the line, IR2 and IR6 on floor**:

1. It samples all pins for 300 ms and decides each pin's level by majority.
2. Background level of the 3-pin module family = majority of IR1, IR2, IR6.
3. If IR4 reads the same as the background, calibration **fails** (LED error blink, no start).
4. `g_lineHigh` for IR1..IR3, IR5, IR6 = opposite of the background; IR4 = its own reading
   (it is a different, 4-pin module); IR7 = opposite of its own floor reading.
5. It warns when a sensor is unstable (HIGH between 10% and 90% of samples).

Calibration only learns which level means "line". It cannot repair a bad potentiometer
setting. Adjust each module's pot first (see `CALIBRATION.md`). To override polarity by
hand, edit the initial values of `g_lineHigh[]` and skip `calibrate()`.

## 4. PD controller

```
err   = g_pos
dRaw  = err - g_lastErr
g_dFilt = 0.5*g_dFilt + 0.5*dRaw
corr  = KP*err + KD*g_dFilt
```

`KP = 32.0`, `KD = 70.0`.

**Why PD.** P reacts to the present lateral error. D reacts to how fast the error changes,
which damps oscillation at curves and after corrections. The integral term is omitted:
there is no constant disturbance to cancel on a flat track, and an integrator would wind
up during line loss and during the detour.

**Binary sensors make the error coarse.** The error moves in steps of 0.5 or 1 position
unit, so the derivative is impulsive. A step of 1 in one tick gives `dRaw = 1`, then
`dFilt = 0.5, 0.25, ...`. The D term contributes `70 * 0.5 = 35` on the first tick, about the same
size as the P term (`32`). The total effect of a step of size `Δe` is `KD * Δe` whatever the
filter does.

**Effect of changing gains**

| Change | Effect |
|---|---|
| Kp up | stronger, faster correction; too high gives zig-zag oscillation |
| Kp down | robot drifts toward the outside of curves and loses the line |
| Kd up | more damping, smoother correction; too high gives sluggish response and noise sensitivity |
| Kd down | overshoot and oscillation |

`KD` is in per-tick units (see `MATHEMATICS.md` section 3). If you change `LOOP_US`, re-tune `KD`.

## 5. Speed profile

```
base = BASE_SPEED - SLOWDOWN*|err|          BASE_SPEED = 140, SLOWDOWN = 14
base = max(base, MIN_SPEED)                 MIN_SPEED = 75
base = base * ramp
base = min(base, cap)
```

- `BASE_SPEED`: straight-line speed on the 0..255 PWM scale.
- `SLOWDOWN`: speed lost per unit of `|err|` (140 at centre, 98 at `|err| = 3`).
- `MIN_SPEED`: floor for the speed before the ramp. With max `|err| = 3` the formula never
  gets below 98, so it only matters if you raise `SLOWDOWN`.
- `MAX_REVERSE = 110`: the inner wheel may go this far backward. In steady state the inner
  wheel stays positive; reversals come from D kicks.
- Start ramp: `ramp = 0.5 + 0.5*t/START_RAMP_MS` for the first `START_RAMP_MS = 400 ms`
  after start or after a detour (scales `base` only).
- `cap = g_speedCap` (90 while approaching an obstacle, otherwise 255) and, for
  `RESUME_SLOW_MS = 600 ms` after a detour, at most `RESUME_SPEED = 100`.

Wheel commands: `l = base + corr`, `r = base - corr`, each constrained to `[-110, 255]`
(see the steady-state table in `MATHEMATICS.md` section 4).

## 6. Lost line

When `g_cnt == 0`, `g_lostFlag` is set and `lost = now - g_lastSeen`:

| `lost` | Behaviour |
|---|---|
| `< GAP_CROSS_MS` (180 ms) and `|g_lastPos| >= 2` | the line left through the edge: pivot toward `g_lastDir` at `SEARCH_SPEED` (120) |
| `< GAP_CROSS_MS` and `|g_lastPos| < 2` | probably a gap: drive straight at `GAP_SPEED` (100) |
| `>= GAP_CROSS_MS` | widening in-place sweep (`sweepDir`, `SEARCH_SWEEP_BASE_MS = 200`), first toward `g_lastDir`, at `SEARCH_SPEED` |
| sweep time `> SEARCH_TIMEOUT_MS` (3500 ms) | `haltRobot("line lost - end of track")` |

Total time from line loss to halt: `180 + 3500 = 3680 ms`. The sweep schedule is in
`MATHEMATICS.md` section 11.

**Why a gap should not trigger a search immediately.** Dashed or broken lines, a reflective
patch, or a brief sensor dropout would send the robot into a spin and it would turn away
from a line that resumes 30 mm ahead. Driving straight for 180 ms covers short gaps at
the cost of about 36 mm of travel if 100 PWM corresponds to 200 mm/s (reference value, USER
MUST MEASURE). The edge case `|g_lastPos| >= 2` is different: the line was last seen at the
edge of the array, which means a sharp turn, so the robot pivots instead of crossing.

**Resuming.** When any sensor sees the line again, `g_lastErr = err` and `g_dFilt = 0` are
reset so no derivative kick is produced, and normal PD following continues.

## 7. Finish bar (disabled)

`ENABLE_FINISH_STOP = false`. If enabled, the robot halts when `g_cnt == 7` for
`FINISH_HOLD_MS` after `FINISH_IGNORE_MS`. With IR1 defective `g_cnt == 7` can never be
true; change the test to the number of working sensors before enabling it.
End of track by line loss is a separate mechanism.
