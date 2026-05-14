---
type: plan
epic: "114"
title: Executable Invariant Checks
author: bob
date: 2026-05-14
stories: 6
---

# Story Breakdown: Executable Invariant Checks (Epic #114)

## Overview

This plan decomposes the design for Executable Invariant Checks into 6 stories. The design adds executable check scripts with structured, agent-readable output and a check runner that integrates into the `dev_code_reviewer` and `qe_verifier` hats. All 4 check scripts are project-specific (`projects/botminter/invariants/checks/`). The check runner is team-level infrastructure (`team/coding-agent/skills/check-runner/`).

The ordering places foundational infrastructure first (contract, runner), then the check scripts in parallel-safe order, then integration and documentation last.

---

## Dependency Graph

```
Story 1 (Contract doc)
  └─> Story 2 (Check runner)
        └─> Story 3 (domain-layer-imports.sh + no-hardcoded-profiles.sh)
        └─> Story 4 (file-size-limit.sh + test-path-isolation.sh)
              └─> Story 5 (Hat integration)
                    └─> Story 6 (CLAUDE.md update)
```

Stories 3 and 4 can execute in parallel once Story 2 is complete.

---

## Story 1: Check Script Contract Documentation

**Type:** Task (docs)
**Parent:** Epic #114
**Labels:** `project/botminter`, `kind/docs`
**Depends on:** None (foundational)

### Description

Create the team-level knowledge document `team/knowledge/check-script-contract.md` that defines the contract all check scripts must follow. This is the foundational reference that the runner and all check scripts depend on.

### Deliverables

- `team/knowledge/check-script-contract.md` covering:
  - Output format: `VIOLATION`, `RULE`, `REMEDIATION`, `REFERENCE` lines
  - Exit code semantics: 0 = pass, 1 + VIOLATION = violation, 1 without VIOLATION or >1 = crash
  - Working directory convention: runner sets `cwd = projects/<project>/` before each script
  - Suppression syntax: `// check:ignore` inline comment excludes a line from violations
  - Script location convention: project-specific in `projects/<project>/invariants/checks/`, team-generic in `team/invariants/checks/` (deferred)
  - Discovery convention: runner discovers all `*.sh` files in the checks directories

### Acceptance Criteria

- **Given** an agent needs to write a new check script, **when** it reads `team/knowledge/check-script-contract.md`, **then** it finds the output format, exit code semantics, suppression syntax, and working directory convention without ambiguity.
- **Given** the contract document exists, **when** it is reviewed, **then** it contains a complete example of a passing check, a violation output, and a crash scenario.
- **Given** the `// check:ignore` suppression syntax is documented, **when** an agent reads the contract, **then** it understands that `// check:ignore` on a line causes that line to be excluded from grep-based violation matching.

---

## Story 2: Check Runner Script

**Type:** Task
**Parent:** Epic #114
**Labels:** `project/botminter`
**Depends on:** Story 1

### Description

Implement the check runner at `team/coding-agent/skills/check-runner/run-checks.sh`. The runner discovers and executes check scripts, classifies results (pass / violation / crash), and produces a structured aggregate exit code.

### Deliverables

- `team/coding-agent/skills/check-runner/run-checks.sh` implementing:
  - Accepts a project name argument (e.g., `botminter`)
  - Discovers `*.sh` files in `projects/<project>/invariants/checks/` (and `team/invariants/checks/` if it exists)
  - Sets `cwd = projects/<project>/` before executing each script
  - Captures stdout and exit code for each script
  - Classifies: exit 0 = pass, exit 1 + VIOLATION on stdout = violation, exit 1 without VIOLATION or exit >1 = crash (logged as warning)
  - Aggregates: exit 0 if all pass, exit 1 if any violation
  - Reports crashed scripts as warnings without blocking
  - Prints a summary line at the end (e.g., "4 checks: 2 passed, 1 violation, 1 crash")

### Acceptance Criteria

- **Given** check scripts exist in `projects/botminter/invariants/checks/`, **when** `bash team/coding-agent/skills/check-runner/run-checks.sh botminter` is invoked, **then** each script runs with `cwd` set to the project root and the runner reports results per-script.
- **Given** a new `*.sh` file is added to the checks directory, **when** the runner executes, **then** the new script is discovered and run automatically without any runner modification.
- **Given** a check script exits with code 2 (or exit 1 without VIOLATION output), **when** the runner processes it, **then** the runner logs a warning for that script and continues with remaining checks, and the crash does not cause the runner to exit non-zero on its own.
- **Given** one script reports a VIOLATION (exit 1 with VIOLATION output) and two scripts pass, **when** the runner finishes, **then** it exits with code 1 and the summary shows the violation count.
- **Given** all scripts pass (exit 0), **when** the runner finishes, **then** it exits with code 0.

