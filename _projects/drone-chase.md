---
title: "Drone Chase — Finding and Tracking a Flying Target"
subtitle: "A follower quadrotor that finds, follows and re-acquires a leader drone using only what its camera can see, in a GenoM3 and Gazebo simulation"
description: "A follower quadrotor finds, tracks and re-acquires a moving leader drone from a simulated camera: FOV model, four-state mission logic and a P + feedforward controller on telekyb3/GenoM3 and Gazebo."
date: 2026-10-14
period: "Aug – Oct 2026"
badge: "Aerial robotics · Team project"
cover: /assets/images/projects/drone-chase/cover.jpg
hero: /assets/images/projects/drone-chase/hero.jpg
hero_alt: "Illustration of the follower drone, its camera field of view and the leader drone flying a figure-8"
hero_caption: "The follower (green) keeps the leader (orange) inside its camera's field of view while the leader flies a figure-8. Geometry from the project code; the field of view is drawn shorter than the real 10 m."
tags: [Python, GenoM3 / telekyb3, Gazebo, State machine, Visual tracking, Trajectory planning]
facts:
  - label: Framework
    value: telekyb3 / GenoM3 (LAAS-CNRS)
  - label: Simulator
    value: Gazebo, two quadrotors
  - label: Period
    value: Aug – Oct 2026
  - label: My focus
    value: Camera model, tracking and search
links:
  - label: Code on GitHub
    url: https://github.com/chettante/tk3lab-ws/tree/chetto/src/final_project
    icon: fab fa-github
  - label: telekyb3
    url: https://git.openrobots.org/projects/telekyb3
    icon: fas fa-arrow-up-right-from-square
team: [Enrico Castelli, Giuliano Livi, Matteo Giovanelli]
image: /assets/images/projects/drone-chase/cover.jpg
---

Two quadrotors share the sky. The **leader** flies its own trajectory; the **follower** has to find it,
stay close behind it and, when it slips out of view, find it again. The catch: the follower may only use
what an on-board **camera** would see. Everything runs in simulation, on the same flight software used on
real aerial robots: Gazebo for the physics, the telekyb3 flight stack (GenoM3 components from LAAS-CNRS)
for each drone, and our Python mission code on top.

I designed and wrote the follower's perception and control: the camera model, the estimation of the
leader's velocity, the tracking controller, the search strategy and their integration in the state machine.

<div class="stat-strip">
  <div><strong>4</strong><span>mission states</span></div>
  <div><strong>20 Hz</strong><span>mission control loop</span></div>
  <div><strong>87° × 58°</strong><span>camera field of view, 10 m range</span></div>
  <div><strong>3 m</strong><span>follow distance</span></div>
</div>

## How it is built

<figure class="figure figure-wide">
  <a href="{{ '/assets/images/projects/drone-chase/architecture.png' | relative_url }}" target="_blank" rel="noopener noreferrer"><img src="{{ '/assets/images/projects/drone-chase/architecture.png' | relative_url }}" alt="Architecture: Python client, GenoM3 components for leader and follower, Gazebo simulator" loading="lazy"></a>
  <figcaption>Our Python client drives two identical GenoM3 flight stacks, one per drone, connected to Gazebo.</figcaption>
</figure>

Each drone runs its own copy of the **telekyb3** stack. `rotorcraft` talks to the motors and the IMU,
`pom` is a Kalman filter that fuses the IMU with motion-capture poses, `nhfc` is the flight controller and
`maneuver` turns our commands into smooth, dynamically feasible references for it. The launcher starts two
instances of every component (`_1` for the leader, `_2` for the follower) and one shared `optitrack`.

On top of them, `main_statemachine.py` runs a **20 Hz loop** through `python-genomix`: it moves the leader
along its trajectory, simulates the follower's camera, steps the state machine and sends the resulting
velocity command to the follower's `maneuver`. All parameters live in a single `config.py`, and a small
script copies the field of view into the Gazebo camera sensor, so the image rendered by Gazebo and the
camera model in the code always match.

## A camera without image processing

<figure class="figure figure-wide">
  <a href="{{ '/assets/images/projects/drone-chase/camera-model.png' | relative_url }}" target="_blank" rel="noopener noreferrer"><img src="{{ '/assets/images/projects/drone-chase/camera-model.png' | relative_url }}" alt="Camera field of view: top view with 87 degrees horizontal, side view with 58 degrees vertical and the effect of tilting" loading="lazy"></a>
  <figcaption>The follower only gets the leader's position while it is inside this pyramid.</figcaption>
</figure>

The leader's position comes from motion capture, but the follower is allowed to use it **only while the
leader is inside the camera's field of view**: a pyramid of 87° × 58°, from 0.1 to 10 m, mounted 45° off
the drone's first arm. This turns perfect data into the realistic problem: the leader can be lost, and the
follower has to keep it in the picture.

The camera is **rigid**, as in Gazebo. To accelerate, a quadrotor has to tilt, and every degree of tilt
moves the leader by one degree in the image. So the tracking acceleration is a *tilt budget*:
a = g · tan(10°) ≈ 1.7 m/s².

<div class="callout" markdown="1">
**Estimating the leader's velocity.** The camera only gives positions. The velocity comes from
differentiating the leader's position between frames, computed in the world frame (the camera frame rotates
with the drone), restarted every time the leader re-enters the image and frozen when motion capture has
not refreshed yet, which would otherwise make the estimate jump between 0 and twice the real value.
</div>

