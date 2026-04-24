---
id: 1
type: decision
status: accepted
date: 2026-04-24
participants: [operator, chief-of-staff]
supersedes: null
refs: [9]
---
# Add assignee labels for member-level issue routing

## Context

GitHub Apps (bot users) cannot be assigned to issues via GitHub's native assignee field — it's a platform limitation. When the board has many items, the operator needs a way to route specific issues to specific members and ensure they are picked up first.

## Decision

Add `assignee/<name>` labels using each member's short name from `botminter.yml`:

- `assignee/bob` — routes to engineer bob
- `assignee/kevin` — routes to chief-of-staff kevin
- `assignee/tom` — routes to sentinel tom

Each member's board scanner prioritizes issues with their matching `assignee/<name>` label above unlabeled issues. Within the labeled group, normal priority tables still apply.

Short names are used (not role-prefixed directory names) to align with `botminter.yml` `name` field and to avoid coupling to the directory naming convention (planned for removal per devguyio-bot-squad/devguyio-agentic-team#104).

The label separator uses `/` (not `:`) to be consistent with existing label conventions (`role/*`, `kind/*`, `project/*`).

## Alternatives Considered

- **GitHub native assignees**: Not possible — bot users can't be assigned.
- **`role/*` labels only**: Already exist but route to roles, not individual members. In a multi-member-per-role setup this wouldn't be specific enough.
- **Priority labels only** (`priority/top`): Doesn't solve routing — all members would compete for the same prioritized issues.

## Consequences

- Operator can now pin specific issues to specific members by adding a label.
- Unlabeled issues continue to be picked up by any member whose scanner matches the status prefix — no change to default behavior.
- When new members are hired, their `assignee/<name>` label should be added to `botminter.yml` and created on the repo.
