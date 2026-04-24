---
name: board-scanner
description: >-
  Board scanning and dispatch procedure for GitHub Projects v2.
  Scans the project board for cos:exec:* statuses and dispatches work
  to the executor hat via priority table.
  Auto-injected into coordinator prompts.
metadata:
  author: botminter
  version: 1.0.0
---

# Board Scanner (Chief of Staff Scope)

This skill defines your PLAN step when coordinating. Scan the GitHub
Projects v2 board for `cos:exec:*` statuses, then DELEGATE by publishing
exactly one event to the executor hat.

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

Parse the JSON to extract items with Status field values starting with `cos:exec:`.

### 6. Log to poll-log.txt

Use `$(date -u +%Y-%m-%dT%H:%M:%SZ)` for all timestamps.

```
2026-03-02T10:15:00Z — board.scan — START
2026-03-02T10:15:01Z — board.scan — 1 cos issues found
2026-03-02T10:15:01Z — board.scan — END
```

### 7. Assignee Label Priority

Before applying the dispatch table, check each actionable item for an
`assignee/<name>` label. Items labeled `assignee/kevin` are prioritized
above unlabeled items. Within the labeled group, the normal priority
table still applies. Unlabeled items are dispatched after all labeled items.

### 8. Dispatch

Dispatch based on the `cos:exec:*` status found. Process one item at a time.

**Priority (highest first):**

| # | Status | Event |
|---|--------|-------|
| 1 | `cos:exec:todo` | `cos.execute` |
| 2 | `cos:exec:in-progress` | `cos.execute` |

No cos work found -> emit `LOOP_COMPLETE`.

## Idempotency

Before dispatching, verify the issue is not already at the target output
status. If it is, skip it and check the next issue.

Include the issue number in the published event context so downstream hats
know which issue to work on.

## Failed Processing Escalation

Before dispatching, count comments matching `Processing failed:` on the issue.

- Count < 3 -> dispatch normally.
- Count >= 3 -> set the issue's project status to `error` (via item-edit),
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
### 📋 chief-of-staff — $(date -u +%Y-%m-%dT%H:%M:%SZ)
```

## Rules

- ALWAYS log to poll-log.txt before publishing.
- Publish exactly ONE event per scan cycle to dispatch work.
- When no work is found, emit `LOOP_COMPLETE`.
