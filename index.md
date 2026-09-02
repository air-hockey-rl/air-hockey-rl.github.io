---
layout: post
title: "Robot Air Hockey"
eyebrow: "Robot Learning · Manipulation Testbed"
home: true
image: /assets/media/overview.jpg
---

Air hockey is a good test for dynamic robot learning. Across tasks, motions need to be
fast, well-timed, and precise. For harder tasks, policies need high-level planning
capabilities.

Robot Air Hockey is a testbed built around a family of such tasks that runs across two
simulators and a real robot arm. On top of it sit teleoperation, imitation/offline
learning, reinforcement learning, and a sim-to-real pipeline. This post is about the
current state of the air hockey system.

The testbed is described in **[Robot Air Hockey: A Manipulation Testbed for Robot Learning
with Reinforcement Learning](https://arxiv.org/abs/2405.03113)** (Chuck et al., ICRA
Workshop on Manipulation Skills, 2024). The sim-to-real pipeline is described in
**[An End-to-End Sim-to-Real Pipeline for Air
Hockey]({{ '/from-testbed-to-robot/' | relative_url }})**.

<figure>
  <img src="{{ '/assets/media/overview.jpg' | relative_url }}"
       alt="Left: a 2D Box2D air hockey rink and a 3D Robosuite scene with a robot arm at a table. Right: an overhead photo of the real air hockey table with a UR5e arm, pucks, and blocks">
  <figcaption>
    The setup in two simulators and on real hardware.
  </figcaption>
</figure>

## Setup

The system runs in three domains of decreasing fidelity. All three share a state space, an
action space, and reward functions.

### Real robot

In the real world, we have a UR5e arm (operational-space control, 20 Hz) holding a paddle
over a 76 × 34 in air hockey table. We use a PlayStation Eye camera to capture the state.

<figure>
  <img src="{{ '/assets/media/real-setup.jpg' | relative_url }}"
       alt="The physical setup, labelled: a top-down camera above a tilted air hockey table, a UR5e arm holding a paddle, and a red puck, with coloured arrows showing reach, touch, and hit motions.">
  <figcaption>
    The real setup. A top-down camera provides the observation and the arm actuates the
    paddle.
  </figcaption>
</figure>

### Robosuite / MuJoCo

We have a Robosuite environment that simulates the arm in 3D, using an operational-space
controller modified to hold the same contact with the table.

### Box2D

We have a simpler Box2D simulator that moves the paddle directly in 2D and allows quick
modifications of system physics. The environment is calibrated against the physical setup
in system structure and in physics parameters by **system identification**. We also
introduce sensor noise, latency, and occlusion.

## Tasks

Tasks can easily be added through our interface after specifying the setup and the reward
function.

We enumerate ten canonical tasks that we have created:

1. Reaching a position
2. Reaching a position at a target velocity
3. Touching the puck
4. Striking a stationary puck
5. Striking a puck into a crowd of blocks
6. Juggling
7. Hitting the puck to a minimum upward velocity
8. Moving a block by hitting it with the puck
9. Hitting the puck into a goal region
10. Hitting the puck into a goal region at a desired arrival velocity

All ten run in Box2D, six in Robosuite, and five on the real robot.

<figure>
  <img src="{{ '/assets/media/robosuite-tasks.jpg' | relative_url }}"
       alt="Three rows of six simulated frames each, labelled Touching, Puck Velocity, and Juggling, showing a robot arm at a table striking a red puck.">
  <figcaption>
    Rollouts in Robosuite for touching, hitting the puck to a minimum upward velocity, and
    juggling.
  </figcaption>
</figure>

<figure>
  <img src="{{ '/assets/media/box2d-crowd.png' | relative_url }}"
       alt="Four frames of a 2D air hockey rink: the paddle strikes a red puck upward into a cluster of orange blocks, which scatter.">
  <figcaption>
    Striking a puck into a crowd of blocks in Box2D. Reward comes from how far the blocks
    spread.
  </figcaption>
</figure>

Box2D additionally supports collaborative and adversarial two-player play.

## Supported workflows

### Teleoperation

We support:

1. **Mouse-teleop** — which streams the overhead view transformed into robot coordinates
   and maps the cursor to a desired end-effector pose.
2. **Shadow-teleop** — which tracks a real paddle a person moves on a flat surface and
   mirrors it onto the arm. This plays more naturally but is less responsive.

<figure class="medium">
  <img src="{{ '/assets/media/teleop.jpg' | relative_url }}"
       alt="Diagram: a human icon branches to Mouse-Teleop, showing a cursor driving the arm, and to Shadow-Teleop, showing a paddle tracked on a white surface driving the arm.">
  <figcaption>
    The two teleoperation modes.
  </figcaption>
</figure>

These produced the testbed's offline dataset: 350 mouse-teleop and 50 shadow-teleop
trajectories — roughly 128,000 frames — from eight participants of varying skill.

