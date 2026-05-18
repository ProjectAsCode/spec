# Future Ideas

A catalog of features intentionally deferred from v0.0.1, with notes on how each can be added later without breaking existing files. This is a memory aid: when revisiting the spec for v0.1, v0.2, or beyond, start here.

For each item: what it was, why it was deferred, when to consider re-adding, and how to add it without breaking v0.0.1 files.

Items are not in priority order. Pick what to add based on user feedback after v0.0.1 ships.

---

## 1. PERT three-point estimates

**What:** Three duration estimates per task (`plan.optimistic`, `plan.most_likely`, `plan.pessimistic`) and the formula `E = (O + 4M + P) / 6` used as the task's effective duration.

**Deferred because:** Single `plan.duration` is sufficient for the core use case (track activities, see when they finish). Three estimates added cognitive load and field-table size; the value is real but niche.

**When to consider:** When users start asking "how do I express uncertainty in my estimates?" or when they want to drive the probability table or path-type views (items 2 and 3 below depend on this).

**How to add without breaking v0.0.1:**
- Add optional fields `plan.optimistic` and `plan.pessimistic` alongside the existing `plan.duration`.
- `plan.duration` stays as the canonical anchor value. When all three are present, treat `plan.duration` as the PERT M (most likely): `E = (plan.optimistic + 4 × plan.duration + plan.pessimistic) / 6`.
- Validation: `plan.optimistic <= plan.duration <= plan.pessimistic` (when all three are present).
- v0.0.1 files with only `plan.duration` continue to work; effective duration is just `plan.duration` (equivalent to E when O = M = P).

---

## 2. PERT probability calculation

**What:** `P(finish ≤ T)` using normal distribution approximation over the critical path's variance sum: `σ² = Σ ((P-O)/6)²`.

**Deferred because:** Depends on PERT (item 1) and critical path (item 4). The percentages are interesting to project managers but rarely consulted by engineering teams.

**When to consider:** After PERT three-point and float/critical path are added, this is a natural follow-up.

**How to add without breaking v0.0.1:**
- Pure derived computation from existing data. No file-format change.
- Implementations MAY surface a probability table; MAY use Abramowitz-Stegun for the normal CDF.

**Design notes:** Variance sum becomes ill-defined when the critical path contains sub-project-linked tasks (their internal variance is hidden at the parent level). Handle by excluding them and surfacing a note ("variance estimate excludes N sub-project-linked tasks on critical path").

---

## 3. Path types (Optimistic / Pessimistic views)

**What:** Three views of the same project: Normal (uses E), Optimistic (uses `plan.optimistic` for every task), Pessimistic (uses `plan.pessimistic`). Critical path may differ between views.

**Deferred because:** Depends on PERT (item 1). Most users only ever look at one view.

**When to consider:** After PERT is added, if users request scenario views.

**How to add without breaking v0.0.1:**
- Pure derived views. No file-format change.
- For sub-projects, the parent's selected path type drives the sub-project's path selection (same rule cascades down).

---

## 4. CPM backward pass, float, and critical path

**What:** Late Start (LS), Late Finish (LF), Float (= LS - ES), and the critical path (tasks with Float = 0).

**Deferred because:** The forward pass alone answers "when will each task finish." Backward pass adds substantial spec text and is only essential for identifying bottlenecks.

**When to consider:** When users start asking "which tasks are critical?" or "how much can task X slip without delaying the project?".

**How to add without breaking v0.0.1:**
- Pure derived computation. No file-format change. v0.0.1 files compute float correctly under the new spec.
- Add §5.3 (or wherever it fits) with the standard formulas:
  - `project_end = max(EF) across all tasks and milestones`
  - `LF(task) = min(LS(successor))` (or `project_end` if no successors)
  - `LS(task) = LF(task) - duration(task)`
  - `Float(task) = LS(task) - ES(task)`
- Tasks/milestones with Float = 0 are on the critical path.

**Design notes:** Under the v0.0.1 unified-schedule model (actual.* values feed into ES/EF), float reflects current reality, not the original plan. This is the right semantic: "how much slack does this task have NOW given everything we know."

---

## 5. "Ready (critical path)" alert

**What:** An alert that fires when a task is on the critical path, all predecessors are complete, and the task has not been started. "Start this now or the project slips."

**Deferred because:** Depends on critical path (item 4).

**When to consider:** Same time as item 4. They ship together naturally.

