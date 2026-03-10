---
name: {{PROJECT_SLUG}}-init
description: >-
  {{PROJECT_NAME}} Orchestrator initiation sequence. Assesses current project state via beads
  and either bootstraps Phase 1 or resumes from the current position in the development loop.
---

# {{PROJECT_NAME}} Init

Run this skill at the start of every Orchestrator session to determine project state and enter
the development loop at the correct point.

## Step 1: Assess current state

```bash
# List all epics to find the current phase
bd list -t epic --json

# Check for any open work across the project
bd list --status open --json | head -40

# Check for stale work (unclaimed > 2 days)
bd stale --days 2 --json
```

## Step 2: Determine entry point

### Case A: No epics exist (fresh project)

Bootstrap Phase 1. Create the epic and all initial child issues based on
`docs/03-IMPLEMENTATION-PLAN.md` Phase 1:

```bash
# Create Phase 1 epic
EPIC=$(bd q "Phase 1: <name from implementation plan>" -t epic -p 1 -l phase-1)

# Create skill tasks under the epic (one per skill listed in Phase 1 "Skills Required")
bd create "Skill: <skill name>" -t task -p 1 --parent $EPIC -l skill,phase-1 --json

# Create validation contract task
bd create "Validation Contract: Phase 1" -t task -p 1 \
  --parent $EPIC -l validation-contract,phase-1 --json

# Create implementation tasks and wire dependencies:
# - skill tasks should block implementation tasks that use them
# - validation contract should block validation-run
```

Read `docs/03-IMPLEMENTATION-PLAN.md` for the full task list, skill list, and validation contract
for Phase 1. Enter the loop at **Step 1: Skill Building**.

### Case B: Epics exist -- find the active phase

```bash
# Find open epic
bd list -t epic --status open --json
```

Then determine where in the loop the phase is:

```bash
# Are skill tasks still open?
bd list --parent <epic-id> --label skill --status open --json

# Is the validation contract open?
bd list --parent <epic-id> --label validation-contract --status open --json

# Are implementation tasks open?
bd list --parent <epic-id> --label implementation --status open --json

# Is there an active validation-run?
bd list --parent <epic-id> --label validation-run --status open --json

# Are there open validation failures to fix?
bd list --parent <epic-id> --label validation-failure --status open --json
```

Resume at the earliest incomplete step:
- Skill tasks open → resume at **skill building**
- Contract open → resume at **validation contract**
- Implementation tasks open → resume at **implementation**
- Validation-run open → resume at **validation** (re-trigger Validator)
- Validation failures open → resume at **remediation** (Workers fix failures, then re-trigger)
- All closed → close the epic and start next phase

### Case C: All epics are closed

All phases are complete. Notify the Overseer that the project is done.

## Step 3: Enter the loop

Once you know your entry point, proceed with the development loop as defined in
`docs/05-OPERATIONAL-PLAN.md`. Spawn Workers and Validators as subagents via the Task tool
using the `worker` and `validator` droids respectively.

Read `docs/03-IMPLEMENTATION-PLAN.md` for the complete task list, skill list, and validation
contract contents for each phase.
