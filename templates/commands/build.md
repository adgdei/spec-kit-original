---
description: "End-to-end build skill: take a technical plan and produce working code. Generates the task breakdown and executes all tasks in one pass. Orchestrates tasks and implement together."
handoffs:
  - label: Validate Implementation
    agent: speckit.validate
    prompt: Validate the implementation against the specification
    send: true
  - label: Analyze Consistency (atomic)
    agent: speckit.analyze
    prompt: Run a project analysis for consistency
    send: true
scripts:
  sh: scripts/bash/check-prerequisites.sh --json
  ps: scripts/powershell/check-prerequisites.ps1 -Json
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Purpose

This is a **deployment skill** that orchestrates multiple Speckit commands into a single build workflow. Instead of running `/speckit.tasks` and `/speckit.implement` separately, this skill generates the task breakdown and executes it in one cohesive pass.

**Workflow position**: This is step 3 of 4 in the full Speckit skill pipeline:

```
/speckit.brainstorm → /speckit.design → /speckit.build → /speckit.validate
```

## Outline

### Phase 1: Setup & Context Loading

1. **Run** `{SCRIPT}` from repo root and parse FEATURE_DIR and AVAILABLE_DOCS list. All paths must be absolute.
   - For single quotes in args like "I'm Groot", use escape syntax: `"I'm Groot"`

2. **Load design documents** from FEATURE_DIR:
   - **Required**: plan.md (tech stack, libraries, structure)
   - **Required**: spec.md (user stories with priorities)
   - **Optional**: data-model.md (entities), contracts/ (API endpoints), research.md (decisions), quickstart.md (test scenarios)

3. **Verify prerequisites**: If plan.md is missing, abort with:
   > "No implementation plan found. Run `/speckit.design` or `/speckit.plan` first."

### Phase 2: Task Generation

Generate a complete, executable task breakdown:

1. **Extract** from loaded documents:
   - Tech stack, libraries, project structure (from plan.md)
   - User stories with priorities P1, P2, P3... (from spec.md)
   - Entities and relationships (from data-model.md, if exists)
   - Endpoints mapped to user stories (from contracts/, if exists)
   - Technical decisions and constraints (from research.md, if exists)

2. **Generate tasks** organized by user story, following the strict checklist format:

   ```text
   - [ ] [TaskID] [P?] [Story?] Description with file path
   ```

   Format components:
   - **Checkbox**: Always `- [ ]`
   - **Task ID**: Sequential (T001, T002, T003...)
   - **[P] marker**: Only if parallelizable (different files, no dependencies)
   - **[Story] label**: [US1], [US2], etc. for story phase tasks only
   - **Description**: Clear action with exact file path

3. **Organize** into phases:
   - **Phase 1**: Setup (project initialization)
   - **Phase 2**: Foundational (blocking prerequisites for all user stories)
   - **Phase 3+**: One phase per user story in priority order
   - **Final Phase**: Polish & cross-cutting concerns

4. **Generate** dependency graph and parallel execution opportunities.

5. **Write** tasks.md to FEATURE_DIR using `templates/tasks-template.md` as structure.

### Phase 3: Pre-Implementation Checks

Before executing code, verify readiness:

1. **Check checklists** (if FEATURE_DIR/checklists/ exists):
   - Scan all checklist files
   - Count total, completed, and incomplete items
   - Display status table:

     ```text
     | Checklist | Total | Completed | Incomplete | Status |
     |-----------|-------|-----------|------------|--------|
     | ux.md     | 12    | 12        | 0          | PASS   |
     | test.md   | 8     | 5         | 3          | FAIL   |
     ```

   - **If any checklist incomplete**: Ask user to proceed or stop
   - **If all pass**: Automatically proceed

2. **Project setup verification**:
   - Create/verify .gitignore based on tech stack
   - Create/verify .dockerignore if Docker detected
   - Create/verify other ignore files as appropriate for detected technologies
   - Append missing critical patterns to existing ignore files

### Phase 4: Implementation Execution

Execute all tasks from the generated breakdown:

1. **Parse** tasks.md structure:
   - Task phases, dependencies, parallel markers
   - Task details: ID, description, file paths
   - Execution flow and ordering

2. **Execute** phase-by-phase:
   - Complete each phase before moving to the next
   - Respect dependencies: sequential tasks in order, parallel tasks [P] together
   - Follow TDD approach if tests are defined: test tasks before implementation
   - File-based coordination: tasks affecting the same files run sequentially

3. **Implementation order**:
   - Setup first: project structure, dependencies, configuration
   - Tests before code (if applicable): contracts, entities, integration scenarios
   - Core development: models, services, CLI commands, endpoints
   - Integration: database connections, middleware, logging, external services
   - Polish: unit tests, performance optimization, documentation

4. **Progress tracking**:
   - Report progress after each completed task
   - Mark completed tasks as `[X]` in tasks.md
   - Halt on non-parallel task failures
   - For parallel tasks [P]: continue with successful, report failed
   - Provide clear error messages with debugging context

### Phase 5: Post-Implementation Check

After all tasks are executed:

1. **Verify** all required tasks are completed (all `[X]` in tasks.md)
2. **Run** available tests and report results
3. **Check** that implemented features match the original specification
4. **Confirm** implementation follows the technical plan

### Phase 6: Report

Output a completion summary:

```
## Build Complete

- **Branch**: [branch name]
- **Tasks**: [absolute path to tasks.md]
- **Total tasks**: [N]
- **Completed**: [X of N]
- **Failed**: [Y tasks, if any]

### Implementation Summary
- Phase 1 (Setup): [status]
- Phase 2 (Foundational): [status]
- Phase 3+ (User Stories): [status per story]
- Final (Polish): [status]

### Test Results
- [test summary if tests were run]

### Next Step
Run `/speckit.validate` to verify the implementation, or `/speckit.analyze` for atomic consistency analysis.
```

## Key Rules

- Use absolute paths throughout
- Tasks must be specific enough that each can be completed without additional context
- Each user story phase should be independently testable
- Respect task dependencies strictly - never execute a task before its prerequisites
- Mark tasks complete in tasks.md as they finish
- If tasks.md already exists from a previous `/speckit.tasks` run, ask the user whether to regenerate or reuse it
