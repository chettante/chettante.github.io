---
title: "AI-ARMS — Learning Pick-and-Place on the SO-101 Arm"
subtitle: "From a 3D-printed open-source arm to an autonomous pick-and-place policy trained by imitation learning"
description: "We printed and assembled an SO-101 arm, teleoperated it leader–follower, and trained an ACT policy with LeRobot to pick and place objects without a human in the loop."
date: 2026-05-06
period: "Oct 2025 – present"
badge: "Student team · AEA"
cover: /assets/images/projects/so101/cover.jpg
hero: /assets/images/projects/so101/hero.jpg
hero_alt: "SO-101 follower arm on the lab bench next to an orange cube and a laptop showing the camera feeds"
hero_caption: "The SO-101 follower arm on the bench, with the camera feeds on the laptop."
tags: [LeRobot, Imitation learning, ACT policy, Teleoperation, Hugging Face, 3D printing]
facts:
  - label: Team
    value: AI-ARMS, Automation Engineering Association
  - label: Period
    value: Oct 2025 – present
  - label: Platform
    value: SO-101 leader + follower arms
  - label: My focus
    value: Training the autonomous pick-and-place policy
links:
  - label: Project page (AEA)
    url: https://www.aeapolimi.it/progetti/armsAI
    icon: fas fa-arrow-up-right-from-square
  - label: Commands guide
    url: https://github.com/AEA-LeRobot/AEA-lerobot-commands
    icon: fab fa-github
  - label: Datasets & models
    url: https://huggingface.co/SOARM1
    icon: fas fa-database
image: /assets/images/projects/so101/cover.jpg
---

**AI-ARMS** is a team of the Automation Engineering Association (AEA) at Politecnico di Milano working
on low-cost robot manipulation. We built an **SO-101** arm from scratch, made it work under
**leader–follower teleoperation**, and then used the demonstrations to teach it a task on its own.

My part was the last step: training an **imitation-learning policy** on the recorded demonstrations and
running it on the real arm, so that it performs **pick-and-place autonomously, with no teleoperation**.

<div class="stat-strip">
  <div><strong>6</strong><span>servo motors per arm</span></div>
  <div><strong>50</strong><span>demonstration episodes</span></div>
  <div><strong>~19.8k</strong><span>frames recorded at 30 fps</span></div>
  <div><strong>2</strong><span>RGB cameras, 640×480</span></div>
</div>

## The SO-101 arm

<div class="split" markdown="1">
<div markdown="1">
The SO-101 is an **open-source, 3D-printable robot arm** designed by TheRobotStudio together with
Hugging Face. It is the successor of the SO-100, with easier assembly and better wiring, and it has become
one of the most popular platforms for learning-based manipulation because:

- every structural part is **printed in PLA**, and the design files are released under Apache-2.0;
- the parts for a full leader + follower pair cost **a few hundred euros**;
- it is natively supported by **LeRobot**, Hugging Face's library for real-world robot learning.

A setup has **two arms**. The **follower** does the work; the **leader**, with a handle and trigger,
is moved by hand to drive the follower. Both use Feetech STS3215 serial-bus servos, with different gear
ratios on the leader to make it easy to back-drive.
</div>
<figure class="figure">
  <img src="{{ '/assets/images/projects/so101/arm-closeup.jpg' | relative_url }}" alt="Close-up of the assembled SO-101 arm with its servo motors" loading="lazy">
  <figcaption>Our assembled arm: 3D-printed links and serial-bus servos.</figcaption>
</figure>
</div>

## Build and teleoperation

<div class="split split-reverse" markdown="1">
<div markdown="1">
We **printed and assembled both arms**, then brought them up with LeRobot's command-line tools:

1. **Motor setup** — assign an ID to every servo on the bus.
2. **Calibration** — record the joint ranges of each arm, so leader and follower agree on every angle.
3. **Teleoperation** — the follower mirrors the leader in real time, while the two cameras stream to the PC.