### Imitation and offline learning

Behavior cloning and offline RL (IQL) train on the offline dataset with no environment
interaction. They are then evaluated on the real robot.

<figure>
  <img src="{{ '/assets/media/real-rollouts.jpg' | relative_url }}"
       alt="Three rows of seven overhead frames of the real table, labelled BC, IQL, and Tele, with the puck circled in red as the arm moves to strike it.">
  <figcaption>
    Real-robot rollouts from behavior cloning, IQL, and a human teleoperator. The puck is
    circled for emphasis.
  </figcaption>
</figure>

### Reinforcement learning

We implement PPO for the standard tasks and SAC with hindsight relabeling for the
goal-conditioned tasks, in either simulator.

For how these methods perform across the three domains, see the
[testbed paper](https://arxiv.org/abs/2405.03113).

### Sim-to-real

After training in the calibrated simulator with domain randomization, a policy can be
deployed directly onto the arm with no real-world fine-tuning.

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
    Policies trained entirely in simulation, applied directly to the robot.
  </figcaption>
</figure>

An automatic reset policy recovers the puck between episodes and returns it to play, enabling
automatic evaluation and continued online learning.

More details are in **[An End-to-End Sim-to-Real Pipeline for Air
Hockey]({{ '/from-testbed-to-robot/' | relative_url }})**.

## Applications

The air hockey system has been used to test many algorithms. We list out some of these works.

### **[Learning Object Manipulation from Scratch via Contrastive Interaction](https://arxiv.org/abs/2606.11525)** (Shen et al., 2026)

IWR is a contrastive RL method that resamples experience around the moments where objects actually interact.
For their experiments, the authors trained goal-conditioned policies in our simulator and deployed them to the arm through the sim-to-real pipeline.

<figure>
  <video src="{{ '/assets/media/crl-goal-reaching.mp4' | relative_url }}"
         poster="{{ '/assets/media/crl-goal-reaching.jpg' | relative_url }}"
         controls muted loop playsinline preload="metadata"></video>
  <figcaption>
    A goal-reaching task on the real table, comparing several algorithms brought to the
    robot through the pipeline.
  </figcaption>
</figure>

### **[A Dual Approach to Imitation Learning from Observations with Offline Datasets](https://arxiv.org/abs/2406.08805)** (Sikchi et al., CoRL 2024)

DILO is an algorithm that can leverage arbitrary suboptimal data to learn imitating policies without requiring expert actions.
For their experiments, the authors ran three tasks on the setup, each with expert and suboptimal datasets collected through the teleoperation system.

<figure>
  <div class="figure-row">
    <img src="{{ '/assets/media/dilo-task-obstacles.jpg' | relative_url }}"
         alt="The arm holds a strawberry in its gripper above the table, with green, blue, and red cups placed around the workspace as obstacles.">
    <img src="{{ '/assets/media/dilo-task-striking.jpg' | relative_url }}"
         alt="The arm holds a paddle on the table beside a stationary red puck.">
    <img src="{{ '/assets/media/dilo-task-hitting.jpg' | relative_url }}"
         alt="Camera view down the length of the table: the arm's paddle waits near the centre line as a red puck travels toward it.">
  </div>
  <figcaption>
    The three tasks Sikchi et al. ran on the setup. <em>Left:</em> move an object to a goal
    without knocking over the obstacles. <em>Centre:</em> strike a stationary puck.
    <em>Right:</em> strike a moving puck.
  </figcaption>
</figure>

### **[Null Counterfactual Factor Interactions for Goal-Conditioned Reinforcement Learning](https://arxiv.org/abs/2505.03172)** (Chuck et al., ICLR 2025)

HInt is a hindsight relabeling method that keeps only the goals reached through an actual interaction between objects.
For their experiments, the authors used the Box2D environment on the task of hitting the puck into a goal region.

<figure>
  <img src="{{ '/assets/media/hint-rollout.png' | relative_url }}"
       alt="Six frames of a Box2D air hockey rollout: the blue paddle rises to meet the red puck, strikes it, and the puck travels up into the green goal region.">
  <figcaption>
    A goal-reaching rollout in the Box2D environment. The paddle (blue) strikes the
    falling puck (red) into the goal region (green).
  </figcaption>
</figure>

## Future work

While there is always room for better robustness, our system is largely in place. Our next
focus is on algorithms in the air hockey setting.

As an example, we are considering Meta-RL for sim-to-real adaptation across longer
contexts. Learned sim-to-real adaptation is most common in locomotion settings, where most
dynamics can be inferred from short interactions with the environment. In air hockey,
important system dynamics may need to be identified from a collision that happened farther
back in context.

We welcome collaborators interested in any aspect of this, whether that is evaluating a
new algorithm or working on the system itself. The [code]({{ site.repo_url }}) is public.
