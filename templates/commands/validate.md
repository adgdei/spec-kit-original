---
description: "End-to-end validation skill: verify the implementation matches the specification and plan. Runs cross-artifact analysis, test validation, checklist review, and completeness checks."
scripts:
  sh: scripts/bash/check-prerequisites.sh --json --require-tasks --include-tasks
  ps: scripts/powershell/check-prerequisites.ps1 -Json -RequireTasks -IncludeTasks
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Purpose

This is a **deployment skill** that orchestrates the verification phase of the Speckit workflow. It performs cross-artifact consistency analysis, validates implementation completeness, reviews quality checklists, and confirms everything works before marking the feature done.

**Workflow position**: This is step 4 of 4 in the full Speckit skill pipeline:

```
/speckit.brainstorm → /speckit.design → /speckit.build → /speckit.validate
```

## Outline

### Phase 1: Setup & Context Loading

1. **Run** `{SCRIPT}` from repo root and parse FEATURE_DIR and AVAILABLE_DOCS list. All paths must be absolute.
   - For single quotes in args like "I'm Groot", use escape syntax: `"I'm Groot"`

2. **Load all artifacts** from FEATURE_DIR:
   - **Required**: spec.md, plan.md, tasks.md
   - **Optional**: data-model.md, contracts/, research.md, quickstart.md, checklists/

3. **Load** `/memory/constitution.md` if it exists.

4. **Verify prerequisites**: If any required file is missing, abort with specific guidance:
   - No spec.md → "Run `/speckit.brainstorm` or `/speckit.specify` first."
   - No plan.md → "Run `/speckit.design` or `/speckit.plan` first."
   - No tasks.md → "Run `/speckit.build` or `/speckit.tasks` first."

### Phase 2: Cross-Artifact Consistency Analysis

Perform the full `/speckit.analyze` detection suite:

1. **Build semantic models** (internal, not output):
   - Requirements inventory: each functional + non-functional requirement with a stable key
   - User story/action inventory: discrete user actions with acceptance criteria
   - Task coverage mapping: each task mapped to requirements/stories
   - Constitution rule set: principle names and MUST/SHOULD statements

2. **Run detection passes** (limit to 50 findings total):

   **A. Duplication Detection**
   - Near-duplicate requirements across spec, plan, tasks
   - Mark lower-quality phrasing for consolidation

   **B. Ambiguity Detection**
   - Vague adjectives (fast, scalable, secure) lacking measurable criteria
   - Unresolved placeholders (TODO, TKTK, ???, `<placeholder>`)

   **C. Underspecification**
   - Requirements with verbs but missing object or measurable outcome
   - User stories missing acceptance criteria
   - Tasks referencing undefined files or components

   **D. Constitution Alignment**
   - Requirements or plan elements conflicting with MUST principles
   - Missing mandated sections or quality gates

   **E. Coverage Gaps**
   - Requirements with zero associated tasks
   - Tasks with no mapped requirement/story
   - Non-functional requirements not reflected in tasks

   **F. Inconsistency**
   - Terminology drift (same concept named differently across files)
   - Data entities referenced in plan but absent in spec (or vice versa)
   - Task ordering contradictions
   - Conflicting requirements

3. **Assign severity**:
   - **CRITICAL**: Constitution MUST violations, missing core requirements, zero-coverage blockers
   - **HIGH**: Duplicate/conflicting requirements, ambiguous security/performance, untestable criteria
   - **MEDIUM**: Terminology drift, missing non-functional coverage, underspecified edge cases
   - **LOW**: Style/wording improvements, minor redundancy

### Phase 3: Implementation Completeness

1. **Task completion audit**:
   - Read tasks.md and count: total tasks, completed `[X]`, incomplete `[ ]`
   - For each incomplete task, note its phase and dependencies
   - Calculate completion percentage

2. **File existence check**:
   - For each task that references a specific file path, verify the file exists
   - Report missing files with their associated task IDs

3. **Test execution** (if test infrastructure exists):
   - Detect test framework from plan.md / project files
   - Run available test suites
   - Report: tests passed, failed, skipped, coverage (if available)
   - If no tests exist and spec requires them, flag as a gap

### Phase 4: Checklist Review

1. **Scan** all checklists in FEATURE_DIR/checklists/:
   - For each checklist, count total, completed `[X]`, incomplete `[ ]`
   - Create summary table:

     ```text
     | Checklist        | Total | Done | Remaining | Status |
     |------------------|-------|------|-----------|--------|
     | requirements.md  | 12    | 12   | 0         | PASS   |
     | ux.md            | 8     | 6    | 2         | FAIL   |
     | security.md      | 6     | 6    | 0         | PASS   |
     ```

2. **For failing checklists**: List the specific incomplete items so the user can address them.

### Phase 5: Specification Conformance

Validate that the implementation actually fulfills the specification:

1. **User story verification**: For each user story in spec.md:
   - Are all functional requirements covered by completed tasks?
   - Do the implemented files match the planned architecture?
   - Are acceptance scenarios testable with the current implementation?

2. **Success criteria check**: For each success criterion in spec.md:
   - Is it measurable with the current implementation?
   - Is there a test or validation mechanism for it?
   - Flag any criterion that cannot be verified

3. **Non-functional requirements**: Check each NFR:
   - Performance targets: are they testable?
   - Security requirements: are they implemented?
   - Accessibility requirements: are they addressed?

### Phase 6: Validation Report

Output a comprehensive validation report:

```
## Validation Report

### Artifact Consistency

| ID | Category | Severity | Location(s) | Summary | Recommendation |
|----|----------|----------|-------------|---------|----------------|
| A1 | Coverage | HIGH | spec.md:FR-003 | Requirement has no task | Add task for... |
| ... |

### Coverage Summary

| Requirement Key | Has Task? | Completed? | Task IDs | Notes |
|-----------------|-----------|------------|----------|-------|
| user-can-login  | Yes       | Yes        | T003     |       |
| user-can-export | Yes       | No         | T014     | In progress |
| ...             |

### Implementation Status
- Total tasks: [N]
- Completed: [X] ([%])
- Incomplete: [Y]
- Missing files: [Z]

### Test Results
- Tests run: [N]
- Passed: [X]
- Failed: [Y]
- Coverage: [%] (if available)

### Checklist Status
[checklist summary table]

### Constitution Alignment
- [PASS/issues listed]

### Metrics
- Total Requirements: [N]
- Total Tasks: [N]
- Coverage: [%] (requirements with >=1 completed task)
- Ambiguity Count: [N]
- Critical Issues: [N]

### Verdict

[ONE OF:]
- **READY**: All checks pass. Feature is complete and verified.
- **NEEDS WORK**: [N] issues to resolve before completion. [list critical items]
- **BLOCKED**: Critical issues prevent completion. [list blockers]

### Next Actions
[Specific recommendations based on findings]
```

## Operating Principles

- **Read-only analysis**: This skill does NOT modify implementation files. It only reads and reports.
- **Exception**: It MAY update checklists with pass/fail status and tasks.md with completion markers.
- **Constitution authority**: Constitution conflicts are always CRITICAL severity.
- **Deterministic**: Rerunning without changes should produce consistent results.
- **Actionable output**: Every finding must include a specific recommendation.
- **Zero issues is valid**: If everything passes, output a success report with coverage statistics.
