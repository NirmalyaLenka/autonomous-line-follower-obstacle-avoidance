# Obstacle state machine

Matches `avGoto()` / `avStep()` / `runStep()` in `src/main.ino`. `HALTED` is `g_state = ST_IDLE`
(motors braked, reached through `haltRobot()`).

```mermaid
stateDiagram-v2
    [*] --> FOLLOW_LINE
    FOLLOW_LINE --> APPROACH_OBSTACLE : 3 readings at or below trigger or emergency
    FOLLOW_LINE --> HALTED : line lost for 3680 ms
    APPROACH_OBSTACLE --> STOP_FOR_OBSTACLE : 2 readings at or below stop or emergency
    APPROACH_OBSTACLE --> FOLLOW_LINE : 4 readings above clear or 5000 ms timeout
    APPROACH_OBSTACLE --> HALTED : ToF stale over 600 ms or line lost
    STOP_FOR_OBSTACLE --> BACKUP_TO_SAFE : distance below turn minimum
    STOP_FOR_OBSTACLE --> TURN_RIGHT : distance ok
    STOP_FOR_OBSTACLE --> HALTED : ToF stale
    BACKUP_TO_SAFE --> STOP_FOR_OBSTACLE : distance reaches stop distance
    BACKUP_TO_SAFE --> HALTED : 1500 ms or ToF stale
    TURN_RIGHT --> SHIFT_RIGHT : TURN_RIGHT_MS
    SHIFT_RIGHT --> TURN_FORWARD : g_rightShiftMs
    SHIFT_RIGHT --> HALTED : emergency
    TURN_FORWARD --> PASS_OBSTACLE : TURN_LEFT_MS
    PASS_OBSTACLE --> TURN_TO_LINE : g_forwardPassMs
    PASS_OBSTACLE --> HALTED : emergency
    TURN_TO_LINE --> SEARCH_LINE : TURN_LEFT_MS
    SEARCH_LINE --> ALIGN_CREEP : line seen for 5 ticks
    SEARCH_LINE --> FAILSAFE_SEARCH : 4000 ms timeout
    SEARCH_LINE --> HALTED : emergency
    ALIGN_CREEP --> ALIGN_TURN : 140 ms
    ALIGN_TURN --> ALIGN_FINE : TURN_RIGHT_MS
    ALIGN_FINE --> RESUME : aligned for 5 ticks
    ALIGN_FINE --> FAILSAFE_SEARCH : 2500 ms timeout
    FAILSAFE_SEARCH --> RESUME : aligned for 5 ticks
    FAILSAFE_SEARCH --> HALTED : 5000 ms timeout
    RESUME --> FOLLOW_LINE : immediately
```

## Transitions

| From | To | Condition (code) |
|---|---|---|
| FOLLOW_LINE | APPROACH_OBSTACLE | `emergencyNow()` or (`g_ignoreObstUntil` passed and `obstacleAhead()`) |
| APPROACH_OBSTACLE | STOP_FOR_OBSTACLE | `emergencyNow()` or `g_stopCnt >= OBSTACLE_STOP_CONFIRM` |
| APPROACH_OBSTACLE | FOLLOW_LINE | `g_clearCnt >= OBSTACLE_CLEAR_CONFIRM`, or `e > APPROACH_TIMEOUT_MS` (then `g_ignoreObstUntil = now + 500`) |
| APPROACH_OBSTACLE | HALTED | stale ToF longer than `TOF_STALE_HALT_MS`; or the follower halts |
| STOP_FOR_OBSTACLE | BACKUP_TO_SAFE | `g_dist < g_turnMinMm` |
| STOP_FOR_OBSTACLE | TURN_RIGHT | otherwise |
| BACKUP_TO_SAFE | STOP_FOR_OBSTACLE | `g_dist >= g_stopEffMm` |
| BACKUP_TO_SAFE | HALTED | `e >= BACKUP_MAX_MS`, or stale |
| TURN_RIGHT | SHIFT_RIGHT | `e >= TURN_RIGHT_MS` |
| SHIFT_RIGHT | TURN_FORWARD | `e >= g_rightShiftMs` |
| TURN_FORWARD | PASS_OBSTACLE | `e >= TURN_LEFT_MS` |
| PASS_OBSTACLE | TURN_TO_LINE | `e >= g_forwardPassMs` |
| TURN_TO_LINE | SEARCH_LINE | `e >= TURN_LEFT_MS` |
| SEARCH_LINE | ALIGN_CREEP | `g_stable >= LINE_CONFIRM_READINGS` |
| SEARCH_LINE | FAILSAFE_SEARCH | `e >= LINE_SEARCH_TIMEOUT_MS` |
| ALIGN_CREEP | ALIGN_TURN | `e >= ALIGN_CREEP_MS` |
| ALIGN_TURN | ALIGN_FINE | `e >= TURN_RIGHT_MS * ALIGN_TURN_PERCENT / 100` |
| ALIGN_FINE | RESUME | `centerAligned()` for `CENTER_CONFIRM_READINGS` ticks |
| ALIGN_FINE | FAILSAFE_SEARCH | `e >= ALIGN_TIMEOUT_MS` |
| FAILSAFE_SEARCH | RESUME | `centerAligned()` for `CENTER_CONFIRM_READINGS` ticks |
| FAILSAFE_SEARCH | HALTED | `e >= FAILSAFE_SEARCH_MS` |
| RESUME | FOLLOW_LINE | immediately (restarts the ramp, caps speed, ignores obstacles for 100 ms) |

Every state except FOLLOW_LINE and APPROACH_OBSTACLE starts with a 150 ms brake pause
(`AV_SETTLE_MS`). The `o` serial command starts at TURN_RIGHT and halts at RESUME.
