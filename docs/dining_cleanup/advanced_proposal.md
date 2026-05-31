# AI-Capstone Advanced Proposal (Group 13)

## Overview

This advanced-level task studies post-meal dining cleanup with a single Franka robot arm. The robot is required to clear tableware from a dining table, place the tableware into a tray, retrieve a wiping cloth, and wipe a specified dirty region of the table. At the same time, it must preserve non-target objects such as a tissue box and a vase, which may be present on the table but should not be moved or collected.

Compared with the entry-level cutlery arrangement task, this task introduces a more complex manipulation setting. The robot must reason about object roles, perform a sequence of dependent subtasks, use a tool for surface cleaning, and satisfy safety constraints. The task is therefore not simply an object transport problem; it is a constrained, multi-stage manipulation problem that combines tableware clearing, tool use, surface coverage, and disturbance avoidance.

## Motivation

Dining cleanup is a common and repetitive household activity. A robot capable of clearing tableware and wiping a table surface could reduce the physical burden of daily chores and support practical applications in home assistance, elder care, mobility assistance, and service robotics.

The entry-level project focuses on pre-meal arrangement, where the robot places utensils around a plate. The advanced task extends the setting to post-meal cleanup. This changes the objective from placing objects in predefined positions to completing a structured cleanup procedure under constraints. The robot must distinguish target objects from protected objects, execute multiple manipulation stages in the correct order, and clean a surface area rather than only reaching a point target.

## Problem Formulation

We formulate dining cleanup as a finite-horizon constrained manipulation problem. At each timestep \(t\), the environment has a state \(s_t\), which includes the robot configuration, gripper state, object poses, table state, and task progress. The robot receives an observation \(o_t\), which may include proprioceptive information and visual observations, and outputs an action \(a_t\) to control the arm and gripper.

The goal is to learn or design a policy:

```text
pi(a_t | o_t)
```

that maximizes the probability of completing the cleanup task within a fixed episode horizon while satisfying placement, wiping, and safety constraints.

### Task Objective

The robot must complete the following high-level objectives:

1. Identify the tableware objects that should be cleared.
2. Move each target tableware object into the tray.
3. Retrieve the cloth after the tableware has been cleared.
4. Wipe the designated dirty region of the table.
5. Avoid disturbing protected objects and avoid dropping task objects.

The task is successful only when all required subgoals are satisfied:

```text
Success = tableware_clearing
          AND wiping_coverage
          AND protected_object_stability
          AND object_safety
```

### Object Roles

The scene contains objects with different semantic roles:

| Object Type | Role in the Task |
|-------------|------------------|
| Bowl and spoon | Target tableware to be cleared |
| Tray | Receptacle for cleared tableware |
| Cloth | Tool used for wiping the table |
| Tissue box and vase | Protected non-target objects |

This role distinction is central to the task. The robot should manipulate target tableware and the cleaning tool, but it should not treat protected objects as cleanup targets.

### Task Constraints

The task includes the following constraints:

1. Target tableware must be placed inside the tray region.
2. The dirty table region must be wiped with sufficient coverage.
3. Protected objects must remain near their initial poses.
4. The robot should avoid severe collisions and should not push objects off the table.
5. The wiping stage should occur after the tableware clearing stage, because uncleared tableware may block the wiping region.

These constraints make the task more challenging than independent pick-and-place. A successful policy must coordinate sequencing, grasping, placement, and wiping behavior under a shared workspace.

## Environment and Data Generation

The environment consists of a dining table, a Franka robot arm, target tableware, a tray, a cloth, and protected objects. The target tableware is initialized at varying tabletop positions to test robustness across different cleanup layouts. The tray, cloth, and protected objects define the structure of the task scene and provide stable reference roles for placement, tool use, and disturbance evaluation.

Training and evaluation data can be generated from two complementary sources:

1. Expert demonstrations that execute the intended cleanup sequence.
2. Human teleoperation demonstrations that capture manual corrections and realistic interaction patterns.

The generated episodes should record the robot observations, actions, object states, and final task outcome. These data can then be used for imitation learning, policy evaluation, and failure analysis.

## Proposed Approach

The proposed solution follows a staged manipulation strategy. The policy should first complete tableware clearing and then perform table wiping. This decomposition reflects the physical structure of the task: the robot should remove objects from the dirty region before attempting to wipe it.

