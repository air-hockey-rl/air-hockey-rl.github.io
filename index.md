---
layout: post
title: "From Testbed to Robot: An End-to-End Sim-to-Real Pipeline for Air Hockey"
subtitle: "Rebuilding the Box2D air hockey environment around real arm dynamics, and closing the loop with online RL on hardware"
date: 2026-08-30
image: /assets/media/autonomous-operation.jpg
---

Air hockey is a good stress test for robot learning. Motions need to be fast, well-timed,
and precise, while still requiring high-level planning for specific tasks.

This post describes the current state of our air hockey stack.

## What came before

Our work builds directly on **[Robot Air Hockey: A Manipulation Testbed for Robot
Learning with Reinforcement Learning](https://arxiv.org/abs/2405.03113)** (Chuck et al.,
2024). That paper introduced air hockey as an interactive RL testbed spanning a family of
tasks — from simple reaching, through striking and goal-scoring, up to pushing a block by
hitting it with a puck and human-interactive play — across two simulators of differing
fidelity and a physical robot.

It contributed the pieces this project started from: a Box2D air hockey simulator, the
task family, real-world data collection via teleoperation and human shadowing, and
baseline results for behavior cloning, offline RL, and RL from scratch.

Continued work focuses on a *pipeline*: a path where a policy trained in Box2D can be
dropped onto the arm and run and learn automatically.

## What we rebuilt

### 1. Sim-to-real capabilities

The Box2D simulator was reworked around the control interface of the robot: the policy
emits a paddle displacement target, a low-pass filter smooths the displacements, a
controller inside the simulator tracks it, and the policy observes a short history of
paddle and puck positions.

<figure class="narrow">
  <video src="{{ '/assets/media/box2d-juggle.mp4' | relative_url }}"
         autoplay loop muted playsinline
         aria-label="Box2D juggling policy in the reworked simulator"></video>
  <figcaption>
    A juggling policy in the reworked Box2D environment. The paddle no longer teleports to
    a commanded position: actions are smoothed across timesteps and tracked by a PD
    controller, and the underlying dynamics were modified to match how the arm actually
    moves.
  </figcaption>
</figure>

We then fit that simulator to the real table by **system identification**, grid-searching
the physics parameters against recorded real trajectories: puck dynamics on the tilted
surface, the paddle controller's response, and wall restitution from real bounce segments.

Even after fitting, the simulator is still meaningfully different from the real table.
Contact is the hardest part, and no amount of parameter tuning closes it entirely. We
bridge the remaining gap with **domain randomization** — the identified physics parameters
are resampled around their fitted values on every reset, alongside sensor-level noise,
latency, and occlusion, so the policy learns to adapt to a range of dynamics.

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

### 2. A pipeline that runs on its own

We developed an **automatic reset policy** that recovers the puck between episodes and
returns it to play. This enables **automatic evaluation** on hardware — deploy a
checkpoint, let it run a fixed batch of episodes, read the numbers off the other end — and
**continued online learning**, where a policy trained in simulation keeps improving on the
real table from data it collects itself.

<figure>
  <video src="{{ '/assets/media/autonomous-operation.mp4' | relative_url }}"
         poster="{{ '/assets/media/autonomous-operation.jpg' | relative_url }}"
         controls muted loop playsinline preload="metadata"></video>
  <figcaption>
    The robot running automatically: episodes go back to back, and the reset policy
    recovers the puck between them without a human at the table. Partway through, the arm
    trips a safety stop — a human clears it and the run picks up from where it left off
    rather than starting over.
  </figcaption>
</figure>

Along the way, there were many improvements to the robustness of the robot system,
including but not limited to: a tightened control and perception loop with predictable
latency, a more reliable camera and puck-detection path, and structured handling of safety
events.

### 3. Plug and play for other algorithms

None of this is specific to our learning algorithm. The simulator is a standard RL
environment defined by a single config, so an external training loop can train against
exactly the physics, observations, and rewards our own trainer uses. The real-robot side
is separated along the same seam: the deployment and evaluation harness treats the policy
as an interface, so bringing a new algorithm to the physical table means supplying an
actor, not writing hardware code. Train in sim, hand over a policy, run it on the arm.

This has already been used outside the project — the sim-to-real path here supported the
real-robot results in **[Learning Object Manipulation from Scratch via Contrastive
Interaction](https://arxiv.org/abs/2606.11525)** (Shen et al.), a contrastive RL method
developed independently of ours.

<figure>
  <video src="{{ '/assets/media/crl-goal-reaching.mp4' | relative_url }}"
         poster="{{ '/assets/media/crl-goal-reaching.jpg' | relative_url }}"
         controls muted loop playsinline preload="metadata"></video>
  <figcaption>
    A goal-reaching task run on the real table, comparing several learning algorithms
    brought to the robot through the same pipeline.
  </figcaption>
</figure>

We are also extending the transfer: **meta-RL extensions to sim-to-real adaptation** are
in progress, aimed at policies that identify and adapt to the dynamics over longer
contexts.

## Where this leaves us

A researcher with a new RL algorithm can train it against a simulator calibrated to a real
arm, deploy the result to that arm without writing hardware code, let it run automatically
through resets and recover from safety stops, and fine-tune it online against whatever gap
remains.
