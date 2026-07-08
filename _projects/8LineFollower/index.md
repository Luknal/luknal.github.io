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

The Arduino firmware was adapted from the provided motor-driver template — mapping pins to
our physical assembly, driving the two motors through the Adafruit shield, and integrating
the sensor readings into the control loop.

### Motor-driver bring-up sketch

Before wiring in the sensors and control loop, we used this sketch to verify the motor
shield and confirm both wheels ran in the correct direction — the LED blinks for two
seconds as a start delay, then the motors run forward, reverse, and stop on a loop.

```cpp
#include <Wire.h>
#include <Adafruit_MotorShield.h>

// Initialize the motor shield and the two drive motors
Adafruit_MotorShield AFMS = Adafruit_MotorShield();
Adafruit_DCMotor *Motor1 = AFMS.getMotor(1);
Adafruit_DCMotor *Motor2 = AFMS.getMotor(2);

// Base motor speeds (0–255); kept separate so a faster motor can be trimmed
int M1Sp = 60;
int M2Sp = 60;

int led_Pin = 13;

void setup() {
  Serial.begin(9600);
  AFMS.begin();                 // start talking to the motor shield
  pinMode(led_Pin, OUTPUT);

  // Blink for ~2 s so there's a moment before the cart actually moves
  for (int i = 0; i < 20; i++) {
    digitalWrite(led_Pin, HIGH); delay(100);
    digitalWrite(led_Pin, LOW);  delay(100);
  }
}

void loop() {
  // Forward for 3 s
  Motor1->setSpeed(M1Sp); Motor1->run(FORWARD);
  Motor2->setSpeed(M2Sp); Motor2->run(FORWARD);
  delay(3000);

  // Reverse for 3 s
  Motor1->run(BACKWARD);
  Motor2->run(BACKWARD);
  delay(3000);

  // Stop for 3 s
  Motor1->run(RELEASE);
  Motor2->run(RELEASE);
  delay(3000);
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
