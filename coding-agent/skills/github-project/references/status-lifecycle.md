# Status Lifecycle Reference

Status is tracked as a single-select field on the GitHub Project. Each value below is an option in the project's "Status" field.

## Status Convention

Statuses follow the format `<role-slug>:<persona>:<activity>`:

- **Role slug** — which member role owns the status: `eng` (engineer), `cos` (chief-of-staff), `snt` (sentinel), `human` (human gates)
- **Persona** — which hat context is active (e.g., `po`, `arch`, `dev`, `qe`)
- **Activity** — what is being done (e.g., `triage`, `design`, `implement`)

Exceptions: `done` and `error` have no role owner.

## Issue Types (GitHub Native)

Classification uses GitHub's native issue types:

- **Epic** — top-level work item (epic)
- **Task** — child work item (story/subtask), linked as native sub-issue
- **Bug** — bug requiring investigation and fix

Stories are linked to epics as native sub-issues.

## Epic Lifecycle (Epic type)

```
eng:po:triage
    ↓
eng:po:backlog
    ↓
eng:arch:design
    ↓
eng:lead:design-review
    ↓
human:po:design-review (human gate)
    ↓
eng:arch:plan
    ↓
eng:lead:plan-review
    ↓
human:po:plan-review (human gate)
    ↓
eng:arch:breakdown
    ↓
eng:lead:breakdown-review
    ↓
eng:po:ready
    ↓
eng:arch:in-progress
    ↓
human:po:accept (human gate)
    ↓
done
```

## Story Lifecycle (Task type, sub-issue of Epic)

```
eng:qe:test-design
    ↓
eng:dev:implement
    ↓
eng:dev:code-review
    ↓
eng:qe:verify
    ↓
eng:arch:sign-off (auto-advance to sentinel)
    ↓
snt:gate:merge (sentinel runs merge gates)
    ↓
done
```

## Bug Lifecycle (Bug type)

### Simple Track (80% of bugs)

QE fixes directly, arch reviews, QE validates.

```
eng:bug:investigate
    ↓
eng:arch:review
    ↓ (approve → eng:qe:verify)
    ↓ (escalate → eng:arch:refine, becomes complex track)
eng:qe:verify
    ↓
done
```

### Complex Track (20% of bugs)

QE plans, arch refines, PO approves, arch creates subtasks.

```
eng:bug:investigate
    ↓
eng:arch:refine
    ↓
human:po:plan-review (human gate)
    ↓ (reject → eng:arch:refine)
eng:bug:breakdown
    ↓
eng:bug:in-progress (monitor subtask completion)
    ↓
eng:qe:verify
    ↓
done
```

Subtasks created during `eng:bug:breakdown` are Task-type sub-issues that flow through the story lifecycle.

## Human Gates

Human approval is required at these statuses (prefixed with `human:`):

1. **human:po:design-review** — PO reviews and approves design (epics)
2. **human:po:plan-review** — PO reviews and approves plan (epics and complex bugs)
3. **human:po:accept** — PO accepts completed work (epics)

All other transitions auto-advance without human-in-loop.

## Auto-Advance Statuses

- **eng:arch:sign-off** → `snt:gate:merge` (sentinel runs merge gates on PR)

## Sentinel Merge Gate

The sentinel role handles PR merge gating at `snt:gate:merge`:

1. Reads merge-gate configuration from `team/projects/<project>/knowledge/merge-gate.md`
2. Runs project-specific tests (e2e, exploratory, coverage)
3. If all pass → merges the PR, advances to `done`
4. If any fail → rejects, returns to `eng:dev:implement`

## Specialist Statuses

### SRE

```
eng:sre:infra-setup
    ↓
done
```

### Content Writing

```
eng:cw:write
    ↓
eng:cw:review
    ↓
snt:gate:merge (sentinel runs merge gates)
    ↓
done
```

Content stories are routed via the `kind/docs` label modifier.

## Chief of Staff Lifecycle

```
cos:exec:todo
    ↓
cos:exec:in-progress
    ↓
cos:exec:done
```

The chief of staff picks up `cos:exec:todo` items and transitions them through execution to completion.

## Rejection Loops

| Gate | Reject target |
|------|---------------|
| `human:po:design-review` | `eng:arch:design` |
| `human:po:plan-review` (epic) | `eng:arch:plan` |
| `human:po:plan-review` (bug) | `eng:arch:refine` |
| `human:po:accept` | `eng:arch:in-progress` |
| `eng:arch:review` (escalate) | `eng:arch:refine` |
| `snt:gate:merge` (reject) | `eng:dev:implement` |
