---
layout: post
title: Real-Time Lip-Sync for a 3D Digital Human
description: Built the speech and animation layer of a browser-based 3D assistant, where the avatar's mouth is driven by the per-phoneme timeline a VITS speech synthesizer reports from its own duration predictor rather than estimated from the audio waveform. The whole speech stack - synthesis, recognition, and text embeddings - runs self-hosted on CPU-only hardware with no third-party AI services.
skills:
- TypeScript
- three.js / WebGL
- VRM (@pixiv/three-vrm)
- Web Audio API
- Python
- Piper / VITS
- Digital signal processing
main-image: /demo.gif
---

---

## Overview

This is the animation and speech half of a digital-human assistant: a 3D character that listens,
answers, and speaks in the browser. My focus here is the part that is easy to do badly and hard to
do well — making the mouth match the voice.

The usual approach reads the audio's volume envelope and opens the jaw proportionally. It is simple,
and it looks like a glove puppet, because loudness cannot tell you *which* sound is being made.

Instead, the mouth is driven by the phoneme timeline emitted by the speech synthesizer itself.

**[▶ Try the interactive demo](/demo/vrm-lipsync/)** — a toggle switches between the two approaches
so you can compare them on the same audio.

---

## How it works

The speech synthesizer is [Piper](https://github.com/OHF-Voice/piper1-gpl), which is built on
[VITS](https://arxiv.org/abs/2106.06103). A VITS model contains a *duration predictor*: before any
waveform exists, it has already decided how many samples each phoneme will occupy. Asking the model
to report those alignments costs one extra output and yields ground truth from inside the
synthesizer:

```json
[
  { "p": "n", "t": 0.000, "d": 0.046 },
  { "p": "i", "t": 0.046, "d": 0.118 }
]
```

Each phoneme maps to the five mouth blendshapes the VRM 1.0 standard defines — `aa`, `ih`, `ou`,
`ee`, `oh` — and a renderer built on three.js and `@pixiv/three-vrm` eases the face toward those
targets in time with playback.

The full loop in the production system is: browser microphone → local speech recognition → streaming
LLM response → local speech synthesis → lip-synced playback.

---

## Measuring the difference

Not every voice service returns phoneme data. Edge TTS, Aliyun and Baidu return audio and nothing
else, so there is also a fallback analyzer that recovers mouth openness from the audio spectrum.
Having both behind one switch turns a design opinion into a measurement.

Sampling the same 10-second clip every 250 ms and recording which blendshape dominates:

| Cue source | Mouth shapes actually used |
|---|---|
| Phoneme timeline | `aa` `ee` `ih` `oh` `ou` — all five |
| Acoustic fallback | `aa` only — the jaw opens and closes |

The fallback cannot recover phoneme identity, only aperture. There is also a category of sound it
can never get right: bilabial closures — `m`, `b`, `p` — are *silent*. The lips meet during the stop
and the energy arrives in the burst afterwards, so an audio-only analyzer opens the mouth at exactly
the moment it should be shut. The production system patches this from the **text** instead, scanning
for characters whose pronunciation starts with a bilabial and inserting a closure before the
syllable's energy onset.

---

## Three problems worth describing

### The mouth must not reset at phoneme boundaries

The median phoneme in these clips lasts about 46 ms — faster than lips physically move. Snapping the
mouth to each new phoneme produces roughly 17 state changes per second and a visibly buzzing face.

So there is exactly one piece of mouth state, and every frame eases it toward the current target
with `1 - exp(-k·dt)`, which keeps the speed consistent regardless of frame rate. Opening is faster
than closing, matching real articulation. A short phoneme therefore only pulls the mouth partway
before the next target arrives — which is what coarticulation looks like in real speech.

Targets are also issued per *syllable* rather than per phoneme, with a minimum hold time, because
the target rate alone makes the mouth unfollowable no matter how the smoothing is tuned.

### Blendshapes add, so you cannot cross-fade them naively

VRM expressions are additive. Setting `aa = 0.5` and `ou = 0.5` does not produce a shape between the
two; it applies both deformations at once, and the result is an averaged half-open oval that
resembles neither.

During any transition two or three weights are non-zero by construction — the outgoing shape is
decaying while the incoming one rises — so a naive pipeline spends most of its time rendering that
averaged mouth. This is why so many avatars look like they are chewing rather than speaking. The fix
is to raise each weight to an exponent before applying it, widening the gap between the dominant and
secondary shapes, then normalise so the total never exceeds 1.

### Lip-sync has to lead the audio

Two separate corrections, both necessary. The smoothing itself lags by roughly one time constant
(~91 ms), so the cue track is sampled slightly into the future rather than speeding the smoothing up
— which would bring the jitter back. From that lead, the audio device's own output latency is
subtracted.

`AudioContext.currentTime` is a *scheduling* clock; sound leaves the speaker some milliseconds
later. Chrome frequently reports `outputLatency` as `0`, which means "not implemented" rather than
"none", so the code falls back to `baseLatency` — using JavaScript's `||` rather than `??`, since
`??` would keep the misleading zero and leave the sync wrong on most machines.

---

## Running without a GPU

The development machine has no CUDA GPU and the deployment target is an ordinary server, so every
self-hosted component had to be viable on CPU:

| Service | Role | Performance on CPU |
|---|---|---|
| Piper | Speech synthesis | RTF ≈ 0.16 — a 22-character sentence yields 4.5 s of audio in ~0.7 s |
| faster-whisper | Speech recognition | Real-time on a laptop CPU |
| bge-m3 | Text embeddings, behind an OpenAI-compatible API | ~0.5–2 s per batch |

Sentence-level prefetching overlaps synthesis with playback, so only the first sentence of a
response is ever waited on. Only LLM inference is a remote call, which means audio and documents
never leave the server.

---

## About the demo

The character model used in the original project is licensed `allowRedistribution: false`, so it is
not bundled with the live demo. The demo loads any VRM file you drop onto it — models from
[VRoid Hub](https://hub.vroid.com/) work — and the audio clips were pre-generated with a local Piper
install, so the page needs no backend at all.

The lip-sync modules in the demo are copied unmodified from the production application rather than
rewritten for display.
