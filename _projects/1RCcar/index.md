---
layout: post
title: RC car
description:  Designed, 3D printed, and assembled a remote-controlled cars including suspension, steering, customized PCB, and power transmission. Iterative CAD design and testing were used to improve durability, handling, and ease of assembly.
skills: 
- Solidwork
- 3D printing
- Onshape
main-image: /Chassis.jpg
---
---
## Electronics
The ESP32-based control electronics and brushed motor mounted on the chassis.
{% include image-gallery.html images="Electronics.jpg" height="400" %}

## Onshape model
{% include image-gallery.html images="FinalAssem.png" height="400" %}

## Steering and suspension
{% include image-gallery.html images="Suspension_steering.png" height="400" %}

## Driving demo
{% assign video_base = page.url | remove: 'index' | remove: '.html' %}
<video controls playsinline preload="metadata" style="width:360px; max-width:100%; height:auto; border-radius:6px; display:block;">
  <source src="{{ video_base }}Demo.mp4" type="video/mp4">
  Your browser does not support the video tag. You can
  <a href="{{ video_base }}Demo.mp4">download the clip</a> instead.
</video>
---


