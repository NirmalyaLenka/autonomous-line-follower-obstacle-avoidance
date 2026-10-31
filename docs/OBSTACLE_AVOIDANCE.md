# Obstacle avoidance

Implemented by `tofTask()`, `computeGeometry()`, `runStep()`, `avGoto()` and `avStep()`.
Diagram: `diagrams/state_machine.md`. Geometry sketches: `diagrams/obstacle_geometry.md`.
Equations: `MATHEMATICS.md`.

The robot always passes the obstacle on its **right**. The design assumes one box of
about 100 x 100 mm (`OBSTACLE_WIDTH_MM`), roughly centred on the line, with free space to
the right. Nothing in the code checks either assumption.

## 1. ToF sensing

The VL53L0X-compatible sensor measures flight time internally (`d = c * dt / 2`); the code
receives the processed distance in mm. A task pinned to core 0 (`tofTask`) calls
`lox.rangingTest()` continuously and publishes `t_dist`, `t_time` and four counters inside a
critical section. `loop()` (core 1) copies them with `tofRefresh()` once per control tick,
so one tick uses one consistent snapshot.

**Validity.** A reading is used only if `RangeStatus != 4` and it lies in
`[TOF_MIN_VALID_MM = 15, TOF_MAX_VALID_MM = 1500]`. Invalid readings are ignored: they do
not change any counter and do not refresh `t_time`.

**Staleness.** `tofFresh()` is true only if the sensor initialised (`g_tofOk`) and the last
valid reading is younger than `TOF_STALE_MS = 400`. If the sensor was not found at start,
`tofFresh()` is always false: obstacle avoidance is disabled and line following continues.

## 2. Thresholds (reference robot)

| Name | Variable | Value | Meaning |
|---|---|---:|---|
| Minimum safe turning distance | `g_minSafeStopMm` / `g_turnMinMm` | 268 mm | `90 + 158 + 20` |
| Stop | `g_stopEffMm` | 300 mm | forward motion stops here |
| Trigger | `g_triggerEffMm` | 400 mm | start slow approach |
| Clear | `g_clearEffMm` | 440 mm | obstacle considered gone |
| Emergency | `g_emergencyEffMm` | 250 mm | dangerously close |

Derived by `computeGeometry()`:

```
g_stopEffMm    = max(OBSTACLE_STOP_MM, g_minSafeStopMm + STOP_BRAKE_ALLOWANCE_MM)
g_triggerEffMm = max(OBSTACLE_TRIGGER_MM, g_stopEffMm + MIN_APPROACH_ZONE_MM)
g_clearEffMm   = max(OBSTACLE_CLEAR_MM, g_triggerEffMm + 40)
```

`OBSTACLE_STOP_MM` below the calculated floor is silently raised unless
`ALLOW_STOP_BELOW_CALCULATED_MINIMUM = true`. The reference value 300 is above the
floor (288), so it is used as given. Reference:

- **Trigger** starts the slow approach. `D_trigger > D_stop` leaves at least
  `MIN_APPROACH_ZONE_MM = 100 mm` to slow to `APPROACH_SPEED = 90` and brake before
  the stop distance. Without that zone the robot would arrive at full speed.
- **Stop** is where forward motion ends. `STOP_BRAKE_ALLOWANCE_MM = 20` is the overshoot
  allowance when braking from `APPROACH_SPEED` (empirical: measure it).
- **Emergency** is checked first in several states (see below).
- **Clear** lies above trigger (hysteresis) so a reading that hovers near the trigger does not
  toggle between approach and follow.

Why 268 and not 100: the robot is not a point. When it rotates, its corners sweep a circle of
radius `R_corner = 158 mm` around the rotation centre. The sensor is not at the rotation
centre either, so its offset enters the calculation. A fixed "stop 100 mm before the
obstacle" is only meaningful relative to the robot's own geometry. See
`CODE_VS_BRIEF_MISMATCHES.md` M7 for the sign convention and limits of this formula.

## 3. Confirmation counts