**How to add without breaking v0.0.1:**
- Add a new alert to §6 (alerts). Pure derived state.
- v0.0.1 files compute the new alert correctly under the new implementation.

---

## 6. Live overrun detection (the hard one)

**What:** Detect that an in-progress task is running slower than planned, before it completes. Old PAC had this as a "projected_finish" calculation: `units_elapsed / actual.progress` projects the total duration.

**Deferred because:** The projection formula requires a "now" position, which calendar removal eliminated. Without a calendar, we have no way to compute how much time has passed since a task started.

**Why it's hard to re-add:**
- Reintroducing a calendar reintroduces the partial-alignment UX risk that motivated the calendar removal.
- Adding a "now position" field that the user manually maintains has stale-indicator risk.
- A structural proxy (compare progress to some threshold) is too noisy to be useful.

**Options if pursued:**
- **(a) User-maintained `[meta].now_position`**: Manual integer/decimal; the user updates it during stand-ups. Stale risk acknowledged in docs.
- **(b) Git-derived position**: Tool infers elapsed working time from `git log` of the file's mtime since first `actual.progress` set. Heuristic, but no manual upkeep.
- **(c) `actual.progress_history` array**: `[(reported_at_position, progress_value), ...]`. Lets the tool compute pace directly. Adds file weight; durable.

**Recommendation if revisited:** Try option (b) first; falls back to option (a). Avoid putting calendar dates in the file format.

**No-break check:** Any of these are additive. v0.0.1 files (no now-position, no progress history) load fine; no overrun alerts fire without the new data.

---

## 7. Integer-arithmetic determinism mandate

**What:** The "MUST scale by 100 and use integer floor division" rule for bit-identical schedules across implementations.

**Deferred because:** Implementation detail bleeding into the spec. The spec text is large and most users don't care. Implementations can still be deterministic without the mandate.

**When to consider:** If two implementations produce visibly different schedules on the same file and users complain, formal determinism becomes worth the spec weight.

**How to add without breaking v0.0.1:**
- Add an appendix or §1.x sub-section with the rule. v0.0.1 files are unchanged.
- Implementations that already match the rule pass automatically; those that don't will need to adjust their math (no file change required).

---

## 8. Mixed `time_unit` in a project tree (sub-project unit conversion)

**What:** Parent project in days, sub-project in hours (or vice versa). Conversion uses `working_hours_per_day` to translate sub-project duration into parent units.

**Deferred because:** Requires `working_hours_per_day` field, the conversion rule, and handling of which project's `working_hours_per_day` drives the conversion. v0.0.1's "all projects in the file share one `time_unit`" is much simpler.

**When to consider:** When users request mixed-unit projects (typical case: a parent project tracking weeks/days with a sub-project tracking detailed hours).

