# {{PROJECT_NAME}} -- Operational Plan

## Autonomous Agent Development Loop

This document defines how {{PROJECT_NAME}} is built using an autonomous multi-agent system
coordinated through beads for all task management, delegation, and work tracking.

---

## Identities

### Overseer (Human)

- Interacts with the Orchestrator at critical decision points or when the Orchestrator encounters
  an issue that cannot be resolved autonomously.
- Reviews and approves phase-level decisions.
- Does NOT need to approve individual task completions, epic transitions, or routine work.
- The Orchestrator should NOT stop and ask if it should move on to the next phase. Always move
  forward unless the Overseer explicitly intervenes.

### Orchestrator (Agent -- Opus 4.6)

- Main agent that orchestrates all work. Manages the inner development loop end-to-end.
- Runs as a **dedicated interactive droid session** (`.factory/droids/orchestrator.md`), launched
  via `start.sh`. The Overseer sees all output and can intervene at any time.
- Creates beads epics, tasks, skills, and validation contracts.
- Assigns work to Workers by creating and structuring beads issues.
- Spawns Workers and Validators as fire-and-forget subagents via the Task tool.
- Monitors progress by querying beads for stale, blocked, or completed work.
- Triggers Validators when an epic's implementation tasks are all complete.
- Handles failures by creating remediation tasks from Validator findings.
- Escalates to the Overseer using AskUser when genuinely stuck (see escalation criteria below).
- **Does not stop to ask permission to proceed to the next phase.** Phases flow automatically
  once the current phase's validation passes.

### Worker (Agent -- Kimi K2.5)

- Ephemeral agents spawned by the Orchestrator via the Task tool (`.factory/droids/worker.md`).
- Each Worker claims a single beads task (`bd update <id> --claim`), executes it, and closes it.
- Workers are stateless between tasks. All context they need is in the beads issue description,
  referenced skills (`.factory/skills/`), and the project codebase.
- Workers produce code changes, test results, or documentation as their output.
- Workers do NOT make architectural decisions. If a task is ambiguous, the Worker creates a
  `discovered-from` beads issue flagging the ambiguity and moves on or blocks.

### Validator (Agent -- GPT 5.3 Codex)

- Ephemeral agents spawned by the Orchestrator via the Task tool (`.factory/droids/validator.md`).
- Validates completed feature epics against the validation contract.
- Runs automated assertions (build checks, test commands, API call/response verification).
- Evaluates behavioral assertions (UI checks via automation or structured procedures).
- Measures performance assertions against defined thresholds.
- Runs regression guards to ensure previous phases are not broken.
- If validation fails: creates new bug/task beads under the epic for each failure, with clear
  reproduction steps and expected vs actual results.
- If validation passes: closes the validation contract task, signaling the Orchestrator that
  the epic can be closed.
- Every time all tasks in an epic are marked done, the Validator re-runs. Only if validation
  passes can the epic be closed. This is the hard gate.

---

## The Development Loop

### High-Level Flow

```
Overseer provides initial project spec and approves operational plan
         |
         v
Orchestrator reads spec, creates Phase 1 epic in beads
         |
         v
+-----------------------------------------------------+
|  PHASE LOOP (repeats for each phase)                |
|                                                     |
|  1. Orchestrator creates skill-building tasks       |
|  2. Workers claim and complete skill tasks          |
|  3. Orchestrator creates validation contract task   |
|  4. Worker writes the validation contract           |
|  5. Orchestrator creates implementation tasks       |
|  6. Workers claim and complete implementation tasks |
|  7. All tasks done -> Orchestrator triggers Validator|
|  8. Validator runs validation contract              |
|     - PASS -> Orchestrator closes epic, starts next |
|     - FAIL -> Validator creates fix tasks, go to 6  |
|                                                     |
+-----------------------------------------------------+
         |
         v
All phases complete -> Overseer notified of project completion
```

### Detailed Step-by-Step

#### Step 1: Phase Initialization

The Orchestrator creates the phase epic and all initial child issues.

```bash
# Create the phase epic
bd create "Phase 1: <Name>" -t epic -p 1 -d "<description>" --json
# Returns: {{BEADS_PREFIX}}-<epic-id>

# Create skill-building tasks under the epic
bd create "Skill: <domain>" -t task -p 1 \
  --parent {{BEADS_PREFIX}}-<epic-id> --label skill,phase-1 --json

# Create validation contract task
bd create "Validation Contract: Phase 1" -t task -p 1 \
  --parent {{BEADS_PREFIX}}-<epic-id> --label validation-contract,phase-1 --json
```

