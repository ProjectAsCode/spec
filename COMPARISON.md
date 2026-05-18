# PAC vs Alternatives

A practical comparison of the PAC specification against the project-planning tools you are most likely choosing between. This document is intentionally not marketing: where PAC is worse than an alternative, we say so.

## What PAC is, in one paragraph

PAC stores a project plan as a single TOML file in your git repository. There is no backend, no account, no SaaS. You read and edit the file in any text editor, diff and review it like code, branch it to model alternative timelines, and tag it to mark approved baselines. Implementations (a CLI, a web viewer, a desktop app) read the file and render it; the data itself stays in your repo, owned by you, in a documented open standard.

## Comparison table

|   | PAC | Jira | Linear | GitHub Projects | Notion | Org-mode | Markdown task lists |
|---|---|---|---|---|---|---|---|
| **Storage** | file in your repo | SaaS database | SaaS database | GitHub database | SaaS database | file in your repo | file in your repo |
| **Format** | open TOML (this spec) | proprietary | proprietary | proprietary | proprietary | Org plain text | Markdown |
| **Diffable / reviewable** | yes, full git | API-only export | API-only export | partial | API-only export | yes, full git | yes, full git |
| **Branchable schedule variants** | yes (git branch) | no | no | no | no | yes | yes |
| **Audit trail** | git log | vendor changelog (cloud-only) | vendor changelog | partial | partial | git log | git log |
| **Real-time multi-user editing** | no (git merge) | yes | yes | yes | yes | no | no |
| **Mobile / web UI** | not yet | yes | yes | yes | yes | no | none |
| **Scheduling: dependencies + forward pass** | yes | weak | weak | minimal | minimal | weak | none |
| **Float / critical path** | deferred to v0.1 | yes (plugins) | no | no | no | partial | no |
| **PERT / probabilistic estimation** | deferred to v0.1 | plugins | no | no | no | no | no |
| **Resource modeling** | not in scope for v0.0.1 | yes | partial | no | no | partial | no |
| **Native integrations** | none (git-adjacent only) | many | many | GitHub-native | many | few | none |
| **Cost** | free (open standard) | per-seat subscription | per-seat subscription | free with GitHub | per-seat subscription | free | free |
| **Vendor lock-in** | none (open standard, your file) | high | medium | medium-high | high | low | none |
| **Offline access** | yes (it is a file) | partial | partial | no | partial | yes | yes |
| **Programmatic access** | parse the TOML | REST API + auth | GraphQL API | REST API | API | parse the file | parse the file |
| **Suitable scale** | small / medium teams; one project tree per file | enterprise | small to mid-size | small | small | individual to small team | individual |

## When PAC is the right choice

- **Your team already lives in git.** If "pull request" is a verb your team uses every day, PAC fits the muscle memory you already have.
- **Your plan and your code share a lifecycle.** When a feature is delivered or descoped, the same commit that updates the code can update the plan. The two histories are one history.
- **You want a permanent, vendor-free record.** PAC files outlive any tool. If projectascode.org went away tomorrow, your `pac.toml` is still a valid, parseable file under a CC-BY license.
- **Small-to-mid team sizes.** Up to roughly 50 people across a few coordinated projects, where dependencies are knowable and the dependency graph fits in a head or a diagram.
- **You want diffable, reviewable planning.** "Why did we slip three weeks?" gets answered by `git blame` on the plan file, with the commit message and the reviewer thread attached.

## When PAC is the wrong choice

- **Your team includes non-engineers who would never touch a text file.** Until the reference UI ships, the raw file is impenetrable to people who don't read TOML. Even after a UI ships, the git workflow (commit, push, pull) is a real adoption tax.
- **You need real-time multi-user editing.** If two people need to update the plan simultaneously and see each other's edits live, PAC is not it. Git merge conflicts will be your enemy.
- **You depend on third-party integrations** for Slack / Salesforce / time-tracking / etc. PAC has none yet, and adding them is a per-tool effort.
- **You manage > 200 people across many cross-organisational dependencies.** The single-file-per-project-tree model is not designed for enterprise-scale coordination.
- **You need rich mobile UX.** No mobile clients exist for PAC. Editing TOML on a phone is unpleasant; no native mobile app is on the near-term roadmap.

## What you give up

Compared with a tool like Jira or Linear, the three big things you give up are:

1. **Real-time collaboration.** Git is the synchronisation mechanism. People who edit the plan need to pull-then-push, and the workflow has the same collision risk as any other code edit. Most teams find this manageable; some teams find it a deal-breaker.
2. **Mobile / web UX out of the box.** Until the reference tooling ships, the spec alone is not consumable by anyone unfamiliar with the file format. You can ship without a UI for some workflows (engineering planning) but not all (cross-functional planning meetings).
3. **An ecosystem of integrations.** No Slack bot, no Salesforce sync, no time-tracking plugin. Building these is feasible (the file is parseable) but it has not been done.

## What you gain

The same things git gives the code it stores, applied to project planning:

- **`git blame` on every line of your plan.** Who changed an estimate, when, why (the commit message).
- **`git diff` between any two points.** What tasks were added, what shifted, what was descoped between sprints.
- **Pull requests for schedule changes.** Reviewable like code. Mergeable when the team agrees.
- **Branches for what-if analysis.** Create `plan/aggressive-timeline` or `plan/reduced-scope`, explore, merge or discard.
- **Tags as approved baselines.** Mark the plan at kickoff as `plan-baseline-v1`. Always recoverable.
- **No vendor risk.** The format is documented. If PAC's reference tools disappear, your file is still text you can open in any editor.

## Migration sketches

### Coming from Jira

Export your active project's issues as JSON or CSV. Map: Jira issues → PAC tasks, sprints → milestones, Jira epics → sub-projects, story points → `plan.duration` (after picking a unit), assignees → `track_id` references. Things you will lose on the way over: comments, attachments, custom fields, workflow states beyond Not Started / In Progress / Done (cancelled work is deleted from the file in PAC; its history lives in git), and (importantly) the live collaboration. Things you will gain: history that lives with your code, no per-seat fee, and a plan that survives any vendor change.

### Coming from Linear

Similar to Jira but cleaner because Linear's data model is closer to PAC's. Linear cycles map to milestones. Linear projects map to PAC sub-projects. You will lose Linear's tight GitHub integration (PR linking, branch automation) unless you reimplement those hooks against the PAC file. You will lose Linear's web UI quality until equivalent PAC tooling exists.

### Coming from Markdown task lists

This is the easiest migration because you are already plain-text-native. Each bullet becomes a task; you add dependencies, durations, and tracks. The change is from unstructured text ("nice-to-have") to structured fields ("scheduler-friendly"). The tool layer can be incremental: today you keep your Markdown notes alongside the PAC file; over time the PAC file absorbs the structured parts.

### Coming from Org-mode

Org-mode users are the closest to PAC's mindset already: file-based, git-friendly, plain text. The migration is mainly a syntax port and an opinion port (Org's free-form properties become PAC's typed fields, gaining validation but losing flexibility). Worth doing if you collaborate with non-Emacs users; possibly not worth doing if your team is fully on Org.

---

For the spec itself, see [SPEC.md](SPEC.md). For features deliberately deferred from v0.0.1, see [FUTURE.md](FUTURE.md).