---

## Story 3: Domain-Layer and Hardcoded-Profile Check Scripts

**Type:** Task
**Parent:** Epic #114
**Labels:** `project/botminter`
**Depends on:** Story 2

### Description

Implement two check scripts that enforce ADR-0007 domain-command layering and the no-hardcoded-profiles invariant:

1. `projects/botminter/invariants/checks/domain-layer-imports.sh` -- Greps for `println!`, `eprintln!`, `use clap`, `use comfy_table`, `use dialoguer`, `use cliclack` in domain modules under `crates/bm/src/`, excluding command-layer files (`commands/`, `main.rs`, `cli.rs`, `agent_main.rs`, `agent_cli.rs`). Scans both directory modules and standalone domain `.rs` files. Supports `// check:ignore` suppression.

2. `projects/botminter/invariants/checks/no-hardcoded-profiles.sh` -- Scans for profile name strings (`scrum-compact`, `scrum`) in `.rs` files under `crates/bm/src/`, excluding `commands/`. Supports `// check:ignore` suppression.

### Deliverables

- `projects/botminter/invariants/checks/domain-layer-imports.sh` following the contract from Story 1
- `projects/botminter/invariants/checks/no-hardcoded-profiles.sh` following the contract from Story 1
- Both scripts produce VIOLATION/RULE/REMEDIATION/REFERENCE output per the contract
- Both scripts handle `// check:ignore` inline suppression
- Both scripts exit 0 when no violations are found

### Acceptance Criteria

- **Given** a domain module `crates/bm/src/bridge/provisioning.rs` contains `eprintln!` on line 81, **when** `domain-layer-imports.sh` runs, **then** it outputs a VIOLATION line naming the file, line number, and the prohibited call, with RULE referencing ADR-0007 and REMEDIATION advising to use `tracing` macros or return structured Result types.
- **Given** a line in a domain module contains `eprintln!("status")  // check:ignore`, **when** `domain-layer-imports.sh` runs, **then** that line is excluded from the violation report.
- **Given** the existing 9 `eprintln!` violations across 4 domain modules, **when** `domain-layer-imports.sh` runs, **then** all 9 are reported with correct file paths, line numbers, and remediation guidance.
- **Given** `crates/bm/src/profile/mod.rs` contains test fixtures with the string `"scrum"`, **when** `no-hardcoded-profiles.sh` runs, **then** violations are reported with RULE referencing the `no-hardcoded-profiles` invariant.
- **Given** command-layer files (`commands/init.rs`, `cli.rs`), **when** either script runs, **then** those files are excluded from scanning.

---

## Story 4: File-Size and Test-Path Check Scripts

**Type:** Task
**Parent:** Epic #114
**Labels:** `project/botminter`
**Depends on:** Story 2

### Description

Implement two check scripts that enforce the ADR-0007 file size limit and the test-path-isolation invariant:

1. `projects/botminter/invariants/checks/file-size-limit.sh` -- Checks `wc -l` on `.rs` files under `crates/bm/src/`. Reports files exceeding 300 lines as violations. Excludes `target/` and test fixtures. Counts all lines (no `#[cfg(test)]` exclusion per design).

2. `projects/botminter/invariants/checks/test-path-isolation.sh` -- Greps for `dirs::home_dir()` and `std::env::home_dir()` in `.rs` files under `crates/bm/tests/` only. Production code is excluded by design.

### Deliverables

- `projects/botminter/invariants/checks/file-size-limit.sh` following the contract from Story 1
- `projects/botminter/invariants/checks/test-path-isolation.sh` following the contract from Story 1
- Both scripts produce VIOLATION/RULE/REMEDIATION/REFERENCE output per the contract
- `file-size-limit.sh` uses the ~300 line soft threshold with violation reporting
- `test-path-isolation.sh` scans only `crates/bm/tests/`, not `crates/bm/src/`

### Acceptance Criteria