Teleoperation is not only a demo: it is how the robot **learns**. Every movement we make with the
leader can be recorded as an example of the task.
</div>
<figure class="figure">
  <img src="{{ '/assets/images/projects/so101/leader-arm.jpg' | relative_url }}" alt="One of the assembled SO-101 arms on the lab bench" loading="lazy">
  <figcaption>One of our arms on the bench during bring-up and calibration.</figcaption>
</figure>
</div>

## Teaching the robot by demonstration

The goal was to remove the human: the robot should see the scene and pick up and place the object by
itself, following the LeRobot imitation-learning workflow.

<ol class="steps">
  <li><h3>Record demonstrations</h3><p>The task is performed many times through teleoperation. Each episode stores the two camera streams, the six joint positions and the actions sent to the follower, at 30 fps.</p></li>
  <li><h3>Build the dataset</h3><p>Episodes are uploaded to the Hugging Face Hub and merged into one training set: 50 episodes and about 19,800 frames over 5 task variants.</p></li>
  <li><h3>Check the data</h3><p>Before training, episodes are replayed with the LeRobot visualizer to catch bad recordings, wrong motions or desynchronized cameras.</p></li>
  <li><h3>Train an ACT policy</h3><p>The policy is trained on a GPU in Google Colab and pushed back to the Hub, so the same model can be loaded on any machine.</p></li>
  <li><h3>Run it on the robot</h3><p>The trained policy drives the follower directly from the camera images and joint states, with no leader arm connected.</p></li>
</ol>

<div class="callout" markdown="1">
**Why ACT.** *Action Chunking with Transformers* is the imitation-learning policy LeRobot suggests
starting with. A ResNet backbone reads the camera images, a transformer combines them with the joint
positions, and the network predicts a **chunk of future actions** at once instead of a single step. This
makes the motion smoother and more robust, and good results are typically reached with about 50
demonstrations — the size of our dataset.
</div>

A training run on the merged dataset, as launched in Colab:

```bash
python src/lerobot/scripts/lerobot_train.py \
  --dataset.repo_id=SOARM1/SOARM101_merged \
  --policy.type=act \
  --policy.device=cuda \
  --output_dir=outputs/train/hf_act_record0 \
  --policy.repo_id=SOARM1/hf_act_recordpolicy0
```

In the team we also tuned the chunking, for example predicting 50 steps and executing 10 before
re-planning, which gave smoother movements on the real arm.

## Pick-and-place in the workcell

<div class="split" markdown="1">
<figure class="figure video-figure">
  <video controls muted loop playsinline preload="metadata" poster="{{ '/assets/images/projects/so101/pick-place-poster.jpg' | relative_url }}">
    <source src="{{ '/assets/images/projects/so101/pick-place.mp4' | relative_url }}" type="video/mp4">
  </video>
  <figcaption>Pick-and-place run in the workcell (video at 2× speed).</figcaption>
</figure>
<div markdown="1">
The workcell is deliberately simple: a cardboard enclosure with tape marking the work area, an
object to grasp and a box to drop it into. Keeping the background and lighting **consistent between
recording and deployment** matters, because the policy learns directly from pixels.

Two practical points of this workflow:

- **The setup is part of the model.** Cameras, lighting and object positions should stay the same
  between recording and deployment.
- **Everything is reproducible.** Datasets and trained policies live on the Hugging Face Hub, so anyone
  in the team can retrain or run a model on their own laptop.
</div>
</div>

## Resources

- [AI-ARMS project page](https://www.aeapolimi.it/progetti/armsAI){:target="_blank"} — Automation Engineering Association, PoliMi
- [Team commands guide](https://github.com/AEA-LeRobot/AEA-lerobot-commands){:target="_blank"} — setup, calibration, recording, training and deployment
- [Datasets and trained policies](https://huggingface.co/SOARM1){:target="_blank"} on Hugging Face
- [LeRobot](https://github.com/huggingface/lerobot){:target="_blank"} and the [SO-101 hardware repository](https://github.com/TheRobotStudio/SO-ARM100){:target="_blank"}
