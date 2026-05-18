# Scenario 05: Unconvert Sub-project

**Spec reference:** §7.3

---

## Overview

Unconverting collapses a sub-project back into the single linked task in the parent project. After the operation:

1. The linked task's `plan.duration` is set to the sub-project's computed total duration (`max(EF)` after the sub-project's forward pass).
2. The linked task's `description`, `notes`, and `emoji` are merged from the sub-project (per §7.3 rules).
3. The linked task's `sub_project_id` is cleared.
4. The sub-project record (and all its tasks, milestones, sub-projects recursively) is removed from the file.

The linked task's `dependencies` in the parent project are unchanged.

---

## unconvert.pac.toml

**Before unconvert:**

| Entity | Field | Value |
|---|---|---|
| Parent task `...002` | sub_project_id | `01CNVT00000000000000000010` |
| Parent task `...002` | description | `"Original task description"` |
| Parent task `...002` | notes | `"Original task notes"` |
| Sub-project `...010` | description | `"Sub-project description"` |
| Sub-project `...010` | notes | `"Sub-project notes"` |
| Sub-task `...011` | plan.duration | 5 |
| Sub-task `...012` | plan.duration | 7 (depends on `...011`) |

**Sub-project forward pass:** total duration = max(EF) = 12.

**After unconvert (expected state of parent task `...002`):**

| Field | Value |
|---|---|
| sub_project_id | (absent: cleared) |
| plan.duration | 12 |
| description | "Original task description\n\nSub-project description" |
| notes | "Original task notes\n\nSub-project notes" |

The sub-project record is deleted from the file entirely.

---

## Implementation notes

- Unconvert is destructive and MUST require explicit user confirmation (§7.3).
- If the sub-project itself contains nested sub-projects, the unconvert is recursive: nested sub-projects must be flattened first (or removed alongside the parent sub-project, depending on the implementation).
- The linked task's `actual.progress` reverts to user-controlled after unconvert (it was read-only while linked).