- **Given** `crates/bm/src/profile/extraction.rs` is 1508 lines, **when** `file-size-limit.sh` runs, **then** it reports a VIOLATION naming the file and its line count with RULE referencing ADR-0007 and REMEDIATION advising to decompose into sub-modules.
- **Given** the existing 47 files exceeding 300 lines, **when** `file-size-limit.sh` runs, **then** all 47 are reported as violations (confirming the design's baseline).
- **Given** a `.rs` file under `crates/bm/src/` has exactly 300 lines, **when** `file-size-limit.sh` runs, **then** it does NOT report that file as a violation (threshold is >300).
- **Given** `crates/bm/tests/integration.rs` contains `//! config via dirs::home_dir() are invoked...` on line 6 (a doc comment), **when** `test-path-isolation.sh` runs, **then** it reports a VIOLATION for that line (known false positive per design -- grep cannot distinguish doc comments from code).
- **Given** production code in `crates/bm/src/config/mod.rs` uses `dirs::home_dir()`, **when** `test-path-isolation.sh` runs, **then** that file is NOT scanned and NOT reported.

---

## Story 5: Hat Integration (dev_code_reviewer and qe_verifier)

**Type:** Task
**Parent:** Epic #114
**Labels:** `project/botminter`
**Depends on:** Stories 3, 4

### Description

Update the `dev_code_reviewer` and `qe_verifier` hat instructions to run the check runner before their core workflows. This is the integration point that makes checks actionable in the development lifecycle.

### Deliverables

- Updated `dev_code_reviewer` hat instructions (in ralph.yml or hat knowledge) adding:
  > Before reviewing, run: `bash team/coding-agent/skills/check-runner/run-checks.sh <project>`. If any VIOLATION is reported, reject to `dev:implement` with the VIOLATION/REMEDIATION output as feedback.
- Updated `qe_verifier` hat instructions adding:
  > As part of verification, run the check runner. Violations block verification.
- Both hats reference the check runner by its canonical path

### Acceptance Criteria

- **Given** the `dev_code_reviewer` hat is activated for a botminter story, **when** the hat reads its instructions, **then** it finds an explicit directive to run `bash team/coding-agent/skills/check-runner/run-checks.sh botminter` before reviewing code.
- **Given** the check runner reports a VIOLATION during code review, **when** `dev_code_reviewer` processes the output, **then** it rejects the story to `eng:dev:implement` with the VIOLATION and REMEDIATION lines included in the rejection comment.
- **Given** the `qe_verifier` hat is activated for a botminter story, **when** the hat reads its instructions, **then** it finds an explicit directive to run the check runner as part of verification, with violations blocking the `eng:qe:verify` pass.
- **Given** all checks pass during code review, **when** `dev_code_reviewer` proceeds, **then** the review continues normally without check-related blocking.

---

## Story 6: CLAUDE.md Update

**Type:** Task (docs)
**Parent:** Epic #114
**Labels:** `project/botminter`, `kind/docs`
**Depends on:** Story 5

### Description

Update `projects/botminter/CLAUDE.md` to reference the check runner, check script locations, and guidance for adding new check scripts. This ensures future agents working on botminter know about the executable invariant checks.

### Deliverables

- Updated `projects/botminter/CLAUDE.md` with:
  - Reference to the check runner as a pre-review step: `bash team/coding-agent/skills/check-runner/run-checks.sh botminter`
  - Reference to `invariants/checks/` as the location for project-specific check scripts
  - Guidance that new invariants should have corresponding check scripts when mechanically enforceable
  - Reference to `team/knowledge/check-script-contract.md` for the check script contract

### Acceptance Criteria

- **Given** an agent opens `projects/botminter/CLAUDE.md`, **when** it reads the file, **then** it finds a section or reference describing the invariant check runner and how to invoke it.
- **Given** a developer needs to add a new check script, **when** they read the CLAUDE.md, **then** they find a pointer to `team/knowledge/check-script-contract.md` for the output format and contract, and a pointer to `projects/botminter/invariants/checks/` for where to place the script.
- **Given** the CLAUDE.md references the check runner path, **when** compared to the actual runner location, **then** the path matches `team/coding-agent/skills/check-runner/run-checks.sh`.

---

## Summary

| # | Story | Type | Depends On | Key Deliverables |
|---|-------|------|-----------|------------------|
| 1 | Check Script Contract Documentation | docs | None | `team/knowledge/check-script-contract.md` |
| 2 | Check Runner Script | code | Story 1 | `team/coding-agent/skills/check-runner/run-checks.sh` |
| 3 | Domain-Layer and Hardcoded-Profile Checks | code | Story 2 | `domain-layer-imports.sh`, `no-hardcoded-profiles.sh` |
| 4 | File-Size and Test-Path Checks | code | Story 2 | `file-size-limit.sh`, `test-path-isolation.sh` |
| 5 | Hat Integration | config | Stories 3, 4 | Updated `dev_code_reviewer` and `qe_verifier` instructions |
| 6 | CLAUDE.md Update | docs | Story 5 | Updated `projects/botminter/CLAUDE.md` |

**Total:** 6 stories. Stories 3 and 4 can execute in parallel. All stories follow TDD flow: `eng:qe:test-design` -> `eng:dev:implement` -> `eng:dev:code-review` -> `eng:qe:verify`.