#### Step 2: Skill Building

Workers claim skill tasks. A skill task requires the Worker to:

1. Research the domain (read docs, explore APIs, test small examples)
2. Produce a skill document at `.factory/skills/<skill-name>/SKILL.md`
3. Validate the skill works by building a minimal proof-of-concept
4. Close the task with a reference to the produced skill artifact

```bash
bd update {{BEADS_PREFIX}}-<skill-id> --claim --json
# ... research and write skill ...
bd close {{BEADS_PREFIX}}-<skill-id> --reason "Skill doc created at .factory/skills/<name>/SKILL.md" --json
```

#### Step 3: Validation Contract Authoring

A Worker claims the validation contract task and writes the contract based on the phase's
acceptance criteria in `docs/03-IMPLEMENTATION-PLAN.md`.

```bash
bd update {{BEADS_PREFIX}}-<contract-id> --claim --json
# ... write contract ...
bd close {{BEADS_PREFIX}}-<contract-id> --reason "Contract written with N assertions" --json
```

#### Step 4: Implementation Task Creation

The Orchestrator creates implementation tasks from `docs/03-IMPLEMENTATION-PLAN.md`. Tasks must
be granular enough for a single Worker session.

```bash
bd create "<Task>" -t task -p 1 \
  --parent {{BEADS_PREFIX}}-<epic-id> --label implementation,phase-1 \
  -d "<description>" --json
```

#### Step 5: Worker Execution

```bash
bd ready --json
bd update {{BEADS_PREFIX}}-<task-id> --claim --json
# ... implement ...
# Run: {{BUILD_COMMAND}}
# Run: {{LINT_COMMAND}}
# Run: {{TEST_COMMAND}}
bd close {{BEADS_PREFIX}}-<task-id> --reason "<summary>" --json
```

If a Worker discovers additional work:
```bash
bd create "Found: <issue>" -t bug -p 1 \
  --deps discovered-from:{{BEADS_PREFIX}}-<task-id> --parent {{BEADS_PREFIX}}-<epic-id> --json
```

#### Step 6: Validation Trigger

```bash
bd list --parent {{BEADS_PREFIX}}-<epic-id> --status open --json
# If empty, create validation run:
bd create "Run Validation: Phase N" -t task -p 0 \
  --parent {{BEADS_PREFIX}}-<epic-id> --label validation-run,phase-N --json
```

#### Step 7: Validator Execution

```bash
bd update {{BEADS_PREFIX}}-<validation-run-id> --claim --json
# ... run assertions ...

# On failure:
bd create "FAIL: <assertion>" -t bug -p 1 \
  --parent {{BEADS_PREFIX}}-<epic-id> --label validation-failure,phase-N \
  -d "Expected: ... Actual: ... Steps: ..." --json

# On all pass:
bd close {{BEADS_PREFIX}}-<validation-run-id> --reason "All N assertions passed" --json
```

#### Step 8: Remediation or Completion

If failures: workers fix, re-trigger validation.
If passed: close the epic.

```bash
bd close {{BEADS_PREFIX}}-<epic-id> --reason "Phase N complete. All assertions passed." --json
```

Then immediately create the next phase epic.

---

## Beads Structure

### Label Taxonomy

| Label | Meaning |
|-------|---------|
| `phase-N` | Which phase the issue belongs to |
| `skill` | Skill-building task |
| `validation-contract` | Validation contract authoring task |
| `validation-run` | Active validation execution |
| `validation-failure` | Bug/task created by Validator for a failed assertion |
| `implementation` | Core implementation task |
| `regression` | Regression bug found during validation of a later phase |
| `blocked` | Task is blocked on another issue |

### Issue Hierarchy per Phase

```
Epic: "Phase N: <Name>"
  ├── Task: "Skill: <domain>"            [skill, phase-N]
  ├── Task: "Validation Contract: Phase N" [validation-contract, phase-N]
  ├── Task: "Implement <A>"              [implementation, phase-N]
  ├── Task: "Implement <B>"              [implementation, phase-N]
  ├── Task: "Run Validation: Phase N"    [validation-run, phase-N]
  ├── Bug:  "FAIL: <assertion>"          [validation-failure, phase-N]
  └── Task: "Run Validation: Phase N (attempt 2)" [validation-run, phase-N]
```

### Dependency Patterns

