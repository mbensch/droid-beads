# Implementation Plan Methodology

Use this template when writing `docs/03-IMPLEMENTATION-PLAN.md`. Every implementation plan
must follow this structure and include all required sections.

---

## Document Structure

```
# <Project Name> -- Implementation Plan

## Overview
[1-2 paragraphs: goals, number of phases, estimated timeline]

---

## Methodology: Skills, Validation Contracts, and Verification

[Include the methodology section verbatim -- see below]

---

## Phase 1 -- <Name>
[Repeat for each phase]
```

---

## Methodology Section (include verbatim)

Every phase follows a three-stage workflow that ensures autonomous agents can complete and verify
work without constant human oversight.

### 1. Skill Building (before implementation begins)

Before a Worker can execute tasks in a phase, the Orchestrator ensures the necessary **agent
skills** exist. A skill is a reusable prompt/instruction set that encodes domain knowledge a
Worker needs to complete a specific type of work. Skills are built once and reused across tasks.

The Orchestrator creates skill-building tasks as beads issues. Once a skill is validated, it is
stored as `.factory/skills/<skill-name>/SKILL.md` (YAML frontmatter with `name` and `description`,
followed by the skill content). Workers reference skills by reading these files when claiming
implementation tasks.

Subagent identities (Orchestrator, Worker, Validator) are encoded as custom droids in
`.factory/droids/`. The Orchestrator runs as a dedicated interactive droid session (launched via
`start.sh`). Workers and Validators are spawned as stateless fire-and-forget subagents by the
Orchestrator using the Task tool.

### 2. Validation Contract (before implementation begins)

Each phase has a **validation contract** -- a structured document that defines exactly what must
be true for the phase to be considered complete. Contracts are written as machine-checkable
assertions wherever possible (compile checks, test commands, API call/response pairs) and
human-observable assertions where automated checking is not feasible.

Validation contracts are created as beads issues (type: `task`, label: `validation-contract`)
under the phase epic. They are written and reviewed before any implementation work starts.

A validation contract contains:
- **Preconditions:** What must be true before validation can run
- **Automated assertions:** Commands that return pass/fail
- **Behavioral assertions:** Step-by-step procedures with expected outcomes
- **Performance assertions:** Measurable thresholds
- **Regression guards:** Assertions that previous phases still pass

### 3. Validation Execution (after implementation)

When all tasks in a phase epic are marked complete, the Orchestrator triggers the **Validator**.
The Validator reads the contract, executes all assertions, and either:
- Creates new bug/task beads for each failure, sets the epic back to `in_progress`
- Closes the validation task if all assertions pass, allowing the Orchestrator to close the epic

The epic cannot be closed until the Validator signs off. This is the hard gate.

---

## Phase Template

Use this template for each phase:

```markdown
## Phase N -- <Name>

**Goal:** [One sentence describing what this phase produces and why it matters.]

### Tasks

- [ ] <Task title>
  - [Specific implementation detail]
  - [Another detail]

- [ ] <Task title>
  ...

### Acceptance Criteria

- [ ] [Verifiable criterion]
- [ ] [Verifiable criterion]

### Skills Required

Before implementation, build and validate these agent skills:

- [ ] **Skill: <Name>** -- [What a Worker needs to know. Specific APIs, crates, patterns,
  versions. Include references to official docs or known-good examples.]

### Validation Contract

**Preconditions:**
- [What must be true for validation to run]

**Automated Assertions:**
- [ ] [Exact command] [expected outcome]
- [ ] [Exact command] [expected outcome]

**Behavioral Assertions:**
- [ ] [Step-by-step procedure and expected visual/functional outcome]

**Performance Assertions:**
- [ ] [Metric, threshold, and measurement method]

**Regression Guards:**
- [ ] [Previous phase assertion that must still pass]

### Validation Steps

1. [Ordered list of what the Validator does]
2. ...
```

---

## Rules for Writing Phases

1. **Tasks must be atomic** -- a single Worker should be able to complete one task in one session.
2. **Acceptance criteria must be verifiable** -- no "should work" -- say exactly what output or
   behavior proves it works.
3. **Skills must be specific** -- name the exact crate/library version, API surface, and any
   known caveats. Generic "learn about X" is not a skill.
4. **Assertions must be executable** -- every automated assertion must be a shell command with
   an expected exit code or output pattern.
5. **Phases must build on each other** -- each phase's validation contract should include
   regression guards from all prior phases.
6. **First phase has no regression guards** -- only Phase 2+ include regression guards.
