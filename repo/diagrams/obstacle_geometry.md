# Obstacle geometry (top views)

Reference robot: L = 250 mm, W = 190 mm, ToF-to-axle 90 mm, box 100 x 100 mm, margin 20 mm.
Dimensions are not drawn to scale.

## 1. Corner radius (why the robot is not a point)

```
          W = 190
   +-------------------+
   |         ^         |        R_corner = sqrt((L/2)^2 + (W/2)^2)
   |         |         |                 = sqrt(125^2 + 95^2)
   |   L/2=125         |                 = 157.003  -> ceil -> 158 mm
   |         |         |
   |   (axle/rotation centre)
   |         |
   |         v  W/2=95 -->
   +-------------------+
        L = 250
```

When the robot spins in place, its corners sweep a circle of radius `R_corner` about the
rotation centre.

## 2. Stop and minimum turning distance

```
 box face                                 rotation centre
   |<----- d (ToF reading) ----->[ToF]<--90 mm-->[axle]
   |
   stop = 300 mm  (g_stopEffMm)
   D_min_safe = 90 + 158 + 20 = 268 mm (g_minSafeStopMm)

   trigger 400 mm > stop 300 mm > D_min_safe 268 mm > emergency 250 mm
   clear 440 mm above trigger
```

As implemented, `D_min_safe` adds the sensor offset (exact if the sensor is behind the axle;
conservative if ahead: `CODE_VS_BRIEF_MISMATCHES.md` M7).

## 3. Detour path

```
 north
   ^
   |   x = 0 (line)             x = 165 mm east of the line
   |     |
   |     |  <---- (5) west ----- end of PASS_OBSTACLE
   |   +-----+                        ^
   |   | BOX |                        | (4) north, D_forward = 545 mm
   |   +-----+                        |
   |     |                            |
   |     |        (3) face north -----+
   |     |  (2) east, D_right = 165 mm
   |     S  (1) stop position: robot centre on the line,
   |            ToF 300 mm from the box face
   +-----------------------------------------> east
```

The line is 20 mm wide and runs north-south through the box. Not to scale.

1. TURN_RIGHT: face east.
2. SHIFT_RIGHT: drive `D_right = 50 + 95 + 20 = 165 mm`.
3. TURN_FORWARD: face north, robot centre 165 mm east of the line.
4. PASS_OBSTACLE: drive `D_forward = 300 + 100 + 125 + 20 = 545 mm`.
5. TURN_TO_LINE: face west; SEARCH_LINE creeps west until a sensor sees the line.
6. ALIGN_CREEP, ALIGN_TURN (right, to north), ALIGN_FINE, RESUME.

## 4. Lateral clearance

```
 line centre
     |<--- 50 --->| box half-width
     |            |
     |            |<-- 20 mm margin -->|<----- 95 (W/2) ----->| robot centre
     |<-------------------- D_right = 165 mm ------------------>|
```

`D_right = B/2 + W/2 + M = 50 + 95 + 20 = 165 mm`: the robot's near edge ends 20 mm from the
box's near edge.

## 5. Time conversion

```
RIGHT_SHIFT_MS  = 165 / 200 * 1000 = 825 ms     (DRIVE_MM_PER_SEC = 200, placeholder)
FORWARD_PASS_MS = 545 / 200 * 1000 = 2725 ms
```