<figure class="figure figure-wide">
  <a href="{{ '/assets/images/projects/drone-chase/velocity-estimate.png' | relative_url }}" target="_blank" rel="noopener noreferrer"><img src="{{ '/assets/images/projects/drone-chase/velocity-estimate.png' | relative_url }}" alt="Leader velocity estimated from the camera compared with the ground truth on the three axes" loading="lazy"></a>
  <figcaption>Validation run with a slow leader (below 1 m/s): the estimate follows the ground truth with a median error of about 0.01 m/s.</figcaption>
</figure>

## Four states

<figure class="figure figure-wide">
  <a href="{{ '/assets/images/projects/drone-chase/state-machine.png' | relative_url }}" target="_blank" rel="noopener noreferrer"><img src="{{ '/assets/images/projects/drone-chase/state-machine.png' | relative_url }}" alt="State machine with IDLE, EXPLORING, TRACKING and SEARCHING and their transitions" loading="lazy"></a>
  <figcaption>The mission logic. Every state checks the camera at each tick.</figcaption>
</figure>

- **IDLE** — hover and look. If the leader appears, start tracking; after 5 s without it, explore.
- **EXPLORING** — one lap on a ring around the search volume, with the camera always facing its centre.
- **TRACKING** — follow the leader and keep it centred. If it leaves the image, keep following its
  extrapolated position for up to 2 s before giving up.
- **SEARCHING** — fly to where the leader should be now, then scan the area; back to tracking as soon as
  it is seen again.

Each state is a small class whose `update()` returns the next state, so the transitions stay explicit and
every state can be developed and tested on its own.

## Tracking: P + feedforward

<figure class="figure figure-wide">
  <a href="{{ '/assets/images/projects/drone-chase/tracking-law.png' | relative_url }}" target="_blank" rel="noopener noreferrer"><img src="{{ '/assets/images/projects/drone-chase/tracking-law.png' | relative_url }}" alt="Tracking control law: geometry and the five steps of one control cycle" loading="lazy"></a>
  <figcaption>One control step of the tracking controller.</figcaption>
</figure>

The follower aims for a point **3 m behind the leader** along the line that joins them, at the leader's
altitude. The command is the leader's estimated velocity (feedforward) plus a proportional correction
toward that point, and the yaw keeps the leader on the camera axis using both the angular error and the
rate at which the bearing is turning.

Two rules keep the leader in the picture. Near the side edges of the image the drone **rotates first** and
corrects its position later; near the top or bottom edge it **stops approaching**, because getting closer
increases the vertical angle. A rate limiter enforces the tilt budget, and when tracking starts the limiter
begins from the follower's measured velocity instead of zero (*bumpless transfer*), so switching state
never makes the drone jerk.

## Losing the leader, and finding it again

<figure class="figure figure-wide">
  <a href="{{ '/assets/images/projects/drone-chase/search-explore.png' | relative_url }}" target="_blank" rel="noopener noreferrer"><img src="{{ '/assets/images/projects/drone-chase/search-explore.png' | relative_url }}" alt="Search after losing the leader and exploration of the volume when no leader has been seen" loading="lazy"></a>
  <figcaption>Searching starts from the last estimate of the leader; exploring covers the volume when there is none.</figcaption>
</figure>

**Searching** starts where the leader should be by now: its last position plus its last velocity times
the time since it was lost. In a single smooth move the follower turns to put that point at the centre of
the image, gets within 3 m of it and matches its altitude. If the leader is still not in view, a fixed scan
follows: hold, fly home, climb while spinning, descend while spinning.

**Exploring** is for when the leader has not been seen at all: a straight connection to a ring that
encloses the 20 × 20 m volume, then one lap at 10 m/s, rising from 6 to 14 m and back, with the camera
pointed at the centre. Point-to-point moves use a quintic (smoothstep) time law sized on the speed and
acceleration limits, so they start and stop with zero velocity and acceleration.

## The leader

<figure class="figure figure-wide">
  <a href="{{ '/assets/images/projects/drone-chase/leader-figure8.png' | relative_url }}" target="_blank" rel="noopener noreferrer"><img src="{{ '/assets/images/projects/drone-chase/leader-figure8.png' | relative_url }}" alt="The leader's figure-8 reference trajectory in 3D, top view and altitude" loading="lazy"></a>
  <figcaption>The figure-8 reference flown by the leader.</figcaption>
</figure>

The leader flies a **figure-8** that climbs from 5 to 10 m and back. It is re-parametrised by arc length,
so the speed stays constant along the curves after a smooth start, and the whole trajectory is computed
once before the loop and then streamed to `maneuver` one sample per tick. For harder tests the leader can
instead fly through **100 random waypoints** spread over a 200 × 200 m area, at speeds of tens of metres
per second.

## What I learned

- **The planner has limits, and they must be respected upstream.** Rate-limiting every command and never
  sending a non-finite value avoids the planner rejecting or crashing on a sudden change.
- **Switching states is part of the controller.** Bumpless transfer and a short grace period stop the
  drone from jerking at every transition and from giving up on a leader that is only briefly out of view.
- **Perception constraints shape the control.** With a rigid camera, the field of view decides how hard the
  follower may accelerate and in which order it should correct yaw and position.
- **One configuration file for everything.** Keeping all gains, limits and the camera geometry in one
  place, and generating the Gazebo camera from it, avoids mismatches between the code and the simulator.