```bash
# Skills block implementation tasks
bd dep add {{BEADS_PREFIX}}-<impl-task> {{BEADS_PREFIX}}-<skill-task> --type blocks

# Validation contract blocks validation runs
bd dep add {{BEADS_PREFIX}}-<validation-run> {{BEADS_PREFIX}}-<contract> --type blocks

# All impl tasks block the validation run
bd dep add {{BEADS_PREFIX}}-<validation-run> {{BEADS_PREFIX}}-<impl-task> --type blocks

# Previous phase blocks current phase
bd dep add {{BEADS_PREFIX}}-<phase-2-epic> {{BEADS_PREFIX}}-<phase-1-epic> --type blocks
```

---

## Orchestrator Decision Rules

### When to Escalate to Overseer

Escalate ONLY when:

1. **Ambiguous requirements** -- The spec doesn't cover a decision and the Orchestrator cannot
   make a reasonable judgment call.
2. **Persistent validation failures** -- The same assertion has failed 3+ times after fixes.
3. **External dependency failure** -- A library doesn't work as documented, or a tool is unavailable.
4. **Conflicting constraints** -- Two requirements contradict each other irresolvably.

Do NOT escalate for: routine implementation decisions, discovered bugs, phase transitions,
performance tuning within defined thresholds.

### When to Move to the Next Phase

Immediately after the current phase epic is closed (validation passed). No pause, no confirmation.

Exception: the Overseer can halt the pipeline by setting the next phase epic to `deferred`.

---

## Worker Execution Protocol

### Claiming Work

```bash
bd ready --json
bd update {{BEADS_PREFIX}}-<id> --claim --json
```

### Executing Work

1. Read task description
2. Read referenced skills from `.factory/skills/`
3. Explore relevant codebase files
4. Implement the change
5. Run checks: `{{BUILD_COMMAND}}` · `{{LINT_COMMAND}}` · `{{TEST_COMMAND}}`
6. Close with a clear summary

---

## Validator Execution Protocol

1. Claim the `validation-run` task
2. Read the validation contract
3. Execute: preconditions → automated assertions → behavioral assertions → performance assertions → regression guards
4. For each failure: create a `bug` issue labeled `validation-failure`
5. Close the validation-run with a pass/fail summary

On re-validation: always run the FULL contract, not just previously failed assertions.

---

## Concurrency Model

- **Parallel Workers**: Multiple Workers can operate simultaneously on non-conflicting tasks.
  Beads' atomic `--claim` prevents double-claiming.
- **Sequential Phases**: Phase N+1 cannot start until Phase N's validation passes.
- **Parallel Skills**: All skill tasks within a phase can be built in parallel.

---

## Failure Modes and Recovery

| Failure | Detection | Recovery |
|---------|-----------|----------|
| Worker crashes mid-task | `bd stale --days 1 --status in_progress` | Orchestrator unclaims; another Worker picks it up |
| Validation fails 3+ times | Orchestrator counts `validation-run` tasks | Escalate to Overseer |
| Build breaks after a change | Build command exits non-zero | Worker creates `bug` with `discovered-from` link |
| Stale issues accumulate | `bd stale --days 7` | Orchestrator reviews and closes or re-assigns |

---

## Monitoring

```bash
bd list --label implementation --status open --json | jq length    # remaining tasks
bd list --parent {{BEADS_PREFIX}}-<current-epic> --json            # current phase
bd stale --days 2 --json                                            # stuck work
bd list --label validation-failure --status open --json | jq length # open failures
bd list --status blocked --json                                     # blocked work
```

---

## Session Boundaries

### Orchestrator Sessions

Launched via `./start.sh` as a dedicated droid session. On start, runs the `init` skill to
assess beads state and resume the loop. The Overseer sees all output in real time. Escalates
to the Overseer via AskUser when genuinely stuck.

### Worker Sessions

Short-lived and stateless: claim → execute → check ({{BUILD_COMMAND}}) → close.

### Validator Sessions

Triggered by the Orchestrator: claim validation-run → run full contract → create failure issues
→ close validation-run.

---

## Quick Reference

| Action | Command |
|--------|---------|
| Create phase epic | `bd create "Phase N: Name" -t epic -p 1 --json` |
| Create child task | `bd create "Task" -t task -p 1 --parent <id> --label impl,phase-N --json` |
| Find work | `bd ready --json` |
| Claim task | `bd update <id> --claim --json` |
| Close task | `bd close <id> --reason "Done" --json` |
| Check epic | `bd list --parent <id> --json` |
| Open impl tasks | `bd list --parent <id> --status open --label implementation --json` |
| Discovered bug | `bd create "Bug" -t bug -p 1 --deps discovered-from:<id> --parent <epic> --json` |
| Add dependency | `bd dep add <blocked> <blocker> --type blocks` |
| Stale check | `bd stale --days 2 --json` |
