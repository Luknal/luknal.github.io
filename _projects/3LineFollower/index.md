---
layout: post
title: Line-Following Robot Car
description: Designed and built an autonomous line-following robot for EE 201 that tracks a black line on a white surface using a custom photoresistor sensor array and a tunable SPID control loop on an Arduino Mega. Contributed a custom KiCad sensor PCB, a 3D-printed chassis and light shield, and modified Arduino firmware — the car completed the course in 53 seconds and was the first team to demo in-section.
skills:
- Arduino / C++
- KiCad PCB design
- Fusion 360
- 3D printing
- Control systems (PID)
- Soldering
main-image: /car.jpg
---
---

## Overview

The EE 201 final project is an autonomous robot car that follows a black track on a
white background. An array of photoresistors reads the contrast between the bright
surface and the dark line; an Arduino Mega processes those readings through a tunable
**SPID** control loop (Speed, Proportional, Integral, Derivative) and drives two DC
motors through an Adafruit motor shield to steer the car and keep it centered on the
line.

Built over five weeks by a three-person team (Logan Plesko, Daniel Li, Lilia Freire),
the car completed the track in **53 seconds** and was the **first team in our section to
demo**.

---

## Sensing the line

- **Custom sensor PCB** — designed in KiCad to hold the photoresistor sensors together
  with LEDs that illuminate the track surface, so the sensors read a consistent, well-lit
  contrast between line and background.
- **Calibration → hardcoded thresholds** — after calibration the photoresistors read
  ~85–110 over the dark line and ~10 to −20 over the white surface. We replaced live
  calibration with measured, hardcoded thresholds to cut setup time and shave seconds off
  each run.
- **3D-printed light shield** — fitted to the sensor PCB to block ambient light and lower
  the photoresistors close to the surface, which sharply improved reading accuracy and
  kept the car on the line.

The board interleaves seven photoresistors (R8–R14) with six illumination LEDs (D1–D6)
and their current-limiting resistors, breaking out to the Arduino through a single header.
Below is a render of the manufactured layout.

{% include image-gallery.html images="pcb.png" height="450" %}

{% include image-gallery.html images="car.jpg" height="450" %}

---

## Control and tuning

The car is steered by a **SPID loop** whose four gains are exposed on four potentiometers,
letting us tune behavior live on the track instead of recompiling. Final checked-off
values were **speed 40, P 20, I 8, D 50**, reached experimentally by iterating on the
track until the car tracked cleanly and handled tight turns.

Before the sensors were wired in, a short bring-up sketch verified the motor shield and
confirmed both wheels ran in the correct direction. The final firmware then reads the seven
photoresistors each pass, maps them to a 0–100 scale using the calibration bounds, collapses
them into a single steering error, and feeds that error through the PID law.

### The control loop

Each iteration runs the full sense → error → PID → drive pipeline:

```cpp
void loop() {
  ReadPhotoResistors();  // read 7 LDRs, map each to 0–100 via calibration bounds
  CalcError();           // collapse the 7 readings into one steering error (-3 … +3)
  PID_Turn();            // PID law -> per-motor trim
  RunMotors();           // apply trim on top of the base speed
}
```

Calibration and gain values can be read live from the four on-board potentiometers, or
pinned to measured constants via `USE_HARDCODED_CAL` / `USE_HARDCODED_SPID` toggles — the
latter skips the calibration routine entirely to cut setup time between runs.

### PID with custom tuning

Beyond the standard proportional/integral/derivative terms, we added a **non-linear edge
boost** so the car reacts harder as the line drifts toward the outer sensors (sharp turns),
plus **integrator anti-windup** to keep the accumulated error from saturating:

```cpp
void PID_Turn() {
  kP = (float)kPRead;  kI = (float)kIRead;  kD = (float)kDRead;

  // Non-linear error boost: small errors pass through, large errors get amplified
  const float EDGE_BOOST = 1.0;
  float normalized = error / 3.0;                           // -1.0 … +1.0
  float boost = 1.0 + EDGE_BOOST * normalized * normalized; // 1.0 at center, higher at edges
  error = error * boost;

  Turn = error * kP + sumerror * kI + (error - lasterror) * kD;

  sumerror += error;
  sumerror = constrain(sumerror, -5, 5);  // anti-windup clamp
  lasterror = error;

  M1P =  Turn;   // opposite trims steer the car
  M2P = -Turn;
}
```

`RunMotors()` then applies an **adaptive base speed** — the car eases off the throttle as the
error grows so it can take tight corners without overshooting, and runs near full speed on
the straights:

```cpp
void RunMotors() {
  // Slow down proportionally to error: 100% speed centered, down to ~30% at the edges
  float curveFactor = 1.0 - min(abs(error) / 3.0, 1.0) * 0.7;
  int adaptiveSp = (int)(SpRead * curveFactor);

  M1SpeedtoMotor = constrain(M1Sp + adaptiveSp + M1P, -255, 255);
  M2SpeedtoMotor = constrain(M2Sp + adaptiveSp + M2P, -255, 255);

  Motor1->setSpeed(abs(M1SpeedtoMotor));
  Motor2->setSpeed(abs(M2SpeedtoMotor));
  Motor1->run(M1SpeedtoMotor > 0 ? FORWARD : BACKWARD);
  Motor2->run(M2SpeedtoMotor > 0 ? FORWARD : BACKWARD);
}
```

---

## Hardware build

- **3D-printed chassis** — modeled in Fusion 360 and printed at the MILL; went through
  **four design iterations** to resolve fit and mounting constraints, producing a stable,
  balanced platform for the fastest possible run.
- **Separate motor power** — added a dedicated 9 V battery for the motors (independent of
  the 6 V Arduino supply) to eliminate brownouts when the motors drew current.
- **Fabrication** — hands-on soldering and 3D printing across the sensor board, shield,
  and chassis.

---

## Outcome

A working autonomous car that reliably completed the track, finishing in 53 seconds and
demoing ahead of the rest of the section. The project spanned the full hardware/software
stack — PCB design in KiCad, mechanical design in Fusion 360, 3D printing, soldering, and
embedded control in Arduino/C++ — and was a lesson in iterative design and team task
delegation.
