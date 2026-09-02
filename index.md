---
layout: post
title: "From Testbed to Robot: An End-to-End Sim-to-Real Pipeline for Air Hockey"
date: 2026-08-30
image: /assets/media/autonomous-operation.jpg
---

Air hockey is a good stress test for robot learning. Motions need to be fast, well-timed,
and precise, while still requiring high-level planning for specific tasks.

This post describes the current state of our air hockey stack: a UR5e arm over a real
table, a Box2D simulator calibrated against it, and the software that connects the two.
The system supports many forms of operation — human teleoperation and demonstration
collection, behavior cloning and offline learning from that data, reinforcement learning
in simulation, and a sim-to-real pipeline that carries a trained policy onto the arm,
evaluates it there automatically, and keeps improving it against the real table. None of
it is tied to one learning algorithm, and several projects beyond our own have already
been built on it.

## What came before

Our work builds directly on **[Robot Air Hockey: A Manipulation Testbed for Robot Learning
with Reinforcement Learning](https://arxiv.org/abs/2405.03113)** (Chuck et al., 2024),
which introduced air hockey as an interactive RL testbed and established most of the
infrastructure this project started from.

**A task family spanning simulation and hardware.** The testbed defines ten tasks over the
same table, paddle, and puck, graded from ones behavior cloning can solve to ones humans
struggle with on the real robot: reaching a position, reaching a position at a target
velocity, touching the puck, striking a stationary puck, striking a puck into a crowd of
blocks, juggling, hitting the puck to a minimum upward velocity, moving a block by hitting
it with the puck, and hitting the puck into a goal region — with and without a desired
arrival velocity. Ten run in Box2D, six in Robosuite, and five on the real robot, with the
table geometry held consistent across all three so that a policy can in principle move
between them.

**Two simulators of increasing fidelity, plus the real system.** A 2D Box2D environment
where the paddle is manipulated directly, fast enough for high-volume interaction and
exposing physics parameters — masses, damping, friction, gravity, initial puck velocities
— as knobs. A 3D Robosuite/MuJoCo environment that actually simulates the arm and its
controller. And the physical setup: a UR5e over a tilted air hockey table, with an
overhead camera doing puck detection and an RTDE controller turning task-space actions
into joint commands.

**A teleoperation system, and the data it produced.** Two ways to get human behavior onto
the table — a virtualized control interface a person drives directly, and human shadowing,
where a person plays and the arm follows what they do. These produced a dataset of 350
mouse-teleoperation and 50 shadow-teleoperation trajectories from eight participants of
varying skill, which is what makes the imitation and offline-RL side of the testbed
possible at all.

**Baselines across algorithm families.** Behavior cloning, offline RL, and RL from
scratch, evaluated on the same tasks across all three domains, so a new method has
something to be measured against. The headline finding was that online interaction matters
in this domain: RL outperformed imitation in simulation, while on the real robot every
baseline fell short of human play — and RL from scratch was impractical there, because the
random exploration it needs wears the arm down.

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

## Applications

None of this is specific to our learning algorithm. The simulator is a standard RL
environment defined by a single config, so an external training loop can train against
exactly the physics, observations, and rewards our own trainer uses. The real-robot side is
separated along the same seam: the deployment and evaluation harness treats the policy as
an interface, so bringing a new algorithm to the physical table means supplying an actor,
not writing hardware code. Train in sim, hand over a policy, run it on the arm.

Projects have entered the system at several different points.

### Contrastive RL, trained in sim and run on the arm

The sim-to-real path described above supported the real-robot results in **[Learning Object
Manipulation from Scratch via Contrastive Interaction](https://arxiv.org/abs/2606.11525)**
(Shen et al.), a contrastive RL method developed independently of ours. It is the clearest
test of the pipeline as a product: an outside algorithm, trained against our simulator and
deployed to our arm without touching hardware code.

<figure>
  <video src="{{ '/assets/media/crl-goal-reaching.mp4' | relative_url }}"
         poster="{{ '/assets/media/crl-goal-reaching.jpg' | relative_url }}"
         controls muted loop playsinline preload="metadata"></video>
  <figcaption>
    A goal-reaching task run on the real table, comparing several learning algorithms
    brought to the robot through the same pipeline.
  </figcaption>
</figure>

### Imitation from observation, on the real table

**[A Dual Approach to Imitation Learning from Observations with Offline
Datasets](https://arxiv.org/abs/2406.08805)** (Sikchi et al., CoRL 2024) used the physical
setup and its teleoperation system as the hard case for learning from action-free
demonstrations. Where most of the benchmark suites in that paper are simulated, the air
hockey table supplied something they could not: a task whose inverse dynamics are genuinely
difficult, since hitting a moving puck requires a wide range of actions from a state space
that human demonstrations only sparsely cover.

<figure>
  <img src="{{ '/assets/media/dilo-puck-hitting.jpg' | relative_url }}"
       alt="Three overhead views of the air hockey table comparing BCO, SMODICE, and DILO on dynamic puck hitting; the puck's path is traced by a fading red trail.">
  <figcaption>
    Dynamic puck hitting on the real table, from Sikchi et al. The red trail traces the
    puck over time. BCO and SMODICE let it run past the paddle and down the side; DILO
    intercepts and returns it up the table. Demonstrations for all three came from the
    teleoperation system.
  </figcaption>
</figure>

### Hindsight relabeling in the Box2D environment

**[Null Counterfactual Factor Interactions for Goal-Conditioned Reinforcement
Learning](https://arxiv.org/abs/2505.03172)** (Chuck et al., ICLR 2025) used the Box2D
environment as one of its evaluation domains, and for a specific reason: air hockey breaks
naive hindsight relabeling. Relabeling whatever puck position a trajectory happened to
reach as the intended goal ends up rewarding the episodes where the puck drifts back to the
agent's own end — that is, rewarding the policy for missing. Their method filters
relabeled goals down to those that follow an actual paddle–puck interaction, and reports
roughly 4× better sample efficiency on the domain as a result.

<figure>
  <img src="{{ '/assets/media/hint-rollout.png' | relative_url }}"
       alt="Six frames of a Box2D air hockey rollout: the blue paddle rises to meet the red puck, strikes it, and the puck travels up into the green goal region.">
  <figcaption>
    A goal-reaching rollout in the Box2D environment, from Chuck et al. The paddle (blue)
    intercepts the falling puck (red) and sends it into the goal region (green). The same
    simulator underlies the sim-to-real work above.
  </figcaption>
</figure>

We are also extending the transfer ourselves: **meta-RL extensions to sim-to-real
adaptation** are in progress, aimed at policies that identify and adapt to the dynamics
over longer contexts.

## Where this leaves us

A researcher with a new RL algorithm can train it against a simulator calibrated to a real
arm, deploy the result to that arm without writing hardware code, let it run automatically
through resets and recover from safety stops, and fine-tune it online against whatever gap
remains.
