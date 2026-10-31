# Wiring

Pin definitions come from `src/main.ino`. Do not change them: the pin map is marked
`DO NOT CHANGE` in the sketch.

## Full table

| Component | Pin | ESP32 GPIO | Function | Notes |
|---|---|---|---|---|
| TB6612FNG | PWMA | 25 | left motor speed (PWM, 20 kHz, 8 bit) | |
| TB6612FNG | AIN1 | 26 | left motor direction | |
| TB6612FNG | AIN2 | 27 | left motor direction | |
| TB6612FNG | STBY | 5 | driver enable (set HIGH) | GPIO5 is a strapping pin on the ESP32 |
| TB6612FNG | BIN1 | 33 | right motor direction | |
| TB6612FNG | BIN2 | 4 | right motor direction | |
| TB6612FNG | PWMB | 32 | right motor speed (PWM) | |
| TB6612FNG | VCC | 3V3 | logic supply | 3.3 V logic as specified |
| TB6612FNG | VM | battery + (7.4 V side) | motor supply | verify polarity first |
| TB6612FNG | GND | common GND | | |
| TB6612FNG | AO1 / AO2 | | left BO motor | swap leads or use `LEFT_INVERT` if reversed |
| TB6612FNG | BO1 / BO2 | | right BO motor | swap leads or use `RIGHT_INVERT` if reversed |
| IR1 | DO | **14** | **defective: do not use** | reserved; the sketch still scans this pin (M1). Leave unconnected or tie it at the floor level, **USER MUST VERIFY** what it reads |
| IR2 | DO | 13 | line sensor | weight -2 |
| IR3 | DO | 16 | line sensor | weight -1 |
| IR4 | DO | **17** | line sensor, **centre / reference** | 4-pin module; must stay on GPIO17; weight 0 |
| IR5 | DO | 18 | line sensor | weight +1 |
| IR6 | DO | 19 | line sensor | weight +2 |
| IR7 | DO | 23 | line sensor | weight +3 |
| IR modules | VCC / GND | regulated supply / common GND | | **USER MUST VERIFY** module voltage and DO level versus 3.3 V GPIOs |
| VL53 ToF | SDA | 21 | I2C data | |
| VL53 ToF | SCL | 22 | I2C clock | 400 kHz |
| VL53 ToF | VCC / GND | regulated supply / common GND | | **USER MUST VERIFY** the breakout's supply range |
| BOOT button | | 0 | start / stop | on the dev board, no wiring; strapping pin |
| LED | | 2 | status | on the dev board, no wiring; strapping pin |
| Gyro | | | not used | do not wire |

## Direction and speed

- `AIN1` high, `AIN2` low drives the left motor forward (before `LEFT_INVERT`); both high is the short brake.
- `BIN1` / `BIN2` do the same for the right motor.
- `PWMA` / `PWMB` carry the PWM duty (0..255). All three states (forward, reverse, brake) use it.

## Strapping pins

GPIO0 (BOOT button), GPIO2 (LED) and GPIO5 (`STBY`) are strapping pins on the ESP32. Avoid
holding BOOT during reset (it enters the bootloader). If the board behaves strangely at
power-up, check that nothing external pulls these pins.

## Power wiring rules

- Battery positive goes to `VM` of the driver and to the buck converter input. The ESP32 and
  the sensors are on the regulated side.
- **Common ground** between battery, buck, driver, ESP32 and all sensors.
- Motor current must not run through thin logic wiring: use short, thick leads for battery and `VM`/motor wires.
- Verify the buck converter's output voltage with a multimeter **before** connecting the ESP32.
- Verify polarity before the first power-up.
- See `hardware/POWER_ARCHITECTURE.md`.

## I2C

SDA = GPIO21, SCL = GPIO22, 400 kHz. Keep the wires short and away from motor leads. Check the
breakout for pull-up resistors (**USER MUST VERIFY**); if you add external pull-ups use a value suitable
for 3.3 V at 400 kHz.

## First power-up checklist

```
[ ] battery polarity checked
[ ] buck converter output measured before the ESP32 is connected
[ ] common ground confirmed
[ ] motors disconnected or wheels lifted for the first run
[ ] serial monitor shows "VL53L0X OK"
[ ] `t` motor test: directions correct
```
