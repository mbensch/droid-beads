# Beads CLI Reference

Beads (`bd`) is a lightweight issue tracker with first-class dependency support. State lives in a local Dolt database. All operations are scriptable and JSON-serializable.

## Setup

```bash
# Initialize in current directory
bd init -p <prefix>          # e.g., bd init -p myapp

# bd init creates:
#   .beads/         database directory
#   AGENTS.md       agent instructions (auto-generated)
```

## Core Issue Operations

### Create

```bash
bd create "Title"                          # create task (default)
bd create "Title" -t epic                  # type: bug|feature|task|epic|chore|decision
bd create "Title" -p 0                     # priority 0 (highest) through 4 (lowest)
bd create "Title" --parent bd-abc          # child issue
bd create "Title" --deps bd-xyz            # depends on bd-xyz
bd create "Title" --deps "blocks:bd-abc,discovered-from:bd-xyz"
bd create "Title" -d "Description text"    # inline description
bd create "Title" --body-file desc.md      # description from file
bd create "Title" -l phase-1,skill:auth    # labels (comma-separated)
bd create "Title" -a alice                 # assignee

# Quick capture (outputs only ID, for scripting)
EPIC_ID=$(bd q "Epic Title" -t epic)
TASK_ID=$(bd q "Task Title" --parent $EPIC_ID)
```

### Read

```bash
bd show bd-abc                # show issue details
bd show bd-abc --long         # show all fields including metadata
bd show bd-abc --children     # show child issues only
bd show bd-abc --refs         # show issues that reference this one
bd list                       # list open issues (default limit 50)
bd list -t epic               # filter by type
bd list -s in_progress        # filter by status: open|in_progress|blocked|deferred|closed
bd list --parent bd-abc       # list children of epic
bd list --label phase-1       # filter by label
bd list --all                 # include closed issues
bd list --pretty              # tree format with status/priority symbols
bd list --json                # machine-readable output
```

### Update

```bash
bd update bd-abc --status in_progress
bd update bd-abc --claim                   # atomic claim: sets assignee + in_progress
bd update bd-abc --add-label skill:auth
bd update bd-abc --remove-label phase-1
bd update bd-abc --priority 1
bd update bd-abc --assignee alice
bd update bd-abc -d "New description"
bd update bd-abc --title "New title"
```

### Close

```bash
bd close bd-abc
bd close bd-abc --reason "Completed per acceptance criteria"
bd close bd-abc bd-xyz bd-123              # close multiple at once
```

### Search / Query

```bash
# Full query language
bd query "status=open AND priority<=2"
bd query "type=epic AND label=phase-1"
bd query "(status=open OR status=in_progress) AND assignee=alice"
bd query "type=bug AND label=regression"
bd query "status!=closed AND priority=0"
bd query "created>7d AND status=open"      # created within last 7 days

# Text search
bd search "authentication"

# Count
bd count --type epic --status open
```

## Dependencies

```bash
# bd-xyz blocks bd-abc (bd-abc cannot start until bd-xyz is closed)
bd dep bd-xyz --blocks bd-abc
bd dep add bd-abc bd-xyz                   # equivalent

# List dependencies
bd dep list bd-abc                         # what does bd-abc depend on?
bd dep list bd-abc --dependents            # what depends on bd-abc?
bd dep tree bd-abc                         # full dependency tree

# Remove
bd dep remove bd-abc bd-xyz

# Relate (non-blocking bidirectional link)
bd dep relate bd-abc bd-xyz
```

## Labels

Labels are free-form strings. Conventions used in this system:

```
phase-<n>           phase ownership (phase-1, phase-2, ...)
skill:<name>        skill required (skill:auth, skill:db-schema, ...)
role:<name>         role assignment (role:worker, role:validator, role:orchestrator)
validation-run      marks a validation execution task
discovered-by-agent marks issues created autonomously by agents
regression          marks regression bugs (use priority 0)
```

```bash
bd label add bd-abc phase-1 skill:auth    # add multiple labels
bd label remove bd-abc phase-1
bd label list bd-abc                      # show labels on issue
bd label list-all                         # show all labels in database
```

## Epic Management

```bash
bd epic status                             # show all epics with completion %
bd epic status --label phase-1            # filter to labeled epics
bd epic close-eligible                    # auto-close epics where all children done
```

## State Machine (for agents)

```bash
# Claim a task atomically (fails if already claimed by another agent)
bd update bd-abc --claim

# Transition states
bd update bd-abc --status in_progress
bd update bd-abc --status blocked
bd update bd-abc --status open            # reopen / unblock
bd close bd-abc --reason "done"

# Operational state (multi-dimensional, uses labels + events)
bd set-state bd-abc mode=degraded --reason "High error rate"
bd set-state bd-abc health=healthy
bd state bd-abc mode                      # query current value of dimension
```

## Issue Types

| Type       | Purpose                                    |
|------------|--------------------------------------------|
| `epic`     | Container for a phase or milestone         |
| `task`     | Unit of work to be implemented             |
| `bug`      | Defect requiring a fix                     |
| `feature`  | New capability                             |
| `chore`    | Maintenance work (no user-visible change)  |
| `decision` | Architecture decision record (ADR)         |

## Common Agent Patterns

### Create an epic with child tasks

```bash
EPIC=$(bd q "Phase 1: Foundation" -t epic -l phase-1)
bd create "Implement auth module" --parent $EPIC -l phase-1,skill:auth
bd create "Set up database schema" --parent $EPIC -l phase-1,skill:db
bd create "Write unit tests" --parent $EPIC -l phase-1 --deps $TASK1
```

### Claim and work a task

```bash
# Find ready work
bd list --ready --label role:worker --sort priority

# Claim atomically
bd update bd-abc --claim

# Close when done
bd close bd-abc --reason "Implemented per acceptance criteria"
```

### Create a validation run

```bash
VAL=$(bd create "Validate Phase 1" \
  -t task \
  -l validation-run,phase-1 \
  --deps $IMPL_TASK_ID \
  -d "Execute full validation contract for Phase 1")

# If validation fails, create a bug
bd create "Auth token expiry not enforced" \
  -t bug \
  -p 0 \
  -l regression,phase-1 \
  --deps "discovered-from:$VAL_ID" \
  -d "Precondition failed: token expiry check missing"
```

### Check epic completion

```bash
# Check if all children of an epic are closed
bd epic status | grep "phase-1"

# List open tasks under an epic
bd list --parent $EPIC_ID --status open
```

## Filtering Ready Work

```bash
# Issues ready to be claimed (open, unblocked, not deferred)
bd list --ready

# Ready tasks for a specific phase and role
bd list --ready --label phase-1 --label role:worker --sort priority

# Blocked issues (have open dependencies)
bd blocked
```

## JSON Output (for scripting)

```bash
bd list --json | jq '.[] | select(.status == "open") | .id'
bd show bd-abc --json | jq '.labels'
bd query "type=epic AND status=open" --json | jq '.[0].id'
```

## Global Flags

All commands accept:

```
--json          Output in JSON format
--quiet (-q)    Suppress non-essential output
--prefix bd-    Specify rig by prefix (if multiple rigs exist)
--db PATH       Explicit path to database file
--sandbox       Disable auto-sync (read-only safe mode)
--readonly      Block all write operations
--actor NAME    Override actor name for audit trail
```
