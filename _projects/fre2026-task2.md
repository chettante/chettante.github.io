---
title: "Field Robot Event 2026 — Plant Health Detection"
subtitle: "Marker detection and semantic Graph SLAM for an autonomous robot in a curved maize field"
description: "Autonomous detection and mapping of diseased plants in a maize field: HSV histogram back-projection, 3D validation and semantic Graph SLAM in ROS 2."
date: 2026-06-18
badge: "Competition · AIRLab"
cover: /assets/images/projects/fre2026/cover.jpg
hero: /assets/images/projects/fre2026/hero.jpg
hero_alt: "The Grasslammer robot driving between maize rows"
hero_caption: "Grasslammer starting a run between the maize rows, with the LED matrix mounted on the back."
tags: [ROS 2, OpenCV, Graph SLAM, GTSAM / iSAM2, Kalman filter, OAK-D depth]
facts:
  - label: Event
    value: 23rd Field Robot Event
  - label: Where & when
    value: Bernburg (DE), 16–18 June 2026
  - label: Team
    value: AIRLab, Politecnico di Milano
  - label: My focus
    value: Marker detection & localization
links:
  - label: Read the report
    url: /assets/docs/FRE2026_Task2_Report.pdf
    icon: fas fa-file-pdf
  - label: Field Robot Event
    url: https://www.fieldrobotevent.eu/
    icon: fas fa-arrow-up-right-from-square
team: [Simone Edmondo Bedini, Enrico Castelli, Mirko Usuelli, Leonardo Gargani]
image: /assets/images/projects/fre2026/cover.jpg
---

