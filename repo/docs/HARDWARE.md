# Hardware

Component list with part notes: `hardware/BOM.md`. Pin table: `hardware/PINOUT.md`.
Wiring: `WIRING.md`. Power: `hardware/POWER_ARCHITECTURE.md`.

## Architecture

| Block | Part | Notes |
|---|---|---|
| Controller | ESP32 Dev Module | Arduino IDE, board "ESP32 Dev Module", core 2.x or 3.x |
| Motor driver | TB6612FNG | dual H-bridge, PWM + two direction pins per motor, `STBY` enable |
| Motors | 2 x BO geared DC motors | differential drive (left = driver channel A, right = channel B) |
| Line sensors | 7 positions planned, **6 working** | IR1 defective and unused; IR2..IR7 active |
| Distance sensor | VL53-series ToF | the software interface is VL53L0X-compatible (`Adafruit_VL53L0X`), I2C |
| Gyro | available during development | **not used by the sketch**; optional / not required |
| Power | 7.4 V battery plus "Total 360" buck converter | see `POWER_ARCHITECTURE.md` |
| Chassis | custom | about 250 mm long, about 190 mm wheel span/width |

## Chassis

The chassis is custom. Approximate dimensions of the reference robot: length 250 mm, width
(wheel span) 190 mm. These are the inputs of the obstacle geometry calculation:
`ROBOT_LENGTH_MM`, `ROBOT_WIDTH_MM` and the corner radius derived from them. **They must be
changed when another builder uses a different chassis.** See `ROBOT_SIZE_ADAPTATION.md`.

Layout requirements that the software assumes:

- Differential drive: two driven wheels on a common axle line, plus a caster (or other passive
  support). The wheel axle is the rotation centre for in-place turns.
- The sensor bar is across the front and perpendicular to the direction of travel. IR4
  (GPIO17) is on the robot's centreline.
- The ToF sensor faces straight forward on the centreline.
- `TOF_SENSOR_TO_ROTATION_CENTER_MM` (reference: 90 mm) is measured from the sensor window to the axle line.
  It is not the same as half the robot's length unless the axle is mid-length; see
  `CODE_VS_BRIEF_MISMATCHES.md` M7.

## Line sensor bar

From the sketch header: 17 mm pitch, 180 mm bar. Positions from the bar's left edge:

| Sensor | GPIO | Position (mm) | Status |
|---|---|---:|---|
| IR1 | 14 | 39 | **defective, unused (reserved)** |
| IR2 | 13 | 56 | active |
| IR3 | 16 | 73 | active |
| IR4 | 17 | 90 (centre) | active, 4-pin module, use its DO pin |
| IR5 | 18 | 107 | active |
| IR6 | 19 | 124 | active |
| IR7 | 23 | 141 | active |

The code header describes six 3-pin modules with a potentiometer each and IR4 as a 4-pin module.
With IR1 out, that leaves five 3-pin modules plus IR4 in use. **USER MUST VERIFY** the module types
on your own bar. A 20 mm line on a 17 mm pitch gives one or two active sensors when aligned.

IR1 was deliberately excluded because it is defective. The final robot operates with six sensors.

Voltage warning: ESP32 GPIOs are **not 5 V tolerant**. If your IR modules are powered at 5 V, check
the level of their DO pin before connecting it to a GPIO. **USER MUST VERIFY** the module supply and output level.

## ToF sensor

- VL53-series breakout, I2C: SDA = GPIO21, SCL = GPIO22 (`Wire.begin(PIN_SDA, PIN_SCL)`, 400 kHz).
- The breakout's supply range and whether it has onboard pull-ups: **USER MUST VERIFY**.
- Mount it facing forward on the centreline, at a height where it sees the face of the obstacle
  (box height **USER MUST MEASURE**).
- Keep the optical window clear of the chassis; fixed mounting avoids changing the aim.

## Motors and driver

BO geared motors are driven through the TB6612FNG. The motors' rated voltage and stall current
are **USER MUST VERIFY**: the project information does not state them, and the battery is 7.4 V nominal.
The TB6612FNG's VCC (logic) is powered from the ESP32's 3.3 V in this design. The motor supply `VM`
is connected to the battery side. All grounds are common.

Functions per channel:

- `AIN1` / `AIN2` set the left motor's direction; `PWMA` sets its speed by PWM.
- `BIN1` / `BIN2` set the right motor's direction; `PWMB` sets its speed by PWM.
- `STBY` enables the driver; the sketch drives it high.

## Gyro

A gyro was available but the final code does not use it. Do not wire it for this firmware.

## Mechanical notes

- Rigid sensor-bar mounting at a constant height above the track: height errors change the
  sensor threshold behaviour.
- Rigid ToF mounting.
- Battery low and secured; the caster should not catch on the line.
- Keep logic wiring away from motor leads.
