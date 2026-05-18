# Scenario 04: Sub-project Duration and Progress

**Spec reference:** §5.3, §4

---

## Overview

A sub-project is a nested project linked to a task in the parent project. The parent task has no effective `plan.duration` of its own when `sub_project_id` is set: the duration is computed from the sub-project's forward pass at load time.

This scenario tests two computations:

1. **Duration derivation:** the linked task's effective duration in the parent, computed as `max(EF)` across the sub-project after running its forward pass and parallelism constraint.
2. **Progress aggregation:** the linked task's `actual.progress` value, computed as a weighted average of the sub-tasks' progress values, weighted by their `plan.duration`.

In v0.0.1, parent and sub-project MUST share the same `time_unit`. No unit conversion is performed.

---

## duration.pac.toml

**Sub-project structure:**

| Sub-task id | plan.duration | dependencies |
|---|---|---|
| 01SPRJ00000000000000000011 | 5 | none |
| 01SPRJ00000000000000000012 | 7 | STASK1 |

**Sub-project forward pass:** STASK1 ends at EF=5, STASK2 ends at EF=12. The sub-project's `max(EF)` is 12.

**Expected:** The linked task `01SPRJ00000000000000000002` presents an effective duration of `12` to the parent's forward pass.

**What breaks if wrong:** If the implementation uses the linked task's stored `plan.duration` (if present), the parent schedule drifts out of sync whenever a sub-task estimate changes. The correct behavior is to recompute from the sub-project on every load.

---

## progress.pac.toml

**Sub-project structure:**

| Sub-task id | plan.duration | actual.progress |
|---|---|---|
| 01SPRJ00000000000000000031 | 5 | 100 (done) |
| 01SPRJ00000000000000000032 | 10 | 50 |
| 01SPRJ00000000000000000033 | 3 | 0 (not started) |

**Weighted average:**

```
numerator   = (5 × 100) + (10 × 50) + (3 × 0) = 500 + 500 + 0 = 1000
denominator = 5 + 10 + 3 = 18
raw         = 1000 / 18 = 55.555...
progress    = round(55.555) = 56
```

**Expected linked task progress:** `actual.progress = 56`.

The stored value in the file (`56`) is a cached display value. Implementations MUST recompute from the sub-tasks on load. The stored value is not authoritative.

**What breaks if wrong:**

1. **Simple average instead of weighted:** `(100 + 50 + 0) / 3 = 50`. Underweights the larger task (STASK2).
2. **Using stored progress as authoritative:** Editing a sub-task progress would not propagate; the parent would show stale data.

---

## Implementation notes

- Milestones in a sub-project are excluded from the progress weighted average (they have no duration weight).
- Sub-project-linked sub-tasks (nested sub-projects) use their own computed total duration as their weight.
- The same single `time_unit` applies across the parent and sub-project tree (v0.0.1 architectural rule, §9).
