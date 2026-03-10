# {{PROJECT_NAME}}

See `docs/01-PROJECT-OVERVIEW.md` for the full technical framing and `docs/02-ARCHITECTURE.md`
for system design details.

## Build & Test

- Build: `{{BUILD_COMMAND}}`
- Lint:  `{{LINT_COMMAND}}`
- Test:  `{{TEST_COMMAND}}`

Run all three before closing any implementation task.

## Project Layout

```
docs/
  00-PRD.md                    Product requirements
  01-PROJECT-OVERVIEW.md       Tech stack and decisions
  02-ARCHITECTURE.md           System design, component structure, dependency pins
  03-IMPLEMENTATION-PLAN.md    Phased build plan with validation contracts
  04-BEADS-REFERENCE.md        Beads CLI reference
  05-OPERATIONAL-PLAN.md       Agent roles, protocols, beads structure

.devcontainer/                 Reproducible dev environment (Dockerfile + devcontainer.json)

.factory/
  droids/
    orchestrator.md            Orchestrator agent (manages development loop)
    worker.md                  Worker agent (implements individual tasks)
    validator.md               Validator agent (executes validation contracts)
  skills/
    {{PROJECT_SLUG}}-init/
      SKILL.md                 Orchestrator session init sequence

start.sh                       Launch the Orchestrator session
```

## Agent Roles

- **Orchestrator** -- Long-running interactive session (`./start.sh`). Manages the full
  development loop: creates beads epics/tasks, spawns Workers and Validators as subagents,
  monitors progress, escalates to the Overseer only when genuinely stuck.
- **Worker** -- Stateless subagent. Claims one beads task, executes it (skill building,
  validation contract authoring, or implementation), runs checks, closes the task, then reports
  back. Does NOT make architectural decisions.
- **Validator** -- Stateless subagent. Claims a `validation-run` task, executes the full
  validation contract for the phase, creates `bug` issues for every failure, closes the run.
  Never fixes code directly.

## Beads Issue Tracking

All work is tracked in beads (`bd`). Key commands:

```bash
bd ready --json                          # find available tasks
bd update <id> --claim --json            # atomically claim a task
bd close <id> --reason "<summary>" --json  # close when done
bd create "Found: <issue>" -t bug -p 1 \
  --deps discovered-from:<task-id> \
  --parent <epic-id> --json              # report a discovered issue

# For descriptions with backticks or nested quotes, use stdin:
echo 'Description with `backticks`' | bd create "Title" --description=- --json
```

See `docs/04-BEADS-REFERENCE.md` for the full reference.

## Development Conventions

- Use bd for ALL task tracking. Do not create markdown TODO lists or use external issue trackers.
- Phases are sequential. Phase N+1 cannot start until Phase N's validation contract passes.
- Build/lint/test must all exit 0 before closing any implementation task.
- Skills (`.factory/skills/`) encode domain knowledge. Read the relevant skill before starting
  implementation tasks that reference it.
- Workers must create a `discovered-from` bug issue for anything unexpected found during a task
  rather than silently skipping it.
- Validation is the hard gate: an epic cannot be closed until the Validator signs off.

## Git Workflow

- Branch from `main`: `feature/<slug>` or `bugfix/<slug>`.
- Commits are atomic and descriptive: `feat: …`, `fix: …`, `test: …`, `chore: …`.
- Never force-push `main`.
- Run build + lint + test before every commit.

## Gotchas

- Do not use `bd edit` -- it opens an interactive editor that agents cannot use. Use
  `bd update <id> --title "…"` or `bd update <id> --description "…"` with flags instead.
- Never skip the validation contract step -- it must be written and closed before implementation
  tasks begin.
- Workers do not make architectural decisions. When a task is ambiguous, create a
  `discovered-from` beads issue and block or move on.
- The Orchestrator escalates to the Overseer via AskUser only for the four conditions defined
  in `docs/05-OPERATIONAL-PLAN.md`. Routine phase transitions are automatic.
