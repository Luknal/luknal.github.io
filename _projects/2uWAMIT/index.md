---
layout: post
title: Ultrasound Imaging Analysis
description:  Helped developing a MATLAB script to track tumor motion from ultrasound imaging using microbubbles, keep the region of interest aligned over time, and extract quantitative data from the images. This analysis supports tumor detection and helps to determine the intensity of TARE treatment for the patients.
skills: 
- Ultrasound
- Matlab
- Microbubbles
main-image: /ContractFundamental.png
---
{% assign video_base = page.url | remove: 'index' | remove: '.html' %}
## Original
Contrast-enhanced ultrasound (CEUS) clip, with contrast mode on the left and B-mode on the right. Patient and scan details have been removed.
<video controls loop playsinline preload="metadata" style="width:640px; max-width:100%; height:auto; border-radius:6px; display:block;">
  <source src="{{ video_base }}original.mp4" type="video/mp4">
  Your browser does not support the video tag. You can
  <a href="{{ video_base }}original.mp4">download the clip</a> instead.
</video>

## ARG
Automatic respiratory gating: pick a trigger frame, track a generated ROI around a uniform bright spot, and remove frames where the intensity on the time-intensity curve (TIC) falls below a 15% tolerance.
<video controls loop playsinline preload="metadata" style="width:640px; max-width:100%; height:auto; border-radius:6px; display:block;">
  <source src="{{ video_base }}respiratory-gating.mp4" type="video/mp4">
  Your browser does not support the video tag. You can
  <a href="{{ video_base }}respiratory-gating.mp4">download the clip</a> instead.
</video>
---
{% include image-gallery.html images="tic.png" height="400" %}
---
{% include image-gallery.html images="par.png" height="400" %}
---
