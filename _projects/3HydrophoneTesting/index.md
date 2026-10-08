---
layout: post
title: Transducer Testing with a Hydrophone
description: Characterized ultrasound transducers in a water tank at Averkiou's lab. I drove each transducer with a tone burst from a signal generator and RF amplifier, positioned a hydrophone in its acoustic field with a LabVIEW-controlled motorized stage, and measured the received signal on an oscilloscope.
skills:
- Ultrasound transducers
- Hydrophone measurement
- LabVIEW
- Signal generator & RF amplifier
- Oscilloscope
main-image: /setup.jpg
---
---

## Overview

At Averkiou's lab I tested ultrasound transducers by measuring their acoustic output in water with a hydrophone. The transducer and hydrophone are both submerged in a glass tank. A motorized positioner mounted on an aluminum frame above the tank moves the hydrophone, and I controlled it from LabVIEW so I could place it at precise, repeatable positions in the transducer's field.

{% include image-gallery.html images="setup.jpg" height="500" %}

## Equipment

| Instrument | Role |
|---|---|
| Signal generator | Creates the tone-burst drive signal |
| RF power amplifier | Boosts the burst to drive the transducer |
| Ultrasound transducer | Device under test, submerged in the tank |
| Hydrophone | Picks up the pressure wave in the water |
| Motorized positioner + LabVIEW | Moves the hydrophone along multiple axes |
| Tektronix DPO7054C oscilloscope | Captures and measures the received waveform |

## Procedure

1. Generate a short sinusoidal tone burst with the signal generator.
2. Amplify the burst and use it to drive the transducer.
3. Use LabVIEW to move the positioner and place the hydrophone at the target point in the acoustic field.
4. Capture the hydrophone signal on the oscilloscope, triggered off the signal generator.
5. Measure peak-to-peak voltage, arrival time and burst length with the scope's cursors and measurement tools.

## Result

{% include image-gallery.html images="scope.jpg" height="500" %}

The hydrophone picked up a clean tone burst of about 11 cycles with very little background noise. Its peak-to-peak amplitude was 144.7 mV, and it arrived about 50 µs after the trigger. That delay is the time the wave takes to travel through the water from the transducer to the hydrophone.

---
