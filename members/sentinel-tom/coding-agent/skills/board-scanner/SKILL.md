---
name: board-scanner
description: >-
  Board scanning and dispatch procedure for GitHub Projects v2.
  Scans the project board for snt:* statuses and open PRs on project forks,
  dispatches work to pr_gate and pr_triage hats.
  Auto-injected into coordinator prompts.
metadata:
  author: botminter
  version: 1.0.0
---

# Board Scanner (Sentinel Scope)

This skill defines your PLAN step when coordinating. Scan the GitHub
Projects v2 board for `snt:*` statuses and project fork PRs, then
DELEGATE by publishing exactly one event to the appropriate hat.

## Scan Procedure

### 1. Scratchpad

Append a new scan section to the scratchpad with the current timestamp.
Delete `tasks.jsonl` if it exists to prevent state bleed from previous
hat activations.

### 2. Sync workspace

```bash
git -C team pull --ff-only 2>/dev/null || true
```

### 3. Auto-detect the team repo

```bash
TEAM_REPO=$(cd team && gh repo view --json nameWithOwner -q .nameWithOwner 2>/dev/null)
```

If `gh repo view` fails (e.g., remote is a local path), extract from git remote:

```bash
TEAM_REPO=$(cd team && git remote get-url origin | sed 's|.*github.com[:/]||;s|\.git$||')
```

### 4. Cache project IDs (once per scan cycle)

```bash
OWNER=$(echo "$TEAM_REPO" | cut -d/ -f1)
PROJECT_NUM=$(gh project list --owner "$OWNER" --format json --jq '.projects[0].number')
PROJECT_ID=$(gh project view "$PROJECT_NUM" --owner "$OWNER" --format json --jq '.id')
FIELD_DATA=$(gh project field-list "$PROJECT_NUM" --owner "$OWNER" --format json)
STATUS_FIELD_ID=$(echo "$FIELD_DATA" | jq -r '.fields[] | select(.name=="Status") | .id')
```

### 5. Query the project board

```bash
gh project item-list "$PROJECT_NUM" --owner "$OWNER" --format json
```

Parse the JSON to extract items with Status field values starting with `snt:`.

### 6. Log to poll-log.txt

Use `$(date -u +%Y-%m-%dT%H:%M:%SZ)` for all timestamps.

```
2026-03-02T10:15:00Z — board.scan — START
2026-03-02T10:15:01Z — board.scan — 1 snt issues found, 0 orphaned PRs
2026-03-02T10:15:01Z — board.scan — END
```

### 7. Assignee Label Priority

Before applying the dispatch table, check each actionable item for an
`assignee/<name>` label. Items labeled `assignee/tom` are prioritized
above unlabeled items. Within the labeled group, the normal priority
table still applies. Unlabeled items are dispatched after all labeled items.

### 8. Dispatch

Dispatch based on the highest-priority work found. Process one item at a time.

**Priority (highest first):**

| # | Status / Condition | Event |
|---|--------|-------|
| 1 | `snt:gate:merge` | `snt.gate` |
| 2 | Orphaned PRs detected on project forks | `snt.triage` |

No sentinel work found → emit `LOOP_COMPLETE`.

### 8. Periodic PR Triage

Every 10th scan cycle (or when no `snt:gate:merge` items are found), also
scan open PRs on all configured project forks for orphaned PRs. If found,
dispatch `snt.triage`.

## Idempotency

Before dispatching, verify the issue is not already at the target output
status. If it is, skip it and check the next issue.

Include the issue number in the published event context so downstream hats
know which issue to work on.

## Failed Processing Escalation

Before dispatching, count comments matching `Processing failed:` on the issue.

- Count < 3 → dispatch normally.
- Count >= 3 → set the issue's project status to `error` (via item-edit),
  skip dispatch, add a comment:
  `"Issue #N failed 3 times: [last error]. Status set to error. Please investigate."`
  If RObot is enabled, also send a `ralph tools interact progress` notification.

Skip items with Status `error` during dispatch.

## Error Handling

If any `gh` command fails during the scan:

1. Log the error to `errors-log.txt` with the full command and output.
2. If the failure is on a specific issue (item-edit, issue comment), skip
   that issue and continue scanning the rest.
3. If the failure is systemic (project not found, auth failure), emit
   `LOOP_COMPLETE` and log the reason.

## Comment Format

All board scanner comments use:

```
### 🛡️ sentinel — $(date -u +%Y-%m-%dT%H:%M:%SZ)
```

## Rules

- ALWAYS log to poll-log.txt before publishing.
- Publish exactly ONE event per scan cycle to dispatch work.
- When no work is found, emit `LOOP_COMPLETE`.
- NEVER merge a PR without running the project's merge-gate tests first.
