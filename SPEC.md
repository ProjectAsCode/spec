# PAC Specification: Version 0.0.1

**Status:** Draft
**Published by:** Project As Code (projectascode.org)
**License:** Creative Commons Attribution 4.0 International (CC-BY 4.0)
**Repository:** github.com/projectascode/spec

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Terminology](#2-terminology)
3. [Data Model](#3-data-model)
4. [Progress and Status Model](#4-progress-and-status-model)
5. [Scheduling Algorithm](#5-scheduling-algorithm)
6. [Alert Definitions](#6-alert-definitions)
7. [Sub-project Rules](#7-sub-project-rules)
8. [File Format](#8-file-format)
9. [Scale Constraints](#9-scale-constraints)
10. [Field Reference](#10-field-reference)

---

## 1. Introduction

Project As Code (PAC) is an open standard for storing project plans as version-controlled text files. A PAC file (always named `pac.toml`) lives in the same repository as the code it describes. It is readable in any text editor, diffable with standard git tooling, and interpretable by any conformant implementation without conversion or export.

### The problem

Project plans typically live in tools disconnected from the code they describe: issue trackers, spreadsheets, presentation slides, or project management SaaS products. When the code changes, the plan does not update automatically. When the plan changes, there is no audit trail of who changed what or why. The plan and the codebase diverge over time, and the plan loses credibility.

### The insight

Infrastructure as Code demonstrated that treating operational configuration as version-controlled text (rather than as manual, undocumented state) produces more reliable, auditable, and collaborative systems. The same principle applies to project planning. A plan stored as text in a repository inherits all of git's capabilities for free: authorship, history, branching, diffing, and code review.

### The relationship to git

A PAC file is a first-class git citizen. The file format is deliberately designed so that small planning changes produce small, readable diffs. Adding a task is a few lines of [TOML](https://toml.io). Changing a time estimate is one line. Reviewers can approve or request changes to a schedule the same way they review code. Branches can model alternative timelines. Tags can mark approved baselines. The audit trail of what was planned, when, and by whom is the git log.

### Scope

PAC v0.0.1 describes the *plan*: what work is intended, in what order, and whether it is done. The scope is deliberately small to invite contribution and feedback. The format is designed to extend without breaking: future versions add new fields and capabilities, but v0.0.1 files remain loadable by all later 0.x and 1.x implementations.

### What v0.0.1 covers

- Projects, tracks, tasks, milestones, and sub-projects
- Finish-to-start dependencies
- A single duration estimate per task
- Track parallelism with priority tiebreaking
- Forward-pass scheduling
- Progress tracking and a done signal
- A single structural alert: "Blocked"

### What v0.0.1 deliberately defers

The following are out of scope for v0.0.1 and noted here so the absence is not mistaken for an oversight. Each can be added in a future minor version without breaking v0.0.1 files (see §8.9 Versioning):

- **Three-point PERT estimates.** v0.0.1 uses a single `plan.duration`. A future version can introduce optional `plan.optimistic` / `plan.pessimistic` fields alongside `plan.duration` (which remains the "most likely" value).
- **Float and critical-path computation.** v0.0.1 runs only the forward pass. A future version can add the backward pass and float derivation; this is a pure computation, no file-format change.
- **Multiple path types** (Optimistic / Pessimistic views). Depends on PERT being added.
- **PERT probability table.** Confidence percentages depend on PERT variance.
- **Live "overrun" detection.** v0.0.1 has no "now" position and cannot detect slow-running tasks mid-flight. Late starts are still recordable via `actual.offset`; late completion via `actual.duration` exceeding `plan.duration` after the fact. Users who need finer signals should split long tasks into smaller ones.
- **Mixed `time_unit` in a project tree.** v0.0.1 requires every project and sub-project in the file to share the same `time_unit` (days OR hours, project-wide). A future version can add unit conversion across the tree.
- **Resource / cost modeling.** Tracks in v0.0.1 are slot counters; no per-resource capacity or cost fields.
- **Cross-file dependencies.** Dependencies must reference tasks or milestones within the same `pac.toml`.
- **Coarse-grained sub-project dependencies.** A parent task linked to a sub-project (via `sub_project_id`) waits for the *entire* sub-project to complete (its `max(EF)`). When a parent task only needs a portion of a sub-project's output (e.g. a Mobile task only needs the Backend API contracts, not the full Backend), the v0.0.1 workaround is to split the sub-project into smaller sub-projects so each parent dependency lands on the right granularity. A future version may introduce cross-level dependencies.

The trade-off chosen for v0.0.1: ship something small that real teams can use today, gather feedback, and let usage shape what gets added.

---

## 2. Terminology

**Project**: The top-level container for a plan. A project contains tracks, tasks, and milestones. It is stored in a `pac.toml` file, together with all its sub-projects.

**Track**: A resource lane (e.g. a team, a person, a department). Tasks are assigned to tracks. Each track has a parallelism constraint that limits how many tasks may run simultaneously within it.

**Task**: A unit of work. A task has a `plan.duration` estimate, optional `actual.*` execution fields (`actual.progress`, `actual.offset`, `actual.duration`), and an optional set of dependencies on other tasks or milestones.

**Milestone**: A zero-duration synchronisation point. A milestone has no duration and no progress value: it is either reached (all predecessor tasks and milestones are complete) or not. Milestones reduce dependency complexity by acting as convergence and divergence points.

**Sub-project**: A full project embedded in the same `pac.toml` file as its parent, and linked to a specific task in the parent project. The sub-project's computed duration and completion bubble up to the linked task in the parent.

**Dependency**: A finish-to-start relationship between two tasks, between two milestones, or between a task and a milestone. A dependent entity cannot start until all its predecessors are complete. In this version of the specification, only finish-to-start dependencies are defined.

**ID string**: A unique string identifier for each entity. Implementations MAY use any format that guarantees uniqueness and is a valid TOML string. [ULID](https://github.com/ulid/spec) (Universally Unique Lexicographically Sortable Identifier) is the recommended format: a 26-character [Crockford Base32](https://www.crockford.com/base32.html) string that sorts lexicographically in creation order. The PAC reference implementation uses ULIDs. Implementations that choose a different format MUST ensure IDs remain stable (never change after creation) and unique within a file.

---

## 3. Data Model

All identifier fields store unique strings (see Section 2). The `description` field on Project, Track, Task, and Milestone, and the `notes` field on Task, accept [CommonMark](https://spec.commonmark.org) markdown. A conformant implementation MAY render CommonMark constructs (headings, lists, emphasis, links, code spans, etc.) as formatted text. Inline or block HTML within these fields MUST be displayed as literal escaped text and MUST NOT be rendered as HTML or executed. Implementations MUST NOT auto-load images or external resources referenced in these fields without explicit user action.

### 3.1 Project

A project is the top-level container. It is stored in a single `pac.toml` file along with all its sub-projects.

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | String | Yes | Unique identifier |
| `name` | String | Yes | Display name |
| `color` | Hex string | No | Accent color. 6-digit uppercase hex prefixed with `#` (e.g. `"#3B82F6"`). Lowercase and 3-digit shorthand are not valid. Default: `"#3B82F6"`. |
| `time_unit` | Enum | Yes | `"hours"` or `"days"`. Default: `"days"`. Every project and sub-project within the same `pac.toml` MUST share the same `time_unit` in v0.0.1. |
| `description` | String | No | Long description; CommonMark supported. |
| `emoji` | String | No | Optional label; a single grapheme cluster stored as-is. |
| `parent_project_id` | String | No | If this is a sub-project: the ID of the parent project. Null for top-level projects. |
| `linked_task_id` | String | No | If this is a sub-project: the ID of the task in the parent project that links to this sub-project. Null for top-level projects. |

### 3.2 Track

A track represents a resource lane. Tasks are assigned to tracks. The default `parallelism = 1` means tracks are serial unless configured otherwise; set `parallelism = 0` for unlimited parallelism, or to any positive integer N for at-most-N concurrent tasks.

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | String | Yes | Unique identifier |
| `project_id` | String | Yes | Parent project |
| `name` | String | Yes | Display name |
| `color` | Hex string | No | Track color. 6-digit uppercase hex prefixed with `#`. Default: `"#10B981"`. |
| `parallelism` | Integer ≥ 0 | Yes | Max tasks running simultaneously. `0` means unlimited. |
| `order` | Integer | Yes | Display order (user-controlled). When two tracks share the same `order` value, they are sorted by `id` ascending as a tiebreak. |
| `description` | String | No | Optional description; CommonMark supported. |
| `emoji` | String | No | Optional label; a single grapheme cluster stored as-is. |

### 3.3 Task

Tasks do not have their own color field. Visual color identity comes from the track they belong to.

Fields are divided into three explicit groups:

**Structural fields** (root level of the task entry): describe what the task is and where it sits in the project. May be updated at any time.

**`plan.*` fields**: the time estimate. In v0.0.1, this is a single `plan.duration`. MAY be edited at any time; git history is the audit trail for revisions.

**`actual.*` fields**: execution reality. Populated as work progresses. MUST be absent entirely when no work has started on the task.

#### Structural fields

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | String | Yes | Unique identifier |
| `name` | String | Yes | Display name |
| `dependencies` | String[] | Yes | Task or milestone IDs this task depends on. Must be within the same project. Always written, even when empty. |
| `track_id` | String | No | Assigned track. Omit if unassigned (virtual track). May be reassigned at any time. |
| `priority` | Integer -100–100 | No | Priority for slot conflict resolution under parallelism constraints. Default: 0. Higher values run earlier within a track. Omit when 0. |
| `description` | String | No | What needs to be done. CommonMark supported. Omit if empty. |
| `notes` | String | No | Additional context. CommonMark supported. Omit if empty. |
| `emoji` | String | No | A single grapheme cluster. Omit if absent. |
| `sub_project_id` | String | No | If this task links to a sub-project. When set, `plan.duration` MAY be omitted and is ignored during scheduling; the sub-project's computed duration is used instead (Section 5.3). |

#### `plan.*` fields

| Field | Type | Required | Description |
|---|---|---|---|
| `plan.duration` | Number > 0 | Required unless `sub_project_id` is set | The estimated duration of the task, in the project's `time_unit`. At most 2 decimal places. |

**Forward-compatibility note:** Future versions may introduce additional optional fields (e.g. `plan.optimistic`, `plan.pessimistic`) for richer estimation models. `plan.duration` will remain the canonical "most likely" estimate; v0.0.1 files (with only `plan.duration`) will continue to load unchanged.

#### `actual.*` fields

| Field | Type | Required | Description |
|---|---|---|---|
| `actual.progress` | Integer 0–100 | No | Completion percentage. Always an integer. Omit entirely when 0 (task not started). `actual.progress = 100` is the done signal. |
| `actual.offset` | Number | No | Signed shift applied to the task's scheduled start, in the project's time unit. Positive = started later than planned. Negative = started earlier than planned (may break predecessor dependency; see §5.1). At most 2 decimal places. Omit when 0. |
| `actual.duration` | Number > 0 | No | Optional override of `plan.duration` in scheduling. MAY be set at any progress level: as a mid-execution revised estimate, or as a post-completion record of actual time taken. At most 2 decimal places. |

**Drift calculation:** `plan.duration` vs `actual.duration` gives the variance between the most recent plan and the recorded reality. Earlier baselines are recoverable through git history.

**`actual.*` absence rule:** When a task has not started, all `actual.*` fields MUST be omitted entirely from the file. Do not write `actual.progress = 0`. The absence of `actual.*` keys is the signal that the task is untouched, preserving the git-diff-friendly design: adding `actual.*` fields to a task is always a meaningful diff.

**Dependency scoping:** A conformant implementation MUST enforce that dependencies reference only tasks and milestones within the same project. Cross-project and cross-level dependencies are not permitted.

**Dependency existence:** A conformant implementation MUST reject a file (on load and on save) where any `dependencies` array entry refers to an ID that does not exist in the same project. This mirrors the orphan-reference rule in §7.5 and prevents silent schedule corruption from stale IDs left behind after a task or milestone is deleted.

### 3.4 Milestone

A milestone is a zero-duration synchronisation point. It has no duration and no progress: it is either reached or not.

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | String | Yes | Unique identifier |
| `project_id` | String | Yes | Parent project |
| `name` | String | Yes | Display name |
| `color` | Hex string | No | Milestone color. 6-digit uppercase hex prefixed with `#`. Default: `"#F59E0B"`. |
| `dependencies` | String[] | Yes | Task or milestone IDs that must all be complete before this milestone is reached. MUST contain at least one entry; an empty `dependencies` array makes a milestone meaningless. |
| `description` | String | No | Optional description; CommonMark supported. |
| `emoji` | String | No | Optional label; a single grapheme cluster stored as-is. |

**Derived state (computed, never stored):**
- `reached`: true when every predecessor task has `actual.progress = 100` and every predecessor milestone is also `reached`.
- ES, EF (equal to ES because duration = 0): computed by the scheduling algorithm exactly like tasks.

Milestones have no `actual.*` fields. Their reached state is a pure function of predecessor state, recomputed on every load. A conformant implementation MUST surface a validation error when a milestone has `dependencies = []`.

---

## 4. Progress and Status Model

The `actual.progress` field (integer 0–100) records the completion percentage of a task. `actual.progress = 100` is the done signal. Task status is derived from the presence and values of `actual.*` fields.

### Derived Status

| Condition | Derived status |
|---|---|
| `actual.progress = 100` | Done |
| Any `actual.*` field present and `actual.progress < 100` (or `actual.progress` absent but other `actual.*` fields present) | In Progress |
| No `actual.*` fields | Not Started |

`actual.duration` is an optional override of `plan.duration`. It MAY be set at any progress level: as a mid-execution revised estimate, or as a post-completion record. When set, scheduling uses `actual.duration` in place of `plan.duration`.

### Cancellation

PAC v0.0.1 has no dedicated cancellation state, field, or convention. A task or milestone that the team has decided not to do is **deleted from the file** along with any references to it. The git history is the record of what was once planned and why it was removed.

The deletion operation:

1. Remove the task or milestone record from the file.
2. Remove its `id` from every other task's or milestone's `dependencies` array (the existing dependency-existence rule in §3.3 already requires this; otherwise the file is invalid).
3. Commit the change with a message that captures the reason (the same way a code change captures why a function was removed).

This is symmetric for tasks and milestones, and consistent with how source code is treated: deleted lines do not stay in the file marked "// REMOVED"; they are gone, and git remembers.

**What this trade implies:**

- Retrospective queries such as "what did we cancel?" or "how much time did we spend on cancelled work?" are not answerable from the current file alone. They are answered by walking the file's git history. Tools that surface these views SHOULD read commit history (e.g. `git log -p -- pac.toml`) rather than rely on in-file markers.
- A cancelled task's `actual.duration` (the real time spent on it) leaves the schedule along with the task. The forward pass no longer accounts for that consumed time. For most projects the inaccuracy is minor; for projects with significant late-stage cancellations, teams may compensate via `plan.duration` adjustments on related work or surface the cumulative cancelled-work duration in their tooling layer.
- Restoring a cancelled task is a `git revert` (or manual re-add) of the cancellation commit. Tooling MAY offer a one-click "restore from history" view that reads git log and lets the user resurrect deleted entries.

### Progress and Sub-projects

When a task has `sub_project_id` set, its `actual.progress` value is read-only and computed from the linked sub-project's overall completion. A conformant implementation MUST NOT allow the user to set `actual.progress` directly on a sub-project-linked task.

**Sub-project progress formula:** Progress is the weighted average of all tasks' effective progress values in the sub-project, weighted by each task's `plan.duration` (or, for a sub-project-linked sub-task, the linked sub-sub-project's computed duration). The effective progress for a task is its `actual.progress` value (treating absent as 0). Milestones are excluded from this calculation (they have no duration weight). The result is rounded to the nearest integer.

```
actual.progress = round( Σ (task.actual.progress × weight(task)) / Σ weight(task) )
```

Where `weight(task)` is:
- For a regular task: its `plan.duration`
- For a sub-project-linked task: its sub-project's total duration (max EF across the sub-project's forward pass)

Applied recursively: if the sub-project contains its own sub-project-linked tasks, their weights are their sub-sub-project durations (already computed by the time this formula runs).

For all other tasks, `actual.progress` is user-controlled (integer 0–100).

### Progress Rules

- When the user first records `actual.progress > 0`: add the `actual.*` group to the task entry in the file.
- When the user marks the task complete: set `actual.progress = 100`. Optionally also set `actual.duration` if the actual time taken differs from `plan.duration`.
- When the user un-marks the task as complete: reduce `actual.progress` below 100. `actual.duration` MAY remain set (its role becomes "current best estimate" again).
- When `actual.progress` is cleared to 0 and `actual.duration` is absent and `actual.offset` is 0: remove all `actual.*` fields from the task entry.

**Actual fields for sub-project-linked tasks:** When a task has `sub_project_id` set, its `actual.*` fields are computed, not user-set:

- The linked task's `actual.*` fields are absent if no sub-task has any `actual.*` fields set.
- `actual.offset` reflects the earliest effective start across all sub-tasks, relative to the linked task's scheduled ES in the parent project.
- `actual.duration` is set only when every sub-task has `actual.progress = 100`, and equals `max(effective_EF) - min(effective_ES)` across the sub-project.
- `actual.progress` is the weighted average per the formula above.

These values MUST be recomputed whenever any sub-task's `actual.*` fields change. A conformant implementation MUST NOT allow the user to set `actual.*` fields directly on a sub-project-linked task.

---

## 5. Scheduling Algorithm

PAC v0.0.1 scheduling operates entirely in relative time units from ES = 0. There are no calendar dates and no "now" position. The result is a deterministic, timezone-independent schedule that is identical for all readers of the same file.

The algorithm has two passes: a forward pass to compute when each task starts and ends, and a track-parallelism constraint that may shift tasks forward if their assigned track is at capacity. Backward-pass concepts (LF, LS, float, critical path) are deferred to a future version.

Reality is described through two execution fields on tasks:
- `actual.offset` — shifts a task's scheduled start (positive = later, negative = earlier).
- `actual.duration` — when set, replaces `plan.duration` in scheduling.

Milestones participate in the algorithm alongside tasks. Wherever "task" is mentioned in this section, the rule applies equally to milestones unless stated otherwise. Milestones have duration = 0, so EF always equals ES. Track parallelism constraints do not apply to milestones.

### 5.1 Forward Pass

There is a single schedule. The plan provides the forecast for un-executed tasks; recorded `actual.*` fields take precedence where present. Both feed into the same ES and EF values.

Tasks are processed in dependency order: a task is eligible for scheduling once all its predecessors have been assigned an EF value. When multiple tasks become eligible at the same point (no ordering constraint between them), they MUST be processed in `id` ascending order. This tie-break rule is what makes the forward pass deterministic for arbitrary DAGs.

```
duration(task) = actual.duration if set, otherwise plan.duration
              (for sub-project-linked tasks, see §5.3)

ES(task) = max(EF(predecessor)) for all predecessors
         = 0 if no predecessors

ES(task) = max(0, ES(task) + actual.offset)   -- apply offset, clamp at project start

EF(task) = ES(task) + duration(task)
```

**Propagation rule:** A successor's ES uses its predecessors' EF as computed above. There is one EF per task. Downstream computation (further forward pass, parallelism, alerts) uses these unified values.

**Offset semantics:**
- Positive `actual.offset`: the task started later than its predecessors finished. ES is shifted forward; dependencies remain satisfied.
- Negative `actual.offset`: the task started earlier than its dependency-derived ES (parallel work, soft dependency, or recorded reality where the task and its predecessor overlapped). The dependency clamp is NOT applied; effective ES MAY be earlier than `max(EF(predecessor))`. The schedule records what actually happened. Implementations MAY visually flag this as a dependency override, but MUST NOT block the file from loading.
- Zero or absent `actual.offset`: ES equals the dependency-derived value.

In all cases, ES is clamped at 0 (the project start). No task can have a negative effective start.

A conformant implementation MUST detect dependency cycles before running the forward pass. If a cycle is detected, scheduling MUST be aborted and the error surfaced to the user.

### 5.2 Track Parallelism Constraint

Applied immediately after the forward pass.

A **slot** is a position in a track's parallelism window. At any point in time, at most N slots may be occupied simultaneously.

Within a track with `parallelism = N` (where N > 0):
- At most N tasks may overlap in time.
- When multiple tasks have the same ES and all N slots are already occupied at that moment, they are sorted by `priority` descending (higher value runs first); tasks with equal priority are sorted by `id` ascending.
- Tasks that cannot be placed because all slots are occupied at their ES have their ES shifted to the earliest moment a slot opens; their EF is updated accordingly.
- This shift may cascade: if a task's EF changes, its successors' ES values must be recalculated.

A conformant implementation MUST iterate the forward pass and parallelism constraint together until no task's ES or EF changes between iterations. Each iteration MUST redo the full forward pass from scratch across all tasks. The maximum iteration count is **200**. If stability is not reached after 200 iterations, the implementation MUST surface a scheduling error to the user and MUST NOT silently use a partially-stable result.

Tracks with `parallelism = 0` have unlimited parallelism; no constraint is applied.

Tasks with `track_id = null` (unassigned) are not subject to any parallelism constraint and are scheduled as if their track has unlimited parallelism.

"The earliest moment a slot opens" means the earliest EF value among all tasks currently occupying a slot in that track.

Parallelism constraints may increase the total project duration beyond what pure dependency-based scheduling would produce.

### 5.3 Sub-project Duration in Parent

When a task has a `sub_project_id`:
1. Run the sub-project's own forward pass (and parallelism constraint, §5.2) to completion.
2. The sub-project's total duration is `max(EF)` across all tasks and milestones in the sub-project.
3. This value is used as the linked task's duration in the parent's forward pass.

Because v0.0.1 requires every project in the file to share the same `time_unit` (§3.1), no unit conversion is needed. The sub-project's `max(EF)` is used directly.

The linked task's own `plan.duration` field, if present, is preserved in the file but MUST be ignored during scheduling. It is permitted to omit `plan.duration` when `sub_project_id` is set.

---

## 6. Alert Definitions

Alerts in v0.0.1 are structural: based on task state and project graph relationships, not on calendar time. There is no "now" indicator. The project does not know what day it is.

v0.0.1 defines a single alert: **Blocked**, on tasks.

### Blocked

| Entity | Trigger | Meaning |
|---|---|---|
| Task | The task has no `actual.*` fields (not started), AND every one of its own predecessors is complete (all predecessor tasks have `actual.progress = 100` and all predecessor milestones are `reached`), AND at least one successor has every other predecessor dependency met (only this task is missing). | This task could be started right now and is the only thing preventing downstream work from starting. |

The "own predecessors complete" clause is what makes the alert actionable: a Blocked task is one where someone could pick up work today and unblock something downstream. Tasks whose own predecessors are not yet complete are gated by upstream work and the Blocked signal would belong upstream.

A conformant implementation MUST surface every active Blocked state.

### Milestones do not fire Blocked

In v0.0.1, milestones are auto-reached: their `reached` state is derived purely from predecessor completion (§3.4). A milestone that is not reached is, by definition, gated by some predecessor task that is itself either incomplete (and may already fire Blocked) or in progress. No additional alert on the milestone would add actionable information; users tracing why a milestone has not been reached look at the upstream tasks.

### Future alerts

Additional alerts (e.g. "On critical path and unstarted", "Overrun risk", "Ahead of schedule") are deferred to future versions. They will be defined as derived states over the existing data model and will not require file-format changes. v0.0.1 files will compute new alerts correctly under v0.1+ implementations.

### Why no "now" indicator

PAC has no concept of calendar time. Showing a "now" line requires knowing what day it is and mapping that to a position on the relative timeline. Without a calendar, "now" would need to be manually set by the user, which adds friction and introduces stale-indicator risk (forgetting to update it). The structural Blocked alert communicates urgency ("this is blocking work, and you can act on it now") without requiring calendar knowledge.

---

## 7. Sub-project Rules

### 7.1 Definition

A sub-project is a full project record stored in the same `pac.toml` file as its parent, linked to a specific task in the parent project. Sub-projects may be nested to a maximum depth of 10 levels.

In v0.0.1, all projects (top-level and sub-projects) in a single `pac.toml` MUST share the same `time_unit`. Mixed-unit project trees are deferred to a future version.

Because all `[[project]]` entries share a single file, all IDs across all projects, tracks, tasks, and milestones MUST be unique within the file (see §8.6). This applies across the top-level project and all sub-projects at every depth.

### 7.2 Converting a Task to a Sub-project

When a task is converted to a sub-project, a conformant implementation MUST:

1. Create a new sub-project with the same `name`, `description`, and `emoji` as the task.
2. Pre-populate the sub-project with one task: a clone of the original task. The clone's `dependencies` MUST be cleared (set to `[]`); dependencies on tasks in the parent project would violate the dependency scoping rule (§3.3) and are not carried over.
3. Set `task.sub_project_id` on the parent task to the new sub-project's ID.

After conversion the parent task's `plan.duration` MAY be cleared (it is ignored during scheduling when `sub_project_id` is set). The task's `actual.progress` becomes read-only, driven by sub-project completion.

### 7.3 Unconverting

Unconverting collapses the sub-project back into the single linked task in the parent project. This is a destructive action that MUST require explicit confirmation. A conformant implementation MUST:

1. Compute the sub-project's total duration: `max(EF)` across all tasks and milestones in the sub-project after running its forward pass and parallelism constraint (per §5).
2. Write that duration as the linked task's `plan.duration`, replacing any prior value.
3. Merge fields from the sub-project into the linked task:
   - `description`: if the task has none, copy the sub-project's. If both exist, append the sub-project's to the task's, separated by a blank line.
   - `notes`: if the task has none, copy the sub-project's. If both exist, append the sub-project's to the task's, separated by a blank line.
   - `emoji`: if the task has none, copy the sub-project's. If both exist, keep the task's own `emoji` unchanged.
4. Clear `sub_project_id` from the linked task. The task's `actual.progress` reverts to user-controlled.
5. Delete the sub-project record and all its contents (tracks, tasks, milestones, and any nested sub-project records) recursively.

The linked task's `dependencies` in the parent project are unchanged.

### 7.4 Navigation

A conformant implementation SHOULD support navigating into and out of sub-projects. When inside a sub-project, the full path from the top-level project SHOULD be visible and each item SHOULD be navigable.

### 7.5 Bidirectional Link: Source of Truth

The sub-project relationship is stored on both sides:
- `task.sub_project_id`: on the task in the parent project
- `project.parent_project_id` + `project.linked_task_id`: on the sub-project

**`task.sub_project_id` is the authoritative side.** On file load, if the two sides disagree (e.g. due to manual TOML editing), `task.sub_project_id` MUST win and the sub-project's `parent_project_id` / `linked_task_id` MUST be corrected to match. A conformant implementation SHOULD surface a warning to the user when this correction is made.

**Orphaned references:** If a sub-project's `linked_task_id` or `parent_project_id` refers to an entity that does not exist in the file, the file is malformed. A conformant implementation MUST surface a validation error and MUST NOT attempt to load or partially load the file. The user MUST manually correct the file before it can be loaded. The same rule applies when `task.sub_project_id` references a sub-project that does not exist in the file.

### 7.6 Circular Dependency Prohibition

A task inside a sub-project MUST NOT depend on any task in its parent project or any ancestor project. This would create a circular scheduling dependency: the sub-project's duration drives the parent linked task's duration, so a sub-project task depending on the parent would leave the scheduler with no stable starting point.

A conformant implementation MUST prevent this at the point of dependency creation, not at save time.

---

## 8. File Format

### 8.1 Filename

The PAC file MUST be named **`pac.toml`**. The filename is fixed and not user-chosen, like `Cargo.toml` or `Makefile`. When saving a new project, a conformant implementation MUST default the filename to `pac.toml`. Two projects in different directories each have their own `pac.toml`; the directory distinguishes them.

### 8.2 Structure

The `pac.toml` file uses the [TOML](https://toml.io) format. The file is designed so that small changes produce small, readable diffs. Every formatting rule below serves that goal.

**Sections:**
- A `[meta]` section at the top of the file (before any `[[project]]` block)
- One `[[project]]` block per project (top-level and sub-projects)
- `[[project.track]]`, `[[project.task]]`, and `[[project.milestone]]` sections within each project

**Structural rules:**
- Fields within each section MUST be written one per line
- Fields MUST be in strict **alphabetical key order**
- No inline tables `{ }`: every value on its own line
- Records MUST be separated by a single blank line

**TOML nesting vs `project_id`:** In TOML, `[[project.task]]` entries structurally belong to the most recently opened `[[project]]` block. The `project_id` field on tracks and milestones is redundant with this structural nesting but MUST be written anyway (it makes the file readable in isolation without tracking structural context). Tasks do not carry `project_id`. On load, the TOML structural parent is authoritative. If `project_id` on a track or milestone disagrees with the structural parent, a conformant implementation MUST correct `project_id` to match the structural parent and MAY log a warning.

### 8.3 Array Rules

**Format:** Empty arrays are written inline (`field = []`). Non-empty arrays are written multiline: one item per line, trailing comma on the last item, closing bracket on its own line:

```toml
dependencies = [
  "01HXYZ3NDEKTSV4RRFFQ69G5F1",
  "01HXYZ3NDEKTSV4RRFFQ69G5F2",
]
```

This ensures that adding or removing a single item is always a **one-line diff**.

**Always-written rule:** The `dependencies` array MUST always be written, even when empty. All other optional array fields MUST be omitted entirely when empty, per §8.4.

### 8.4 Null and Empty Field Rules

Optional fields that are null or empty string MUST be **omitted entirely** from the file. Do not write `field = ""` or `field = null`.

Exception: `dependencies = []` MUST always be written, even when empty, so its presence is explicit.

### 8.5 Record Ordering

- Tasks within a project MUST be sorted by `id` alphabetically; not by display order. Reordering tasks in the UI MUST NOT change the file.
- Milestones within a project MUST be sorted by `id` alphabetically.
- Tracks within a project MUST be sorted by `id` alphabetically. The `order` field controls display order separately.
- Sub-projects MUST immediately follow their parent project in the file. The ordering is depth-first: a sub-project and its entire subtree appear before the next sibling sub-project. Example: A has two sub-projects B and C; B itself has sub-project D:
  ```
  [[project]]  # A
  [[project]]  # B  (first child of A)
  [[project]]  # D  (child of B: B's subtree before next sibling)
  [[project]]  # C  (second child of A)
  ```

### 8.6 Identifier Format

An `id` MUST be a non-empty string that is unique within the file. A conformant implementation MUST NOT reuse or reassign an identifier once written. The RECOMMENDED format is [ULID](https://github.com/ulid/spec): a 26-character uppercase [Crockford Base32](https://www.crockford.com/base32.html) string. Example: `"01HXYZ3NDEKTSV4RRFFQ69G5FA"`. Implementations MAY use other formats (UUID, nanoid, sequential integers as strings, etc.) provided uniqueness and stability are guaranteed.

### 8.7 Parallelism Sentinel

The value `0` for the `parallelism` field means unlimited parallelism. A conformant implementation MUST write `parallelism = 0` (not a large number) to the file when the user selects unlimited parallelism.

### 8.8 Meta Section

A `[meta]` section MUST be written at the top of every `pac.toml` file:

```toml
[meta]
spec_version = "0.0.1"
```

| Field | Required | Description |
|---|---|---|
| `spec_version` | Yes | Semver string identifying the PAC specification version used to write this file. A conformant implementation MUST read this field before parsing any other section. |

### 8.9 Versioning and Migration

On file load, a conformant implementation MUST read `spec_version` before parsing any other section.

**Compatibility rule:** Two `spec_version` values are compatible if they share the same major version number (following [Semantic Versioning](https://semver.org/)). Patch differences are always backwards compatible.

**Minor version rule:** A minor version increment MUST NOT introduce a breaking change unless a machine-readable migration script is published alongside the new version. If no migration script is provided or possible, the change MUST be made backwards compatible, or deferred to the next major version increment. Files produced under any 0.x version are loadable by any other 0.x implementation (with unknown fields ignored); if a minor bump requires a migration, the migration script handles that transition.

| Condition | Required behaviour |
|---|---|
| Same major version, same or older minor/patch | Parse normally |
| Same major version, newer minor, no migration script | Parse normally; ignore unknown fields |
| Same major version, newer minor, migration script available | Run the migration script, then parse normally |
| Older major version | Run the appropriate migration function(s) before loading |
| Newer major version | Show a warning; attempt to load, ignoring unknown fields |

Migration functions MUST be documented and testable independently.

**Unknown fields rule:** A conformant implementation MUST ignore unknown fields it does not recognise (treat them as opaque round-trip data, preserved on save). This is what allows future versions to add fields (e.g. `plan.optimistic`, `plan.pessimistic` for PERT) without breaking v0.0.1 files.

**Enum fields:** If an enum field (such as `time_unit`) contains a value not defined in this specification, a conformant implementation MUST surface a clear error and MUST NOT silently default or ignore it. Unknown enum values cannot be safely ignored the way unknown string fields can.

### 8.10 Example

A complete `pac.toml` showing all entity types:

```toml
[meta]
spec_version = "0.0.1"

[[project]]
color = "#3B82F6"
id = "01ARZ3NDEKTSV4RRFFQ69G5FA0"
name = "API Launch"
time_unit = "days"

[[project.track]]
color = "#10B981"
id = "01ARZ3NDEKTSV4RRFFQ69G5FB0"
name = "Backend"
order = 0
parallelism = 1
project_id = "01ARZ3NDEKTSV4RRFFQ69G5FA0"

[[project.task]]
dependencies = []
id = "01ARZ3NDEKTSV4RRFFQ69G5FC0"
name = "Design review"
priority = 10
track_id = "01ARZ3NDEKTSV4RRFFQ69G5FB0"

plan.duration = 3

[[project.task]]
dependencies = [
  "01ARZ3NDEKTSV4RRFFQ69G5FC0",
]
id = "01ARZ3NDEKTSV4RRFFQ69G5FC1"
name = "Implementation"
priority = 50
track_id = "01ARZ3NDEKTSV4RRFFQ69G5FB0"

plan.duration = 5

actual.offset = 1
actual.progress = 40

[[project.milestone]]
color = "#F59E0B"
dependencies = [
  "01ARZ3NDEKTSV4RRFFQ69G5FC0",
  "01ARZ3NDEKTSV4RRFFQ69G5FC1",
]
id = "01ARZ3NDEKTSV4RRFFQ69G5FD0"
name = "Backend complete"
project_id = "01ARZ3NDEKTSV4RRFFQ69G5FA0"
```

Note: `description`, `emoji`, `notes`, and other optional null fields are absent; this is correct behaviour. `actual.*` fields are absent on the not-started task because all `actual.*` keys MUST be omitted when a task has not started.

---

## 9. Scale Constraints

### 9.1 Architectural constraints

These are fixed by the design of the format, not by implementation capacity:

| Constraint | Rule |
|---|---|
| Top-level projects per file | Exactly 1 (projects where `parent_project_id` is null). If a file contains more than one project with `parent_project_id` omitted or null, a conformant implementation MUST surface a parse error and MUST NOT attempt to load the file. |
| Sub-project nesting depth | Maximum 10 levels. Top-level project is depth 0; each sub-project adds 1. A conformant implementation MUST validate depth on load and reject files with sub-project chains deeper than 10 with a validation error. |
| `emoji` field | Exactly 1 grapheme cluster. A conformant implementation MUST reject an `emoji` value containing more than one grapheme cluster with a validation error. |
| `time_unit` uniformity | All projects in a single file MUST share the same `time_unit`. Mixed-unit files MUST be rejected with a validation error. |

### 9.2 Implementation limits

Implementations have different performance characteristics and target different use cases. This specification does not mandate specific numeric limits beyond the architectural constraints above.

A conformant implementation MUST:
- Document its own limits clearly (e.g. maximum tasks, maximum file size)
- Surface a clear error when a user action would exceed a limit
- MUST NOT silently truncate, corrupt, or partially save data when a limit is reached

A conformant implementation SHOULD degrade gracefully as files grow, rather than failing abruptly at an arbitrary threshold.

---

## 10. Field Reference

Complete type, nullability, default, and TOML format for every stored field in v0.0.1. Fields are listed alphabetically within each entity type.

**Authority split:** The prose sections above are normative for behaviour (algorithms, rules, ordering). This appendix is normative for types and defaults. If a conflict exists between the two, it is a spec bug; please file an issue at github.com/projectascode/spec.

### 10.1 Project Fields

| Field | Type | Nullable | Default | TOML format |
|---|---|---|---|---|
| `color` | Hex string | Yes (omit) | `"#3B82F6"` | `"#3B82F6"` |
| `description` | String | Yes (omit) | (none) | `"..."` |
| `emoji` | String | Yes (omit) | (none) | single grapheme cluster |
| `id` | String | No | Generated | `"01HXYZ..."` |
| `linked_task_id` | String | Yes (omit if top-level) | null | `"01HXYZ..."` |
| `name` | String | No | (none) | `"My Project"` |
| `parent_project_id` | String | Yes (omit if top-level) | null | `"01HXYZ..."` |
| `time_unit` | Enum string | No | `"days"` | `"hours"` or `"days"` |

### 10.2 Track Fields

| Field | Type | Nullable | Default | TOML format |
|---|---|---|---|---|
| `color` | Hex string | Yes (omit) | `"#10B981"` | `"#10B981"` |
| `description` | String | Yes (omit) | (none) | `"..."` |
| `emoji` | String | Yes (omit) | (none) | single grapheme cluster |
| `id` | String | No | Generated | `"01HXYZ..."` |
| `name` | String | No | (none) | `"Frontend"` |
| `order` | Integer | No | Append order | `0` |
| `parallelism` | Integer ≥ 0 | No | 1 | `2` (or `0` for unlimited) |
| `project_id` | String | No | (none) | `"01HXYZ..."` |

### 10.3 Task Fields

#### Structural fields

| Field | Type | Nullable | Default | TOML format |
|---|---|---|---|---|
| `dependencies` | String[] | No | `[]` | multiline array, always written |
| `description` | String | Yes (omit) | (none) | `"..."` |
| `emoji` | String | Yes (omit) | (none) | single grapheme cluster |
| `id` | String | No | Generated | `"01HXYZ..."` |
| `name` | String | No | (none) | `"CI Setup"` |
| `notes` | String | Yes (omit) | (none) | `"..."` |
| `priority` | Integer -100–100 | Yes (omit when 0) | 0 | `10` |
| `sub_project_id` | String | Yes (omit) | null | `"01HXYZ..."` |
| `track_id` | String | Yes (omit = unassigned) | null | `"01HXYZ..."` |

#### `plan.*` fields

| Field | Type | Nullable | Default | TOML format |
|---|---|---|---|---|
| `plan.duration` | Number > 0 | Yes (omit when `sub_project_id` set) | (none) | `3` or `1.5` |

#### `actual.*` fields

| Field | Type | Nullable | Default | TOML format |
|---|---|---|---|---|
| `actual.duration` | Number > 0 | Yes (omit when no override) | null | `2` or `3.5` |
| `actual.offset` | Number (signed) | Yes (omit when 0) | 0 | `1` or `-1` |
| `actual.progress` | Integer 0–100 | Yes (omit when 0) | 0 | `40` |

### 10.4 Milestone Fields

| Field | Type | Nullable | Default | TOML format |
|---|---|---|---|---|
| `color` | Hex string | Yes (omit) | `"#F59E0B"` | `"#F59E0B"` |
| `dependencies` | String[] | No | (must be non-empty) | multiline array, always written |
| `description` | String | Yes (omit) | (none) | `"..."` |
| `emoji` | String | Yes (omit) | (none) | single grapheme cluster |
| `id` | String | No | Generated | `"01HXYZ..."` |
| `name` | String | No | (none) | `"Design Complete"` |
| `project_id` | String | No | (none) | `"01HXYZ..."` |