A single ToF reading is not trusted: surface reflectivity, incidence angle, sensor noise,
box shape and ambient light all make individual readings fluctuate. Each counter counts
**consecutive valid** readings and resets when one reading fails the condition.

| Constant | Value | Condition | Use |
|---|---:|---|---|
| `OBSTACLE_CONFIRM_COUNT` | 3 | reading `<= trigger` | FOLLOW -> APPROACH |
| `OBSTACLE_STOP_CONFIRM` | 2 | reading `<= stop` | APPROACH -> STOP |
| `OBSTACLE_CLEAR_CONFIRM` | 4 | reading `> clear` | APPROACH -> FOLLOW (false alarm) |
| `OBSTACLE_EMERGENCY_CONFIRM` | 2 | reading `<= emergency` | emergency |

The counters saturate at 250 and are reset (`tofResetCounters()`) on every state change
except into APPROACH, so old readings cannot trigger a new decision.

## 4. State machine

Every state except FOLLOW_LINE and APPROACH_OBSTACLE is entered with a brake pause of
`AV_SETTLE_MS = 150 ms`: motors are shorted (TB6612 short brake) and `avStep()` waits.
During that pause no checks run.

| State | What it does | Leaves when |
|---|---|---|
| `FOLLOW_LINE` | PD line following at full speed. | `emergencyNow()`, or past the ignore window and `obstacleAhead()` (3 readings <= trigger) -> APPROACH. |
| `APPROACH_OBSTACLE` | Same follower, speed capped at `APPROACH_SPEED = 90`. If ToF is not fresh: brake and wait. | emergency or `g_stopCnt >= 2` -> STOP; `g_clearCnt >= 4` -> FOLLOW (false alarm); 5000 ms (`APPROACH_TIMEOUT_MS`) -> FOLLOW and ignore obstacles for 500 ms; ToF stale > 600 ms -> HALT; line lost -> HALT via the follower. |
| `STOP_FOR_OBSTACLE` | Braked. Reads `g_dist`. | stale -> HALT; `g_dist < g_turnMinMm` (268) -> BACKUP; otherwise counts the obstacle and -> TURN_RIGHT. |
| `BACKUP_TO_SAFE` | Reverse at `AV_BACKUP_SPEED = 90`. | `g_dist >= g_stopEffMm` -> back to STOP (which re-checks); `BACKUP_MAX_MS = 1500` -> HALT; stale -> HALT. |
| `TURN_RIGHT` | In-place spin right at `AV_TURN_SPEED = 130`. | `TURN_RIGHT_MS` (420, placeholder) -> SHIFT_RIGHT. |
| `SHIFT_RIGHT` | Drive straight at `AV_DRIVE_SPEED = 100` (sideways relative to the line). | `g_rightShiftMs` (825) -> TURN_FORWARD; emergency -> HALT. |
| `TURN_FORWARD` | In-place spin left, back to the original heading. | `TURN_LEFT_MS` -> PASS_OBSTACLE. |
| `PASS_OBSTACLE` | Drive straight past the box. | `g_forwardPassMs` (2725) -> TURN_TO_LINE; emergency -> HALT. |
| `TURN_TO_LINE` | In-place spin left, now facing the line. | `TURN_LEFT_MS` -> SEARCH_LINE. |
| `SEARCH_LINE` | Creep at `LINE_SEARCH_SPEED = 80`, scanning all sensors. | `g_cnt >= LINE_CONFIRM_MIN_SENSORS (1)` for `LINE_CONFIRM_READINGS (5)` consecutive ticks -> ALIGN_CREEP; `LINE_SEARCH_TIMEOUT_MS = 4000` -> FAILSAFE_SEARCH; emergency -> HALT. |
| `ALIGN_CREEP` | Creep on for `ALIGN_CREEP_MS = 140` so the axle is over the line. | time -> ALIGN_TURN. |
| `ALIGN_TURN` | Spin right for `TURN_RIGHT_MS * ALIGN_TURN_PERCENT / 100` (= 420 ms). | time -> ALIGN_FINE. |
| `ALIGN_FINE` | Slow spin (`ALIGN_FINE_SPEED = 90`) until `centerAligned()`; direction from the sign of `g_pos`, or right when nothing useful is seen. | aligned for `CENTER_CONFIRM_READINGS = 5` ticks -> RESUME; `ALIGN_TIMEOUT_MS = 2500` -> FAILSAFE_SEARCH. |
| `FAILSAFE_SEARCH` | Stopped for the settle time, then widening in-place sweep at `FAILSAFE_SEARCH_SPEED = 90` (first to the right). | aligned for 5 ticks -> RESUME; `FAILSAFE_SEARCH_MS = 5000` -> HALT. |
| `RESUME` | Resets PD history, restarts the start ramp, caps speed at `RESUME_SPEED = 100` for 600 ms, ignores obstacles for 100 ms. | immediately -> FOLLOW_LINE. |