**How to add without breaking v0.0.1:**
- Add optional `working_hours_per_day` field on Project. Default to a sensible value (e.g. 8) for top-level; sub-projects inherit from parent if omitted.
- Relax the v0.0.1 "uniform time_unit" rule: cross-unit trees become valid.
- Conversion rule: in §5.3 (sub-project duration in parent), when `time_unit` differs, convert the sub-project's `max(EF)` using the sub-project's `working_hours_per_day` (preserves the sub-project team's view).
- v0.0.1 files (uniform time_unit) still load and produce identical schedules.

**Design notes:** The previous spec resolved this with conversion-using-parent's-hours-per-day, then switched to sub-project's. The latter is cleaner. Reach this decision once before re-adding.

---

## 9. Display rounding rule

**What:** When fractional schedule values (e.g. EF = 3.4 days) are displayed in whole-unit UI contexts, round up to the next whole unit.

**Deferred because:** v0.0.1's single `plan.duration` field rarely produces fractional EF unless `actual.duration` or `actual.offset` use decimals.

**When to consider:** When PERT comes back (PERT E is fractional by construction) or when sub-project unit conversion is added (conversion can produce fractional values).

**How to add without breaking v0.0.1:**
- Add a one-paragraph "Display rounding" rule in §5. UI implementations round up; internal arithmetic preserves unrounded values.

---

## 10. Worked example in spec

**What:** A detailed worked example showing forward pass, parallelism, backward pass, and PERT probability with concrete numbers.

**Deferred because:** ~90 lines of spec text. Test scenarios cover the same ground more rigorously.

**When to consider:** Never in the normative spec, but a good fit for the `docs/` site or a "PAC by example" companion document.

**How to add without breaking v0.0.1:**
- Not part of the spec; lives in `docs/`. v0.0.1 files unaffected.

---

## 11. Three-group field expansion (`plan.*` enrichments)

The current `plan.*` group has only `plan.duration` but the namespace is reserved for future planning fields. Candidates worth considering:

- `plan.optimistic`, `plan.pessimistic` (PERT, item 1)
- `plan.buffer` (explicit padding for known risks)
- `plan.confidence` (free-text or enum: "low" / "medium" / "high")
- `plan.assumptions` (CommonMark string)

Each is additive. v0.0.1 implementations ignore unknown fields per §8.9.

---

## 12. `actual.*` group enrichments

Similar: the `actual.*` namespace is reserved for execution-reality fields. Candidates:

- `actual.start_position` and/or `actual.end_position`: absolute working-unit positions, for implementations that want to track them (currently `actual.offset` is relative to plan).
- `actual.progress_history`: array of `(position, progress)` tuples (see item 6).
- `actual.blockers`: array of free-text reasons a task is blocked.

Additive. v0.0.1 implementations ignore unknown fields.

---

## 13. Resource and cost modeling

**What:** Tasks consume specific named resources (a person, a license, a piece of equipment); each resource has finite capacity; the scheduler honors capacity.

**Deferred because:** Substantial complexity. Tracks in v0.0.1 are a lightweight slot counter for "how many tasks can this team do in parallel."

**When to consider:** When users hit the "I want to model that Alice and Bob are different and Alice can only do task A" wall.

**How to add without breaking v0.0.1:**
- Add a new entity type `[[project.resource]]` with id, name, capacity.
- Add optional `task.resources` field: array of resource IDs the task consumes.
- v0.0.1 files (no resources) load unchanged.

---

## 14. Cross-file dependencies

**What:** A task in one `pac.toml` depending on a task in another `pac.toml` (different repo, different directory).

**Deferred because:** Discovery / resolution semantics are unclear (paths? URLs? git refs?). Cross-file diffs lose locality.

**When to consider:** When multi-repo project teams ask for it. Probably v1.x territory.

**How to add without breaking v0.0.1:**
- Dependency entries in v0.0.1 are plain string IDs (local). Extension: allow a URI form `"<file-path>#<task-id>"` or `"pac://repo/path#task-id"`.
- v0.0.1 implementations reject unknown dependency formats; v0.x+ implementations handle them.
- This is the riskiest extension for compatibility; design carefully.

---

## 15. Baseline / approved-plan snapshots

**What:** A way to mark a specific snapshot of the plan as "the approved baseline" and compare current state to it.

**Deferred because:** Git tags already do this (`git tag plan-v1`). Adding a parallel mechanism in PAC would duplicate without benefit.

**When to consider:** Probably never as a file-format feature. Could be a tool-level convenience (auto-create git tags on user request).

---

## 16. Concurrent editing tooling

**What:** Tools for resolving merge conflicts in `pac.toml` files when two team members edit simultaneously.

**Deferred because:** Out of scope for the file format. The format design (alphabetical keys, multiline arrays) already reduces conflicts; tool-level helpers (custom merge drivers, visual conflict resolution) are the right layer.

**When to consider:** Tool-layer concern, not spec.

---

## Adding new alerts

The current §6 has one alert (Blocked). The deferred set above includes "Ready (critical path)" and "Overrun risk". The general pattern for adding alerts:

- Define the trigger in terms of existing fields and derived state (forward pass, float, sub-project completion).
- No file-format change.
- v0.0.1 files compute new alerts correctly under new implementations.

Candidate alerts beyond the deferred ones:

- **"Ahead of schedule"** (`actual.offset < 0` or `actual.duration < plan.duration` on completion): celebrate good news; can be used as a positive signal in the UI.
- **"Estimate revised"** (git history shows `plan.duration` changed on an in-progress or done task): flags drift between plan and execution.
- **"Long unstarted critical task"** (on critical path, not started, large `plan.duration` relative to project total): identifies the riskiest pending work.

---

## How to use this document

When considering a feature for a new minor version:

1. **Find the entry here.** If it's covered, the design hints save time.
2. **Verify the no-break check still holds.** Sometimes earlier minor versions add fields that change the compatibility surface.
3. **Update §1's "What v0.0.1 deferred" list** in [`SPEC/introduction.md`](SPEC/introduction.md) to remove the item (it's no longer deferred).
4. **Remove the entry from this document** once shipped, with the migration captured in the spec's CHANGELOG.

The goal: ship small, learn from real usage, grow what users actually want.
