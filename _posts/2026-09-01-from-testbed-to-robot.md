---
layout: post
title: "An End-to-End Sim-to-Real Pipeline for Air Hockey"
eyebrow: "Robot Learning · Sim-to-Real"
author: "Da Liu"
description: "An end-to-end sim-to-real pipeline for robot air hockey"
date: 2026-09-01
permalink: /from-testbed-to-robot/
blog: true
image: /assets/media/autonomous-operation.jpg
---


Our work builds directly on **[Robot Air Hockey: A Manipulation Testbed for Robot Learning
with Reinforcement Learning](https://arxiv.org/abs/2405.03113)** (Chuck et al., ICRA
Workshop on Manipulation Skills, 2024), which introduced air hockey as an interactive RL
testbed and established most of the infrastructure this project started from.

Continued work focuses on a *Sim2Real pipeline*: where a policy trained in Box2D can be
dropped onto the arm and run and learn automatically.

## Updates

### 1. Sim2Real Capabilities

The original Box2D simulator, while supporting many tasks, was not designed to fit the dynamics of the real world.
We reworked the simulator around the control interface of the robot: the policy
emits a paddle displacement target, a low-pass filter smooths the displacements, a
PD controller inside the simulator tracks it, and the policy observes a short history of
paddle and puck positions.

<figure class="narrow">
  <video src="{{ '/assets/media/box2d-juggle.mp4' | relative_url }}"
         autoplay loop muted playsinline
         aria-label="Box2D juggling policy in the reworked simulator"></video>
  <figcaption>
    A juggling policy in the reworked Box2D environment. Actions are smoothed across timesteps and tracked by a PD
    controller, and the underlying dynamics were modified to match how the arm actually
    moves.
  </figcaption>
</figure>

We then fit that simulator to the real table by **system identification**. We grid-searched
the physics parameters against real trajectories, focusing on puck dynamics, the paddle (robot-arm) controller dynamics, and wall restitution for bounces.

Even after fitting, the simulator is still meaningfully different from the real table.
Box2D cannot be expected to re-enact the real world dynamics perfectly. We
bridge the remaining gap with **domain randomization**. The identified physics parameters
are sampled around their fitted values on every reset, alongside sensor-level noise,
latency, and occlusion, so the policy learns to adapt to a range of dynamics, given a short history as context.

After going through this training procedure, a policy trained in simulation can be
deployed directly onto the real robot.

<figure>
  <div class="figure-row">
    <video src="{{ '/assets/media/robot-juggle-1.mp4' | relative_url }}"
           autoplay loop muted playsinline
           aria-label="Simulation-trained juggling policy on the real robot, first example"></video>
    <video src="{{ '/assets/media/robot-juggle-2.mp4' | relative_url }}"
           autoplay loop muted playsinline
           aria-label="Simulation-trained juggling policy on the real robot, second example"></video>
  </div>
  <figcaption>
    Policies trained entirely in the calibrated, randomized simulator, applied directly to
    the robot with no real-world fine-tuning.
  </figcaption>
</figure>

### 2. Automated Pipeline

We developed an automatic reset policy that recovers the puck between episodes and
returns it to play. This enables automatic evaluation on hardware and
continued online learning, where a policy trained in simulation keeps improving on the
real table from data it collects itself.

<figure>
  <video src="{{ '/assets/media/autonomous-operation.mp4' | relative_url }}"
         poster="{{ '/assets/media/autonomous-operation.jpg' | relative_url }}"
         controls muted loop playsinline preload="metadata"></video>
  <figcaption>
    The robot running automatically. Partway through, the arm
    trips a safety stop — a human clears it and the run picks up from where it left off
    rather than starting over.
  </figcaption>
</figure>

Along the way, there were many improvements to the robustness of the robot system,
including but not limited to: a tightened control and perception loop with predictable
latency, a more reliable camera and puck-detection path, and structured handling of safety
events.

## Application

Our Sim2Real system allows plug-and-play for novel RL algorithms, and has already been used to support real-robot results. 

**[Learning Object Manipulation from Scratch via Contrastive Interaction](https://arxiv.org/abs/2606.11525)** (Shen et al., 2026), is a contrastive RL method that resamples experience around the moments where objects actually interact. For their experiments, the authors trained goal-conditioned policies in our simulator and deployed them to the arm through the sim-to-real pipeline.

<figure>
  <video src="{{ '/assets/media/crl-goal-reaching.mp4' | relative_url }}"
         poster="{{ '/assets/media/crl-goal-reaching.jpg' | relative_url }}"
         controls muted loop playsinline preload="metadata"></video>
  <figcaption>
    A goal-reaching task on the real table, comparing several algorithms brought to the
    robot through the pipeline.
  </figcaption>
</figure>

## Future work

With the system mostly in place, we plan on exploring more advanced Sim2Real approaches. As an example, we have considered 
Meta-RL extensions aimed at policies that identify and adapt to dynamics over longer contexts.

We also welcome collaborators who are interested in working on the robot or testing their algorithms.

## Conclusion

A researcher/practioner with an RL algorithm can train it against a simulator calibrated to a real
arm, deploy the result to that arm without writing hardware code, let it run automatically
through resets, recover from safety stops, and fine-tune it online if desired.