`centerAligned()` is true when IR4 sees the line, `g_cnt <= ALIGN_MAX_ACTIVE_SENSORS (3)`
and `|g_pos| <= CENTER_POSITION_TOLERANCE (0.5)`. "Line found" (any sensor) and "aligned"
(centred, bar across the line) are deliberately different tests: when the robot reaches the
line while facing it, the bar lies **along** the line and many sensors fire at once; only
after the right turn does the bar cross the line.

### Path of the detour

Robot starts heading north on a north-south line, box centred on the line:

1. TURN_RIGHT: now facing east.
2. SHIFT_RIGHT: east `D_right` = 165 mm.
3. TURN_FORWARD: facing north again, 165 mm east of the line.
4. PASS_OBSTACLE: north `D_forward` = 545 mm.
5. TURN_TO_LINE: facing west.
6. SEARCH_LINE: west until the line is seen; ALIGN_CREEP rolls on 140 ms.
7. ALIGN_TURN: right turn to north; ALIGN_FINE centres IR4 on the line; RESUME.

## 5. Why a state machine and not delay() sequences

`runStep()` runs every `LOOP_US = 4000 us` (250 Hz) and every state is a short, non-blocking
function. Sensors and ToF keep updating, the emergency check stays alive in the states
that have it, state transitions are explicit and visible on the serial port (`AV -> NAME`),
and each timing constant can be changed without touching the others. Only the serial test
commands, start-up calibration and the 1.5 s start countdown use `delay()`.

## 6. Failsafes implemented

| Mechanism | Behaviour |
|---|---|
| ToF not found at boot | warning printed; obstacle avoidance disabled; line following continues |
| Invalid / out-of-range reading | ignored |
| Stale ToF | APPROACH: brake, HALT after 600 ms; STOP, BACKUP: HALT immediately |
| Confirmation counts | see section 3 |
| Emergency distance | FOLLOW -> APPROACH; APPROACH -> STOP; STOP -> BACKUP if `< g_turnMinMm`; SHIFT_RIGHT, PASS_OBSTACLE, SEARCH_LINE -> HALT |
| Approach timeout | 5000 ms -> back to FOLLOW, obstacles ignored for 500 ms |
| Backup timeout | 1500 ms -> HALT |
| Line-search timeout | 4000 ms -> FAILSAFE_SEARCH |
| Alignment timeout | 2500 ms -> FAILSAFE_SEARCH |
| Failsafe sweep timeout | 5000 ms -> HALT |
| Lost-line timeout | 3680 ms from loss -> HALT |
| Calibration check | IR4 equal to the floor level -> no start |
| User stop | BOOT button or `s` stops at any time |
| Motor stop | `haltRobot()` applies the TB6612 short brake |

Not implemented: battery-voltage monitoring, motor stall or wheel-slip detection, a hardware
watchdog, a check that the right side is free, any check that the box is centred.

## 7. Known limits

- All detour movements are open loop and time based; battery voltage and floor friction change
  the real distances and angles. Recalibrate after changing battery state.
- During the turns, SHIFT and PASS the ToF faces the direction of travel only; the box
  is beside the robot and is not seen.
- Geometry formula limits: `CODE_VS_BRIEF_MISMATCHES.md` M7.
- The 250 mm emergency threshold can trip on walls during the detour (M8).
