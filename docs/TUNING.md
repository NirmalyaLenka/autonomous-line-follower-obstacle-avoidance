# PD tuning guide

Complete `CALIBRATION.md` sections 1 and 2 first. Tune the line follower **without** the
obstacle (remove the box or keep it out of the ToF's range, or the ToF will interfere).
The step sizes below are rules of thumb, not measured values.

Use `d` to see `mask`, `cnt` and `pos` at 10 Hz while it runs.

## Parameters

| Parameter | Value | Role |
|---|---:|---|
| `BASE_SPEED` | 140 | straight-line PWM |
| `MIN_SPEED` | 75 | floor of `base` before the ramp |
| `MAX_REVERSE` | 110 | most the inner wheel may reverse |
| `KP` | 32 | proportional gain |
| `KD` | 70 | derivative gain (per tick) |
| `SLOWDOWN` | 14 | speed lost per unit of `|err|` |
| `START_RAMP_MS` | 400 | soft start |
| `LOOP_US` | 4000 | control period |

## Sequence

**Step 1: low `BASE_SPEED`.** Set `BASE_SPEED` to about 90 to 100, `KD = 0`. The robot should
move slowly enough that you can see what it does. Keep `SLOWDOWN` as it is.

**Step 2: tune `KP`.** Start from about half the current value. Place the robot on a straight
line, slightly off-centre.

**Step 3: raise `KP` until the robot follows curves** but begins to oscillate (a visible
zig-zag at constant frequency). Then back it off by about 20%.

**Step 4: raise `KD`** to damp that oscillation. Increase in steps of about 20% until the
zig-zag disappears and curves are smooth. Stop before the robot becomes slow to
respond or jittery.

**Step 5: raise `BASE_SPEED`** gradually (about 10 PWM steps), re-checking steps 3 to 4 at each
speed, since the right gains change with speed.

**Step 6: tune `SLOWDOWN`.** A higher value slows the robot more when the line is off-centre
(curves); lower is faster but overshoots. Remember that max `|err|` is 3, so the slowest
speed is `BASE_SPEED - 3*SLOWDOWN`.

**Step 7: test hard cases.** Sharp curves, gaps, intersections, the start line (the soft start
helps), the line leaving the array through the edge (the `|g_lastPos| >= 2` pivot).

## Symptoms

| Symptom | Cause | Action |
|---|---|---|
| drifts off the line, slow to correct | `KP` too low | raise `KP` |
| zig-zag with a steady period | `KP` too high | lower `KP` |
| oscillation that decays slowly or grows at curves | `KD` too low | raise `KD` |
| sluggish, jittery or "nervous" response | `KD` too high (sensor quantisation noise) | lower `KD` |
| overshoots and loses the line at curves | speed too high | lower `BASE_SPEED`, raise `SLOWDOWN` |
| wanders on straights | `LEFT_SCALE` / `RIGHT_SCALE` mismatch or one sensor flickering | trim; check `d` |
| spins away at start | steering sign wrong | `CONTROL_ALGORITHM.md` "Reversing the steering direction" |

## Notes

- The error is quantised (steps of 0.5 or 1 position unit), so the derivative term is
  impulsive. That is normal with binary sensors.
- `KD` is per tick; changing `LOOP_US` changes its meaning.
- Re-tune after changing wheels, motors, battery state or sensor height.
- Do not start at `BASE_SPEED = 140` on a new robot. Reach it by steps 5 and 6.
