# Audio Identification Edge Device R&D Guide

This repository is a **design note / R&D brief**, not a production implementation. It documents a practical path for building an edge device that can detect and classify gunshots from microphone audio while staying within microcontroller or small-FPGA limits.

This material is intended for **lawful, safety-focused research and evaluation**. It provides high-level R&D guidance for sensing, classification, and system design; it is **not** operational advice for weapon use or tactical deployment.

## Problem framing

The desired outcomes are:

1. **Gunshot vs. non-gunshot detection**
2. **Coarse distance estimation**
3. **Coarse ammunition / firearm class estimation**

Those goals are not equally difficult:

- **Detection** is very attainable on edge hardware.
- **Distance estimation** is attainable, but accuracy will depend heavily on calibration, microphone placement, weather, echoes, and whether you have one microphone or an array.
- **Exact round identification from audio alone** is the hardest goal and should be treated as **coarse class estimation** at first, such as:
  - handgun vs rifle
  - suppressed vs unsuppressed
  - subsonic vs supersonic
  - near vs mid vs far

## Recommended technical direction

Start with a **two-stage pipeline** instead of a single end-to-end model:

1. **Stage A: event detector**
   - Very small model or rule-based trigger
   - Finds impulsive events that look like gunshots
2. **Stage B: event classifier**
   - Runs only on short clipped windows around detected events
   - Predicts gunshot / non-gunshot, distance bucket, and weapon / round class

This is the simplest path that still supports the identification goals and edge constraints.

## Why this approach fits edge devices

Gunshots are short, high-energy impulsive events. That lets you:

- process audio in small windows
- wake heavier logic only when a candidate event occurs
- avoid running a larger classifier continuously
- keep memory, compute, and power usage low

## Signal theory to rely on

Useful characteristics in the waveform include:

- **Attack shape**: gunshots have a very fast onset
- **Peak amplitude and crest factor**: strong transient energy
- **Short-time spectrum**: energy distribution over frequency
- **Temporal decay**: how energy falls after the transient
- **Shockwave / muzzle blast structure**: for supersonic rounds, sometimes visible as multiple components depending on geometry
- **Reverberation pattern**: loosely related to distance and environment

For edge systems, you usually do **not** want raw waveform modeling first. Prefer compact hand-engineered features or very small spectrogram models.

## Audio front-end recommendation

Use a microphone chain that prioritizes **clean impulse capture** over bitrate marketing numbers:

- Prefer **sample rate** in the **48 kHz to 96 kHz** range
- Prefer **16-bit or 24-bit PCM**
- Ensure the microphone and ADC do not clip badly on sharp impulses
- Use automatic gain control only if it can be disabled or characterized

`96 kbps` is a compressed-stream rate, not a microphone sampling spec. For this problem, the important specs are:

- sample rate
- bit depth
- dynamic range
- microphone frequency response
- clipping behavior

## Minimal viable processing pipeline

### 1. Continuous trigger

Run a lightweight trigger on streaming audio:

- frame audio into short windows, for example 5-20 ms
- compute energy, peak, zero-crossing rate, and spectral flux
- flag candidate impulsive events when thresholds rise sharply above local background

This stage can be rule-based and may not need ML.

### 2. Event extraction

For each trigger:

- save a short window around the event, for example 100-300 ms
- normalize carefully without destroying relative amplitude cues
- optionally compute a noise estimate from the pre-event region

### 3. Feature generation

Start with compact features:

- log-mel spectrogram
- MFCCs
- spectral centroid
- spectral rolloff
- bandwidth
- crest factor
- attack time
- decay slope

These features are much more attainable on MCU-class hardware than large raw-audio models.

### 4. Classification

Start with the smallest model that solves the task:

- **Baseline**: gradient boosted trees / random forest on engineered features
- **Edge ML option**: tiny CNN on log-mel patches
- **Ultra-small option**: DS-CNN or 1D CNN with quantization

Recommended order:

1. rule-based trigger
2. classical ML classifier on extracted features
3. tiny neural network only if classical ML is not good enough

## Distance estimation strategy

Distance from a **single microphone** is fundamentally difficult because source loudness varies and the environment changes the signal.

What is attainable:

- **distance buckets** such as near / medium / far
- regression only after collecting calibrated data in known environments

Useful cues:

- direct-to-reverberant ratio
- peak-to-tail energy ratio
- spectral high-frequency loss
- shockwave vs muzzle-blast timing when geometry allows it

If distance matters a lot, a **small microphone array** is much stronger than a single microphone because it enables:

- time-difference-of-arrival features
- direction-of-arrival estimation
- more stable range heuristics

## Round / firearm type estimation strategy

Treat this as **hierarchical classification**, not exact brand-level identification:

1. gunshot vs non-gunshot
2. handgun vs rifle
3. suppressed vs unsuppressed
4. subsonic vs supersonic
5. optional finer class buckets if data supports it

This reduces model confusion and is more realistic for edge deployment.

## What to avoid early

- large end-to-end deep models
- exact caliber claims from small datasets
- training only on clean range audio
- heavy compression in the training data
- solving distance and exact round ID before basic detection is robust

## Data collection plan

Model quality will be limited mainly by data quality. Build the dataset around the deployment environment.

Capture:

- multiple firearms / ammo classes
- multiple distances
- multiple terrains and echo conditions
- wind, traffic, machinery, voices, fireworks, nail guns, slammed doors, cars backfiring
- different microphone orientations and mounting conditions

For every sample, label:

- event type
- estimated firearm class
- ammo class if known
- true measured distance
- environment
- microphone hardware
- gain setting

## Suggested R&D phases

### Phase 1: prove detection

- collect impulsive positives and hard negatives
- build rule-based trigger
- train a simple feature-based classifier
- target high recall with manageable false alarms

### Phase 2: add coarse classes

- handgun vs rifle
- suppressed vs unsuppressed
- near / mid / far

### Phase 3: optimize for edge

- quantize features or model weights
- prune model size
- benchmark latency, RAM, and energy use
- move the trigger to fixed-point if needed

### Phase 4: only then attempt finer identification

- limited round categories
- environment-specific calibration
- optional array processing if distance accuracy is still weak

## Edge deployment guidance

For a microcontroller target:

- use fixed-point or int8 inference where possible
- keep windows short and reuse buffers
- prefer streaming feature extraction
- store only event snippets, not full continuous audio

For a small FPGA target:

- implement the trigger and feature extraction in hardware
- offload classification to a tiny soft core or compact inference block
- prioritize deterministic low-latency preprocessing

## Practical first prototype

A strong first prototype is:

1. 48-96 kHz mono PCM audio
2. rule-based transient trigger
3. 150-250 ms event clip
4. log-mel + transient features
5. gradient boosted tree or tiny CNN
6. outputs:
   - gunshot probability
   - handgun vs rifle
   - suppressed vs unsuppressed
   - near / medium / far

That prototype is much more attainable than trying to infer exact round type from raw audio immediately.

## Success criteria

Track these metrics separately:

- gunshot detection recall
- false positives per hour
- classification accuracy by class
- distance-bucket accuracy
- latency on target hardware
- RAM / flash usage
- robustness across environments

## Recommended next steps

1. Lock the sensing hardware and sampling format.
2. Collect a small but well-labeled dataset with hard negatives.
3. Build a rule-based trigger and verify event capture quality.
4. Train a classical ML baseline on engineered features.
5. Add a tiny quantized CNN only if the baseline plateaus.
6. Treat distance and round type as coarse buckets first.
7. Re-evaluate whether a microphone array is needed for distance performance.