The [Field Robot Event](https://www.fieldrobotevent.eu/){:target="_blank"} is an international competition
for autonomous robots in real agricultural fields. In **Task 2** the robot drives on its own through
several curved rows of maize in three minutes, recognises the plants marked as "diseased" and reports
each one: a public visual signal on the spot, plus a CSV log with the row number and the distance from
the start of the row, within **10 cm**.

I worked on the perception and localization side of the system: detecting the markers reliably outdoors
and placing them on a map that does not drift.

<div class="stat-strip">
  <div><strong>3 min</strong><span>per run, fully autonomous</span></div>
  <div><strong>40 cm</strong><span>window to signal each plant</span></div>
  <div><strong>10 cm</strong><span>required mapping precision</span></div>
  <div><strong>2</strong><span>marker classes: yellow &amp; jute</span></div>
</div>

## The challenge

<div class="split" markdown="1">
<div markdown="1">
Diseased plants are simulated by rectangular markers made of **yellow paper or jute**, placed next to a
plant. The task breaks down into three problems:

1. **Perception** — find the markers under changing outdoor light, partly hidden by leaves.
2. **Localization & mapping** — measure where each one is along a *curved* row, where wheel odometry
   alone drifts on loose soil.
3. **Real-time feedback** — light up a signal while the marker is still inside the 40 cm zone.
</div>
<figure class="figure">
  <img src="{{ '/assets/images/projects/fre2026/field-layout.png' | relative_url }}" alt="Layout of the curved maize field used in Task 2" loading="lazy">
  <figcaption>Task 2 field: curved rows 0.75 m apart, start and finish on the headland.</figcaption>
</figure>
</div>

## The robot

<div class="split split-reverse" markdown="1">
<div markdown="1">
**Grasslammer** is AIRLab's four-wheeled field platform. For this task the relevant sensors were:

- two **OAK-D** RGB-D cameras on the sides, one per row of plants;
- two **SICK** single-plane LiDARs, front and rear;
- a downward-facing **ZED** stereo camera;
- a custom **LED matrix** on the back for the public signal.
</div>
<figure class="figure">
  <img src="{{ '/assets/images/projects/fre2026/grasslammer.jpg' | relative_url }}" alt="The Grasslammer robot during field tests" loading="lazy">
  <figcaption>Grasslammer during field tests.</figcaption>
</figure>
</div>

## Perception: from pixels to 3D markers

A ROS 2 detection node turns every camera frame into validated 3D marker positions.

<figure class="figure figure-wide">
  <img src="{{ '/assets/images/projects/fre2026/detection-node.png' | relative_url }}" alt="State diagram of the detection node" loading="lazy">
  <figcaption>Detection node: process the image, validate the mask, publish the detection pose.</figcaption>
</figure>

<ol class="steps">
  <li><h3>Calibrate on site</h3><p>Before each run, a region of interest is placed over a real marker (via ROS 2 dynamic parameters) and a service call stores its Hue–Saturation histogram, so the detector matches the light of that moment.</p></li>
  <li><h3>Histogram back-projection</h3><p>Each frame is converted to HSV, which separates colour from brightness and copes better with shadows. Back-projecting the stored histogram gives a probability map of "marker-coloured" pixels.</p></li>
  <li><h3>Clean the mask</h3><p>A threshold turns the map into a binary mask; morphological opening removes isolated noise and closing fills the gaps inside the marker.</p></li>
  <li><h3>Check the shape</h3><p>Contours that are too small are dropped. The rest must pass aspect-ratio and extent tests, which separate rectangular markers from irregular blobs.</p></li>
  <li><h3>Go to 3D</h3><p>The median of the valid aligned depth pixels and the camera intrinsics project the marker centre into 3D. Only markers inside the 40 cm lateral zone are passed on, in the robot frame.</p></li>
</ol>

<figure class="figure figure-mid">
  <img src="{{ '/assets/images/projects/fre2026/detection-mask.jpg' | relative_url }}" alt="Detected jute marker and its binary mask" loading="lazy">
  <figcaption>A jute marker detected in the image (left) and the cleaned binary mask (right).</figcaption>
</figure>

## Tracking and mapping

Raw detections are noisy: the same marker is seen in many frames, and reflections can look like markers.
Before reaching the map they go through a **multi-object tracker** from the open-source
[orchards-semantic-slam](https://github.com/wur-abe/orchards-semantic-slam){:target="_blank"} library:

- a cost matrix combines **Euclidean distance** (with a gating threshold) and **semantic class**
  (yellow vs jute), and the **Hungarian algorithm** finds the best assignment;
- each track is smoothed by a **Kalman filter** and confirmed only after at least two hits.

Confirmed markers then enter a **semantic factor graph** solved with **GTSAM / iSAM2**. Odometry adds a
new pose only after a minimum displacement, and each observation links a pose to a landmark through
a range-bearing factor.

<figure class="figure figure-wide">
  <img src="{{ '/assets/images/projects/fre2026/mapping-node.png' | relative_url }}" alt="State diagram of the localization and mapping node" loading="lazy">
  <figcaption>Mapping node: one graph per row, optimized and exported to CSV on exit.</figcaption>
</figure>

<div class="callout" markdown="1">
**Anchoring both ends of the row.** Only the distance from the row start matters, so a fresh graph is
started at every row entry, which also discards the drift from the headland turns. A prior fixes the first
pose at the origin; on exit, a second prior fixes the final position to the known row length. A last
optimization then spreads the accumulated drift back over the whole trajectory.
</div>

<figure class="figure figure-wide">
  <img src="{{ '/assets/images/projects/fre2026/factor-graph.png' | relative_url }}" alt="Optimized factor graph with robot poses and marker positions" loading="lazy">
  <figcaption>Optimized trajectory (blue) and the estimated positions of yellow and jute markers along one row.</figcaption>
</figure>

## Real-time signal

The LED matrix is driven by the tracker's confirmation logic, not by single frames, so sensor noise does
not cause false alarms. When a plant is confirmed, an arrow points left or right, towards the camera
that saw it.

## Results and lessons learned

The perception layer was **robust against false positives** in real outdoor conditions. The field showed
two clear limits, which point to the next steps:

- **Calibration was too narrow.** A histogram taken in one light and orientation sometimes missed markers
  seen at other angles. Combining several histograms, or widening the stored H–S range, should help.
- **Mapping missed the 10 cm target.** The Graph SLAM pipeline worked end to end, but needs more
  tuning of the graph noise models in live tests to reach the required precision.
