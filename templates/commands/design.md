---
description: "End-to-end design skill: take a clarified specification and produce a complete technical plan with research, data model, contracts, and quality analysis. Orchestrates plan, checklist, and analyze in one pass."
handoffs:
  - label: Build the Project
    agent: speckit.build
    prompt: Build the project from this plan
    send: true
  - label: Generate Tasks (atomic)
    agent: speckit.tasks
    prompt: Break the plan into tasks
    send: true
scripts:
  sh: scripts/bash/setup-plan.sh --json
  ps: scripts/powershell/setup-plan.ps1 -Json
agent_scripts:
  sh: scripts/bash/update-agent-context.sh __AGENT__
  ps: scripts/powershell/update-agent-context.ps1 -AgentType __AGENT__
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty). The user input typically includes the tech stack and architectural preferences (e.g., "I am building with Python and FastAPI").

## Purpose

This is a **deployment skill** that orchestrates multiple Speckit commands into a single design workflow. Instead of running `/speckit.plan`, `/speckit.checklist`, and `/speckit.analyze` separately, this skill runs them as one cohesive pass.

**Workflow position**: This is step 2 of 4 in the full Speckit skill pipeline:

```
/speckit.brainstorm → /speckit.design → /speckit.build → /speckit.validate
```

## Outline

### Phase 1: Setup & Context Loading

1. **Run** `{SCRIPT}` from repo root and parse JSON for FEATURE_SPEC, IMPL_PLAN, SPECS_DIR, BRANCH. All paths must be absolute.
   - For single quotes in args like "I'm Groot", use escape syntax: `"I'm Groot"`

2. **Load context**:
   - Read FEATURE_SPEC (the specification from brainstorming)
   - Read `/memory/constitution.md` (if exists)
   - Load IMPL_PLAN template (already copied by setup script)

3. **Verify prerequisites**: If spec.md is missing or empty, abort with:
   > "No specification found. Run `/speckit.brainstorm` or `/speckit.specify` first."

### Phase 2: Implementation Plan

Follow the plan template structure to generate a complete technical plan:

1. **Technical Context**:
   - Extract tech stack from user input and spec context
   - Mark unknowns as "NEEDS CLARIFICATION"
   - Fill Constitution Check section from constitution (if exists)
   - Evaluate constitutional gates (ERROR if violations are unjustified)

2. **Phase 0 - Research** (`research.md`):
   - For each NEEDS CLARIFICATION item, research and resolve
   - For each dependency, identify best practices
   - For each integration, identify patterns
   - Consolidate findings with: Decision, Rationale, Alternatives Considered
   - **Output**: `research.md` with all unknowns resolved

3. **Phase 1 - Design** (data model, contracts, quickstart):

   a. **Data Model** (`data-model.md`):
      - Extract entities from feature spec
      - Define fields, relationships, validation rules
      - Document state transitions if applicable

   b. **API Contracts** (`contracts/`):
      - Map each user action to an endpoint
      - Use standard REST/GraphQL patterns
      - Output OpenAPI/GraphQL schema

   c. **Quickstart** (`quickstart.md`):
      - Key validation scenarios
      - Integration test patterns

   d. **Agent Context Update**:
      - Run `{AGENT_SCRIPT}` to update agent-specific context
      - Add only new technology from current plan

4. **Re-evaluate** Constitution Check post-design. Document any gate adjustments.

5. **Write** the completed plan to IMPL_PLAN path.

### Phase 3: Quality Checklist

Generate a domain-specific quality checklist for the plan. This checklist tests the **requirements quality**, not the implementation.

1. **Analyze** the spec and plan to identify the most relevant quality dimensions:
   - If the feature involves UI → generate UX requirements quality items
   - If APIs are defined → generate API requirements quality items
   - If security-sensitive → generate security requirements quality items
   - If performance-critical → generate performance requirements quality items

2. **Create** the checklist at `FEATURE_DIR/checklists/[domain].md` following the "unit tests for English" principle:

   Each item must ask about the **quality of the requirements**, not whether the implementation works:
   - "Are [requirement type] defined/specified for [scenario]?"
   - "Is [vague term] quantified with specific criteria?"
   - "Are requirements consistent between [section A] and [section B]?"
   - "Can [requirement] be objectively measured?"

   Number items sequentially (CHK001, CHK002, etc.). Include traceability references (`[Spec FR-X]`, `[Gap]`, `[Ambiguity]`).

3. **Self-evaluate**: Run through the checklist against the spec/plan. Mark items that pass, flag items that fail. Fix any spec/plan issues found (max 2 iterations).

### Phase 4: Consistency Analysis

Perform a lightweight cross-artifact analysis (subset of `/speckit.analyze`):

1. **Build** a requirements inventory from the spec (functional + non-functional requirements with stable keys).

2. **Check** for:
   - **Coverage gaps**: Requirements with no plan-level design
   - **Terminology drift**: Same concept named differently in spec vs plan
   - **Constitution violations**: Any MUST principle not reflected in plan
   - **Ambiguity**: Vague adjectives lacking measurable criteria
   - **Inconsistency**: Conflicting statements between spec and plan

3. **Classify** findings by severity:
   - **CRITICAL**: Constitution violations, missing core requirements coverage
   - **HIGH**: Conflicting requirements, ambiguous security/performance attributes
   - **MEDIUM**: Terminology drift, missing non-functional coverage
   - **LOW**: Style/wording improvements

4. **If CRITICAL issues found**: Fix them immediately. Update spec/plan as needed.

5. **If only MEDIUM/LOW issues found**: Note them in the report but proceed.

### Phase 5: Report

Output a completion summary:

```
## Design Complete

- **Branch**: [branch name]
- **Plan**: [absolute path to plan.md]
- **Research**: [absolute path to research.md]
- **Data Model**: [absolute path to data-model.md]
- **Contracts**: [absolute path(s) to contract files]
- **Quickstart**: [absolute path to quickstart.md]
- **Quality Checklist**: [absolute path to domain checklist]
- **Agent Context**: [updated/skipped]

### Consistency Summary
- Requirements: [N total]
- Plan coverage: [X of N requirements have plan-level design]
- Issues found: [CRITICAL: X, HIGH: X, MEDIUM: X, LOW: X]
- Constitution alignment: [PASS/issues noted]

### Next Step
Run `/speckit.build` to generate tasks and implement, or `/speckit.tasks` for the atomic task generation command.
```

## Key Rules

- Use absolute paths throughout
- ERROR on unresolved constitutional gate failures
- The plan must be concrete enough that task generation can proceed without ambiguity
- Research decisions must have rationale documented
- Design artifacts must trace back to spec requirements
