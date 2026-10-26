---
layout: post
title: RC Car Remote & Receiver PCBs
description: Designed a matched pair of two-layer KiCad boards for my RC car, a handheld transmitter and an on-car receiver, both built around an Arduino Nano and an nRF24L01+ PA+LNA 2.4 GHz radio. The work covered circuit revisions, part and footprint selection, power design for a 2S Li-ion pack, placement and routing, and DRC checks on both boards.
skills:
- KiCad
- Schematic capture
- PCB layout
- Arduino Nano
- nRF24L01+ (SPI)
- Power design (buck, LDO)
main-image: /pcb.jpg
---
---

## Overview

The car needs two boards that talk to each other over 2.4 GHz radio:

- **Transmitter (TX), the remote.** It reads an analog joystick and two buttons, packs
  them into a small message, and sends it to the car many times a second.
- **Receiver (RX), the car.** It listens for those messages and turns them into servo
  pulses for the steering servo and the ESC that drives the motor.

Both boards share the same core: an **Arduino Nano**, an **nRF24L01+ PA+LNA** radio
module on hardware SPI, a **2S Li-ion** battery (6.0–8.4 V), and a regulated 5 V and
3.3 V supply. I placed and routed both boards myself in KiCad. The schematics started
from my first hand-drawn versions and were revised and reviewed with help from an AI
assistant (Claude).

| | Transmitter (remote) | Receiver (car) |
|---|---|---|
| Board | 58.8 × 52.0 mm, 2-layer | 63.5 × 53.0 mm, 2-layer |
| MCU | Arduino Nano (ATmega328P), socketed | Arduino Nano (ATmega328P), socketed |
| Radio | nRF24L01+ PA+LNA, external antenna | nRF24L01+ PA+LNA, external antenna |
| Battery | 2 × 18650 (2S) on JST-XH | 2S Li-ion on XT30, shared with the ESC |
| 5 V supply | MP1584EN buck module | 5 V 3 A UBEC (2S–6S input) |
| 3.3 V supply | AMS1117-3.3 LDO for the radio | AMS1117-3.3 LDO for the radio |
| I/O | Joystick (2 analog axes + push), 2 buttons | Steering servo, ESC, 1 spare servo channel |
| Extras | Power LED, link LED, battery monitor | Power LED, link LED, battery monitor, 470 µF servo bulk cap |

---

## PCB layout

### Transmitter

{% include image-gallery.html images="tx-layout.png, tx-3d-iso.jpg" height="420" %}

- The Nano's **USB port faces the top edge**, so it can be programmed in place.
- The **radio header sits right beside the Nano's SPI pins**, which keeps the SPI traces
  short, and the module extends past the right edge so its SMA antenna is clear of the board.
- Battery, power switch, and buck-module connectors line the **left edge**, where the
  wires leave the board. Buttons sit **bottom-right** under the thumb, next to the pins they
  use.

### Receiver

{% include image-gallery.html images="rx-layout.png, rx-3d-iso.jpg" height="420" %}

- The **XT30 battery connector faces the board edge**, with nothing in front of the plug.
  An earlier placement had it facing inward, behind an LED and a resistor.
- The radio module **overhangs the bottom edge**, so nothing tall sits underneath it.
- The three servo/ESC headers sit together beside the Nano, and the battery input path
  runs on wider traces.

### Both boards

- **Track widths.** Signal traces are 0.3 mm. Power traces are 1.0 mm, and the car's battery
  input path is 1.5 mm.
- **Ground.** Both boards have a solid GND pour on each layer.
- **Checks.** Both boards have **zero unconnected items**, and every net matches the
  schematic. What DRC still reports is cosmetic: silkscreen clipped where a module
  overhangs the board edge, and single-spoke thermal reliefs on two of the Nano's
  ground pins, which are tied together on the Nano itself anyway.

---

## Circuit design

Both circuits are split into the same functional blocks: **Power**, **Radio (3.3 V)**,
**MCU**, and **Controls** (TX) or **Outputs** (RX).

### Power

- **Reverse-polarity protection.** A Schottky diode sits in series after the power
  switch: a 1N5819 (1 A) on the remote, and a 1N5822 (3 A) on the car, because that one also
  carries the servo current. Plugging a battery in backwards just does nothing.
- **5 V rail.** The remote uses a small MP1584EN buck module, since it draws only about
  130 mA. The car uses a **5 V 3 A UBEC**, because steering servos can pull 2–3 A when they
  stall. I first picked an XL4015 module, then found it needs at least 8 V in, which a
  2S pack drops below for most of its discharge, so I swapped it for a UBEC rated for 2S–6S.
- **3.3 V rail for the radio.** The nRF24L01+ is a 3.3 V part and the Nano's own 3V3 pin
  can't supply its transmit bursts, so each board has its own **AMS1117-3.3** with 10 µF on
  the input and output.
- **Radio decoupling.** The PA+LNA module pulls roughly 115 mA in short bursts when it
  transmits. A **100 nF** capacitor sits a few millimetres from its VCC pin, backed by a
  **100 µF** bulk capacitor. Poor decoupling is the usual cause of dropped packets with
  these modules.
- **Off-board power switches.** The PA+LNA current is more than a small PCB slide switch
  is rated for (100 mA), so both boards bring the switch out on a JST-XH header to a
  panel-mounted MTS-102 toggle switch. The 5 V converters connect the same way, through a
  4-pin JST-XH header, so any module can be swapped in.
- **Battery monitor.** A 20 kΩ / 10 kΩ divider scales the battery to one third
  (8.4 V → 2.8 V) so the Nano's ADC can read it. Firmware can then warn at about 6.8 V and stop
  the car at 6.4 V, before the Li-ion cells are over-discharged.

### MCU and radio

- **Hardware SPI.** The radio uses the Nano's SPI pins (D11 MOSI, D12 MISO, D13 SCK),
  with CE on D9 and CSN on D10, the RF24 library's default wiring.
- **Pin choices on the remote follow the PCB.** All SPI and radio pins sit at the Nano's
  USB end, so the radio goes there. The joystick uses **A6/A7**, which are at the opposite end
  next to the Nano's 5 V and GND pins and the button pins (D2–D4). That keeps the
  joystick's five wires together, and the battery monitor moves to A0.
- **Buttons** use the ATmega328P's internal pull-ups (`INPUT_PULLUP`), so they need no
  external resistors.
- **Servo outputs** on the car (D3, D5, D6) carry standard 1–2 ms servo pulses, so the
  same headers work for a servo or an ESC.
