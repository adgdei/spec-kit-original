---
description: "End-to-end brainstorming skill: take a raw feature idea and produce a complete, clarified specification with quality validation. Orchestrates constitution, specify, and clarify in one pass."
handoffs:
  - label: Design Technical Plan
    agent: speckit.design
    prompt: Design a technical plan for this specification. I am building with...
  - label: Build Technical Plan (atomic)
    agent: speckit.plan
    prompt: Create a plan for the spec. I am building with...
scripts:
  sh: scripts/bash/create-new-feature.sh --json "{ARGS}"
  ps: scripts/powershell/create-new-feature.ps1 -Json "{ARGS}"
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Purpose

This is a **deployment skill** that orchestrates multiple Speckit commands into a single brainstorming workflow. Instead of running `/speckit.constitution`, `/speckit.specify`, and `/speckit.clarify` separately, this skill runs them as one cohesive pass.

**Workflow position**: This is step 1 of 4 in the full Speckit skill pipeline:

```
/speckit.brainstorm → /speckit.design → /speckit.build → /speckit.validate
```

## Outline

### Phase 1: Constitution Check

1. Check if `/memory/constitution.md` exists and contains concrete principles (not just placeholder tokens like `[PRINCIPLE_1_NAME]`).

2. **If constitution exists and is populated**: Read it, note the governing principles. Proceed to Phase 2.

3. **If constitution is missing or only has placeholders**: Ask the user ONE question:

   > "No project constitution found. Want me to create one based on your project context? (yes/skip)"

   - If **yes**: Derive 3-5 principles from the feature description, any README, and repository context. Write them to `/memory/constitution.md` using concise, declarative MUST/SHOULD statements. Keep it lean - this can be refined later via `/speckit.constitution`.
   - If **skip**: Proceed without a constitution. Note that downstream plan gates will have nothing to check against.

### Phase 2: Feature Specification

1. **Generate a concise short name** (2-4 words) for the feature branch:
   - Analyze the feature description and extract the most meaningful keywords
   - Use action-noun format when possible (e.g., "add-user-auth", "analytics-dashboard")
   - Preserve technical terms and acronyms

2. **Check for existing branches before creating new one**:

   a. Fetch all remote branches:

      ```bash
      git fetch --all --prune
      ```

   b. Find the highest feature number across all sources for the short-name:
      - Remote branches: `git ls-remote --heads origin | grep -E 'refs/heads/[0-9]+-<short-name>$'`
      - Local branches: `git branch | grep -E '^[* ]*[0-9]+-<short-name>$'`
      - Specs directories: Check for directories matching `specs/[0-9]+-<short-name>`

   c. Determine the next available number (N+1), starting from 1 if none found.

   d. Run `{SCRIPT}` with the calculated number and short-name:
      - Bash: `{SCRIPT} --json --number N+1 --short-name "your-short-name" "Feature description"`
      - PowerShell: `{SCRIPT} -Json -Number N+1 -ShortName "your-short-name" "Feature description"`
      - Parse JSON output for BRANCH_NAME and SPEC_FILE paths
      - For single quotes in args like "I'm Groot", use escape syntax: `"I'm Groot"`

   **IMPORTANT**: Only run this script once per feature.

3. **Load** `templates/spec-template.md` to understand required sections.

4. **Generate the specification**:
   - Parse the user's feature description
   - Extract key concepts: actors, actions, data, constraints
   - For unclear aspects, make informed guesses based on context and industry standards
   - Only mark with `[NEEDS CLARIFICATION: specific question]` if the choice significantly impacts scope or UX AND no reasonable default exists
   - **LIMIT: Maximum 3 `[NEEDS CLARIFICATION]` markers**
   - Fill all mandatory template sections: User Scenarios, Functional Requirements, Success Criteria, Key Entities
   - Focus on WHAT and WHY, never HOW (no tech stack, APIs, code structure)

5. **Write** the specification to SPEC_FILE.

### Phase 3: Inline Clarification

Instead of deferring to a separate `/speckit.clarify` pass, resolve ambiguities immediately.

1. **Scan** the spec for `[NEEDS CLARIFICATION]` markers and perform a quick ambiguity check across these categories:
   - Functional scope & behavior
   - Domain & data model
   - Interaction & UX flow
   - Non-functional quality attributes
   - Edge cases & failure handling

2. **If no meaningful ambiguities found**: Skip to Phase 4. Report: "No critical ambiguities - spec is ready for planning."

3. **If ambiguities exist** (max 5 questions total):

   Present questions ONE AT A TIME. For each question:

   - Provide a **recommended answer** with reasoning
   - Present options as a compact table:

     | Option | Description |
     |--------|-------------|
     | A | First option |
     | B | Second option |
     | Short | Provide a different short answer (<=5 words) |

   - Accept "yes" or "recommended" to use your suggestion
   - After each answer, immediately integrate it into the spec:
     - Record in `## Clarifications` → `### Session YYYY-MM-DD` section
     - Update the relevant spec section (requirements, edge cases, etc.)
     - Remove the `[NEEDS CLARIFICATION]` marker
     - Save the spec file after each integration

   - Stop asking when: all resolved, user says "done"/"skip", or 5 questions reached.

### Phase 4: Quality Validation

1. **Create** a quality checklist at `FEATURE_DIR/checklists/requirements.md`:

   ```markdown
   # Specification Quality Checklist: [FEATURE NAME]

   **Purpose**: Validate specification completeness and quality
   **Created**: [DATE]

   ## Content Quality

   - [ ] No implementation details (languages, frameworks, APIs)
   - [ ] Focused on user value and business needs
   - [ ] Written for non-technical stakeholders
   - [ ] All mandatory sections completed

   ## Requirement Completeness

   - [ ] No [NEEDS CLARIFICATION] markers remain
   - [ ] Requirements are testable and unambiguous
   - [ ] Success criteria are measurable
   - [ ] Success criteria are technology-agnostic
   - [ ] All acceptance scenarios defined
   - [ ] Edge cases identified
   - [ ] Scope clearly bounded
   - [ ] Dependencies and assumptions identified

   ## Feature Readiness

   - [ ] All functional requirements have clear acceptance criteria
   - [ ] User scenarios cover primary flows
   - [ ] Feature meets measurable outcomes in Success Criteria
   - [ ] No implementation details leak into specification
   ```

2. **Validate** the spec against each checklist item. Fix any failures (max 3 iterations). Update the checklist with pass/fail status.

### Phase 5: Report

Output a completion summary:

```
## Brainstorm Complete

- **Branch**: [branch name]
- **Spec**: [absolute path to spec.md]
- **Checklist**: [absolute path to requirements.md]
- **Constitution**: [exists/created/skipped]
- **Clarifications resolved**: [N of M]
- **Quality validation**: [PASS/items remaining]

### Next Step
Run `/speckit.design` to create a technical plan, or `/speckit.plan` for the atomic plan command.
```

## Guidelines

- Focus on **WHAT** users need and **WHY** - never HOW
- Written for business stakeholders, not developers
- Make informed guesses rather than asking excessive questions
- Keep the process conversational but efficient
- The goal is a spec that's ready for technical planning in one session