### Stage 1: Tableware Clearing

In the first stage, the robot identifies the target tableware objects and transports them into the tray. The policy must decide which object to grasp, approach it with a stable grasp configuration, lift it safely, and release it inside the tray. This stage tests object recognition, object-specific grasping, transport stability, and constrained placement.

### Stage 2: Table Wiping

After the target tableware has been cleared, the robot retrieves the cloth and wipes the dirty region. The wiping behavior should cover a sufficient portion of the target surface while maintaining table contact and avoiding protected objects. This stage evaluates tool use, surface coverage, and motion planning in a cluttered tabletop workspace.

## Expected Outcome and Evaluation

The task will be evaluated using four primary criteria: tableware placement, wiping coverage, protected-object stability, and object safety.

### 1. Tableware Placement Success

Let \(N\) be the number of target tableware objects. For each target object \(i\), define an indicator \(P_i\), where \(P_i = 1\) if the object is successfully placed inside the tray region and \(P_i = 0\) otherwise.

The tableware clearing score is:

```text
Score_clear = (sum_i P_i) / N
```

The placement subtask is successful when all target tableware objects are inside the tray region.

### 2. Wiping Coverage

Let \(R_w\) denote the dirty table region and let \(C\) denote the subset of that region covered by the cloth during wiping. The wiping coverage is:

```text
Coverage = area(C intersection R_w) / area(R_w)
```

The wiping subtask is successful when the coverage is greater than or equal to a predefined threshold.

### 3. Protected-Object Stability

For each protected object \(j\), let \(p_j^0\) be its initial position and \(p_j^T\) be its final position. The displacement is:

```text
D_j = ||p_j^T - p_j^0||
```

The protected-object constraint is satisfied when every protected object's displacement remains below a predefined tolerance.

### 4. Object Safety

The task should fail if a target object, the cloth, or a protected object falls off the table or is displaced in a way that indicates severe unintended contact. This criterion is included to discourage policies that achieve placement or coverage by using unsafe motions.

### Overall Success

An episode is considered successful when:

1. All target tableware objects are placed in the tray.
2. The dirty region is wiped above the required coverage threshold.
3. All protected objects remain within the allowed displacement tolerance.
4. No task object falls off the table or undergoes severe unintended disturbance.

## Anticipated Challenges

### 1. Multi-Stage Task Sequencing

The robot must complete the subtasks in a meaningful order. If the wiping stage begins before the tableware is cleared, the cloth may collide with the tableware or push it into protected objects. The policy must therefore maintain task progress and transition between stages only when the previous stage is complete.

### 2. Object-Specific Grasping

The bowl and spoon have different shapes and require different grasping behavior. A bowl is curved and may require stable edge contact, while a spoon is thin and elongated, making it sensitive to gripper alignment. This increases the difficulty of learning a robust manipulation policy.

### 3. Role Distinction

The robot must distinguish between target objects, tools, receptacles, and protected objects. This is important because the correct behavior depends not only on object geometry but also on task role. Moving a protected object may be physically easy, but it violates the task objective.

### 4. Wiping as Area Coverage

Wiping is evaluated over a surface region rather than a single target pose. The robot must keep the cloth in contact with the table and generate motions that cover enough of the dirty region. This requires a different success representation from standard pick-and-place tasks.

### 5. Disturbance Avoidance

The table contains protected objects near the robot's workspace. The robot must avoid collisions while carrying tableware and while wiping with the cloth. This is challenging because the robot, grasped objects, and cloth all occupy space and may interact with nearby objects.

### 6. Demonstration Quality

Learning from demonstrations requires trajectories that are temporally smooth, physically plausible, and correctly labeled. Because the task contains multiple stages, an error in an early stage can affect later behavior and lead to ambiguous failure cases.

## Current Scope and Future Extensions

The current scope focuses on a single-arm dining cleanup scenario with two target tableware objects, one tray, one cloth, and protected tabletop objects. This scope is sufficient to study sequencing, role-aware manipulation, constrained placement, and wiping coverage.

Future extensions may increase scene diversity by adding more tableware, randomizing receptacle and protected-object locations, introducing additional obstacle types, or using more realistic deformable cloth dynamics. Another extension is to develop adaptive wiping policies that plan coverage trajectories based on the current table layout rather than following a fixed cleaning pattern.
