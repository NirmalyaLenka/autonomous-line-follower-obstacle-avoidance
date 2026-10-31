# Architecture

## Overview

```
 IR array (6 working + 1 unused) --> readSensors() -> g_pos, g_cnt, g_mask --+
                                                                             |
 VL53L0X --I2C--> tofTask (core 0) --critical section--> tofRefresh()        |
                                                              |              v
                                                       runStep()  <-- loop() (core 1)
                                                         |    |
                              FOLLOW_LINE: followLine()   |    +-> avStep(): obstacle state machine
                                                         v
                                              setMotors(l, r) -> TB6612FNG -> 2 BO motors
```

All code is in `src/main.ino`.

## Tasks and cores

| Piece | Where | Rate |
|---|---|---|
| `loop()` / `runStep()` | Arduino loop task (core 1) | gated to `LOOP_US = 4000 us` = 250 Hz |
| `tofTask()` | FreeRTOS task pinned to core 0, 6144-byte stack, priority 1 | as fast as `rangingTest()` returns, `vTaskDelay(1)` between calls |

The ToF task owns `t_dist`, `t_time`, `t_close`, `t_stop`, `t_clear`, `t_emg`. It writes
them inside `portENTER_CRITICAL(&g_tofMux)`. `loop()` copies them into `g_*` variables with
`tofRefresh()` once per control tick, so every decision in a tick sees one snapshot. The
thresholds `g_triggerEffMm` etc. are computed in `computeGeometry()` **before** the task
starts and are read-only afterwards.

## Modes

- `ST_IDLE`: motors stopped, LED blinks slowly. `s` or the BOOT button starts.
- `ST_RUN`: `runStep()` executes every tick.

Start sequence (`startSequence()`): geometry warning if `GEOMETRY_MEASURED` is false,
`calibrate()`, a 1.5 s LED blink (cancelable), reset of control state, then `ST_RUN`.

## Control tick

```
runStep():
  if (now - last tick < LOOP_US) return
  tofRefresh()
  if g_avState == AV_FOLLOW:
      if emergencyNow() or (ignore window passed and obstacleAhead()): avGoto(AV_APPROACH)
      else followLine()
  else:
      avStep()        // APPROACH ... RESUME
  optional debug print
```

## Non-blocking design

Timing uses `millis()` and `micros()` with unsigned subtraction (`now - t0`) and signed
comparison for "future" timestamps (`(int32_t)(now - g_avT0)`), which is safe across counter
wrap. A state records its start time in `g_avT0` and each tick compares elapsed time with the
state's duration. The settle pause is a start time set `AV_SETTLE_MS` into the future.

Blocking `delay()` is used only in: `calibrate()` (300 ms sampling), the start countdown
(1.5 s), `errorBlink()`, `motorTest()`, `timedRun()` (serial test drives) and the idle branch of
`loop()` (`delay(5)`).

Reasons for the state machine: sensors and ToF continue to be read during every phase,
the emergency stop stays responsive in the states that check it, each transition is explicit
and logged, and timing constants are independent parameters.

Loop frequency: `LOOP_US = 4000 us = 4 ms`, so `f = 1/0.004 = 250 Hz`.

## Motor layer

`setMotors(left, right)` takes values `-255..255`. `driveMotor()` applies the trim scale
(`LEFT_SCALE`, `RIGHT_SCALE`), clamps, applies the invert flag, writes the direction pins
(IN1/IN2) and the PWM duty. `motorsStop(true)` is the TB6612 short brake (IN1 = IN2 = HIGH, PWM 255);
`motorsStop(false)` is a coast stop (both inputs low, duty 0). `STBY` is set high
in `setup()` and in every `setMotors()`.

PWM: 20 kHz, 8 bit. The `PWM_ATTACH` / `PWM_WRITE` macros select `ledcAttach(pin, ...)` on
ESP32 Arduino core 3.x and `ledcSetup` + `ledcAttachPin` on core 2.x.

## Serial commands (115200 baud)

`s` start/stop, `c` calibrate, `d` debug stream, `t` motor test, `v` ToF distance (3 s),
`i` IR raw/active bits (3 s), `g` geometry, `r`/`l` turn tests, `f` shift test,
`p` pass test, `o` detour test, `h` help. Most are ignored while running.

## Global state summary

| Group | Variables |
|---|---|
| Sensors | `g_lineHigh[7]`, `g_mask`, `g_cnt`, `g_pos`, `g_calibrated` |
| PD | `g_lastErr`, `g_dFilt`, `g_lastPos`, `g_lastDir`, `g_lostFlag`, `g_lastSeen`, `g_rampStart` |
| ToF snapshot | `g_dist`, `g_distTime`, `g_closeCnt`, `g_stopCnt`, `g_clearCnt`, `g_emgCnt`, `g_tofOk` |
| Derived geometry | `g_cornerRadiusMm`, `g_minSafeStopMm`, `g_turnMinMm`, `g_stopEffMm`, `g_triggerEffMm`, `g_clearEffMm`, `g_emergencyEffMm`, `g_dRightMm`, `g_dForwardMm`, `g_rightShiftMs`, `g_forwardPassMs` |
| Detour | `g_avState`, `g_avT0`, `g_stable`, `g_alignDir`, `g_staleStart`, `g_speedCap`, `g_ignoreObstUntil`, `g_resumeCapUntil`, `g_testDetour` |

## Gyro

A gyro was available during development. The final sketch does not use it. Turns are
timed (`TURN_RIGHT_MS`, `TURN_LEFT_MS`). A gyro would be a natural way to replace
timed turns, but that is a design change and is not part of this project.
