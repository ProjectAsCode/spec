# Scenario 03: Alerts (Blocked)

**Spec reference:** §6

---

## Overview

PAC v0.0.1 defines a single alert: **Blocked**. It is a structural alert (no calendar, no "now" position). It fires for tasks and milestones that are gating downstream work.

A task is Blocked when:
- It has no `actual.*` fields (not started), AND
- At least one successor has every other predecessor dependency met (only this task is missing).

A milestone is Blocked when:
- It is not yet `reached` (per §3.4), AND
- At least one successor task has every other predecessor dependency met.

---

## case-a-task-blocked.pac.toml

**Why this case exists:** The simplest Blocked trigger for a task. A linear chain A → B → C where A is not started but B has all its other predecessor deps met (there is only one, A). A is the bottleneck for B.

**Expected alert for task `01ALRT00000000000000000001`:** Blocked

---

## case-b-milestone-blocked.pac.toml

**Why this case exists:** The Blocked trigger applied to a milestone. Predecessor task A is complete; milestone M depends on A and B; B is not done. Successor task X depends on M. M is blocking X because B is incomplete.

In v0.0.1, M's Blocked state is triggered when M is not `reached` and a successor task is otherwise ready. Here X has only M as its predecessor; once M is reached, X can start.

**Expected alert for milestone `01ALRT00000000000000000020`:** Blocked

---

## case-c-not-blocked-multiple-predecessors.pac.toml

**Why this case exists:** A task that is not-started but whose successors have OTHER unfinished predecessors must NOT show Blocked. The alert only fires when the not-started task is the SOLE remaining blocker.

Project structure: A and B both feed into C. A is not started. B is also not started. Neither A nor B is "the only thing blocking C" because both are missing.

**Expected alert for tasks `01ALRT00000000000000000031` and `01ALRT00000000000000000032`:** None (not Blocked)

**What breaks if wrong:** An implementation that fires Blocked whenever a not-started task has any successor would flood the diagram with false-positive Blocked alerts on every unstarted task. The "only remaining blocker" check is essential.

---

## Implementation notes

- Blocked is computed from the task graph alone. No reference to "now" or any calendar position.
- A task that is in-progress (`actual.*` fields present) does NOT fire Blocked. Work that has started is no longer "blocking" in the harmful sense.
- A milestone with `actual.*` fields cannot exist (milestones have no `actual.*` group); the `reached` state is fully derived from predecessor completion.
- Multiple Blocked alerts can fire simultaneously across the project. Implementations MUST surface all of them.
