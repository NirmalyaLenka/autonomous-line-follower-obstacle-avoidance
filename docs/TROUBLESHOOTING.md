# Troubleshooting

| Problem | Likely cause | Test | Solution |
|---|---|---|---|
| Robot moves backward | one motor (or both) reversed | `t` motor test | set `LEFT_INVERT` / `RIGHT_INVERT`, or swap the motor leads |
| Only one motor works | TB6612 wiring, motor lead, GPIO, `STBY` low | `t`; check `STBY` (GPIO5) is high; measure `AO1/AO2`, `BO1/BO2` | fix the wiring (see `WIRING.md`); check the driver's VM supply |
| Robot oscillates | `KP` too high or `KD` too low | `d` and watch `pos` swing | lower `KP`, raise `KD` (`TUNING.md`) |
| Robot reacts slowly | `KP` too low, speed too low, `KD` too high | slow, late corrections | raise `KP`, lower `KD`, raise `BASE_SPEED` |
| Robot loses line at curves | speed too high, `KP`/`KD` wrong, `SLOWDOWN` too low | `d`, look for `cnt=0` at curve entry | lower `BASE_SPEED`, raise `SLOWDOWN`, retune |
| Robot steers away from the line | array orientation vs motors reversed | `i` while moving a strip | `CONTROL_ALGORITHM.md` "Reversing the steering direction" |
| ToF distance unstable | alignment, surface, wiring, ambient light | `v` | aim at a flat matt target, shorten I2C wires, check pull-ups, check supply |
| `VL53L0X not found` at boot | wiring (SDA GPIO21 / SCL GPIO22), supply, address | serial log | check wiring and power; obstacle avoidance is disabled meanwhile |
| False obstacle detection | noise, reflective surfaces, confirmation too low | `v`, `d` | raise `OBSTACLE_CONFIRM_COUNT`; check sensor aim; remove the nearby object |
| Robot never detects the box | box too low / ToF aim / dark surface | `v` in front of the box | re-aim the sensor, check the box's reflectivity |
| Robot hits the obstacle | stop distance too low, geometry wrong, braking overshoot | `g`; measure the real stop distance | raise `OBSTACLE_STOP_MM` / `STOP_BRAKE_ALLOWANCE_MM`, lower `APPROACH_SPEED`, re-measure geometry |
| Robot backs up repeatedly | stops closer than `g_turnMinMm` | serial: "Too close to turn" | raise the stop distance or lower `APPROACH_SPEED` |
| Robot turns too far | `TURN_x_MS` too high | `r` / `l` | lower `TURN_RIGHT_MS` / `TURN_LEFT_MS` |
| Robot does not reach the passing position | shift timing too low | `f` | re-measure `DRIVE_MM_PER_SEC` (a lower value gives a longer time) or type a longer `RIGHT_SHIFT_MS` |
| Robot clips the box on the second turn | pass too short (M7) | `o` | raise `FORWARD_PASS_MS` |
| Robot passes the obstacle but misses the line | forward pass, turn or line-search parameters | `o`, watch SEARCH_LINE | recheck `TURN_LEFT_MS`, `FORWARD_PASS_MS`, `LINE_SEARCH_*` |
| Halts with "object too close during SHIFT_RIGHT / PASS_OBSTACLE / SEARCH_LINE" | `OBSTACLE_EMERGENCY_MM` (250) trips on a wall | serial | lower `OBSTACLE_EMERGENCY_MM`, move the track away from walls |
| "line not reacquired - sweep timed out" | line search or alignment failed | `i` after the halt | recheck turn times and sensor polarity |
| "line lost - end of track" at a gap | gap longer than `GAP_CROSS_MS` plus search | `d` | increase `GAP_CROSS_MS` (and recheck) |
| `CALIBRATION FAILED: IR4 reads the same as the floor` | IR4 not on the line, pot, polarity | `i` | put IR4 on the line; retune the pot |
| "IR1/IR2/IR6 disagree" warning | IR1 defective or a sensor on the line | `i` | expected with the dead IR1; see `CODE_VS_BRIEF_MISMATCHES.md` M1 |
| ESP32 resets | power instability, buck converter, missing common ground | watch the brownout message; measure the 3.3 V/5 V rail during motor start | separate motor and logic wiring, check the buck and ground; see `POWER_ARCHITECTURE.md` |
| IR sensor always reads the same value | height, potentiometer, wiring, polarity | `i` | adjust height and pot; check VCC/GND/DO wiring |
| Robot does not start when BOOT is pressed | GPIO0 held low at reset, or serial busy | check the serial log | release BOOT before reset; press again after boot |
| Geometry warning on start | `GEOMETRY_MEASURED = false` | start log | measure the geometry and set it to `true` (intentional warning) |
