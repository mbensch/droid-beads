---
name: bootstrap-project
description: >-
  Bootstrap any software project from idea to autonomous multi-agent development. Guides you
  through PRD creation, technical discovery, implementation planning, dev environment setup, and
  infrastructure scaffolding. All state is tracked in beads. Re-invoke at any time to resume.
  Use when starting a new project from scratch.
disable-model-invocation: true
---

# Bootstrap Project

This skill walks you through a five-stage pipeline to take any project from idea to autonomous
multi-agent development. Each stage produces a concrete artifact stored in the project. All
progress is tracked in beads -- re-invoke `/bootstrap-project` at any time to resume from where
you left off.

## Entry Point: Detect State

On every invocation, first check whether beads is initialized:

```bash
bd info --json 2>/dev/null
```

- **Exit code non-zero / command not found**: run Stage 0 (first-time init)
- **Exit code 0**: query beads for open bootstrap epics and resume

## Stage 0: First-Time Init

Run this only if beads is not yet initialized.

Ask the user two questions via AskUser:
1. What is the project name? (human-readable, e.g. "GhostDev")
2. What beads prefix should be used? (short, e.g. "gd", "bd", "proj")

Then:

```bash
git init
bd init -p <prefix>
```

Now create all five bootstrap epics with the dependency chain (Epic 1 → 2 → 3 → 4 → 5).
Each epic blocks the next via `bd dep add`:

```bash
# Epic 1: PRD
bd create "Bootstrap: PRD" -t epic -p 1 --label bootstrap,phase-prd --json
# → returns <epic1-id>
bd create "Gather product context" -t task -p 1 --parent <epic1-id> --label bootstrap,phase-prd --json
bd create "Write PRD document" -t task -p 1 --parent <epic1-id> --label bootstrap,phase-prd --json

# Epic 2: Technical Discovery
bd create "Bootstrap: Technical Discovery" -t epic -p 1 --label bootstrap,phase-discovery --json
# → returns <epic2-id>
bd create "Research technology options" -t task -p 1 --parent <epic2-id> --label bootstrap,phase-discovery --json
bd create "Write Project Overview" -t task -p 1 --parent <epic2-id> --label bootstrap,phase-discovery --json
bd create "Write Architecture" -t task -p 1 --parent <epic2-id> --label bootstrap,phase-discovery --json

# Epic 3: Implementation Planning
bd create "Bootstrap: Implementation Planning" -t epic -p 1 --label bootstrap,phase-planning --json
# → returns <epic3-id>
bd create "Write Implementation Plan" -t task -p 1 --parent <epic3-id> --label bootstrap,phase-planning --json

# Epic 4: Dev Environment Setup
bd create "Bootstrap: Dev Environment Setup" -t epic -p 1 --label bootstrap,phase-devenv --json
# → returns <epic4-id>
bd create "Verify Docker is available" -t task -p 1 --parent <epic4-id> --label bootstrap,phase-devenv --json
bd create "Generate devcontainer config" -t task -p 1 --parent <epic4-id> --label bootstrap,phase-devenv --json
bd create "Verify devcontainer builds" -t task -p 1 --parent <epic4-id> --label bootstrap,phase-devenv --json

# Epic 5: Scaffold Infrastructure
bd create "Bootstrap: Scaffold Infrastructure" -t epic -p 1 --label bootstrap,phase-scaffold --json
# → returns <epic5-id>
bd create "Generate Operational Plan" -t task -p 1 --parent <epic5-id> --label bootstrap,phase-scaffold --json
bd create "Generate Beads Reference" -t task -p 1 --parent <epic5-id> --label bootstrap,phase-scaffold --json
bd create "Create Orchestrator droid" -t task -p 1 --parent <epic5-id> --label bootstrap,phase-scaffold --json
bd create "Create Worker droid" -t task -p 1 --parent <epic5-id> --label bootstrap,phase-scaffold --json
bd create "Create Validator droid" -t task -p 1 --parent <epic5-id> --label bootstrap,phase-scaffold --json
bd create "Create Init skill" -t task -p 1 --parent <epic5-id> --label bootstrap,phase-scaffold --json
bd create "Create start.sh" -t task -p 1 --parent <epic5-id> --label bootstrap,phase-scaffold --json
bd create "Generate AGENTS.md" -t task -p 1 --parent <epic5-id> --label bootstrap,phase-scaffold --json
bd create "Generate droid hooks" -t task -p 1 --parent <epic5-id> --label bootstrap,phase-scaffold --json

# Wire dependencies: each epic blocks the next
bd dep add <epic2-id> <epic1-id> --type blocks
bd dep add <epic3-id> <epic2-id> --type blocks
bd dep add <epic4-id> <epic3-id> --type blocks
bd dep add <epic5-id> <epic4-id> --type blocks
```

Store the project name and prefix in the Epic 1 description for later retrieval:
```bash
bd update <epic1-id> -d "project_name: <name>\nbeads_prefix: <prefix>" --json
```

Fall through to resume logic (Epic 1 will be the earliest open epic).

## Resume Logic

On every invocation (including after Stage 0), find the earliest open bootstrap epic:

```bash
bd list -t epic --label bootstrap --status open --json
```

Sort by creation order, take the first result. Then find its first open task:

```bash
bd list --parent <epic-id> --status open --json
```

Work through tasks in order. When all tasks in the epic are closed, close the epic:

```bash
bd close <epic-id> --reason "Stage complete" --json
```

Then check if there are more open bootstrap epics. If yes, start the next one. If no, the
bootstrap is complete -- print the completion message.

---

## Epic 1: PRD

**Goal:** Produce `docs/00-PRD.md`.

### Task: Gather product context

Ask the user via AskUser: "Do you have an existing PRD or product spec document?"

- **Yes** → Ask them to provide the file path. Read it. Move on.
- **No** → Conduct a structured interview using the section guide in
  `skills/bootstrap-project/templates/prd-structure.md` (in the plugin). Ask one section at a
  time, synthesize answers.

### Task: Write PRD document

Write the gathered content to `docs/00-PRD.md`. Create the `docs/` directory if needed.

Close both tasks, then close the epic.

**Review pause:** Tell the user:

> PRD written to `docs/00-PRD.md`. Review it, make any edits, then run `/bootstrap-project` to continue to Technical Discovery.

---

## Epic 2: Technical Discovery

**Goal:** Produce `docs/01-PROJECT-OVERVIEW.md` and `docs/02-ARCHITECTURE.md`.

### Task: Research technology options

Read `docs/00-PRD.md`. Identify the technical domain (web app, desktop app, CLI tool, API, etc.).

Use WebSearch to research relevant technology options (language, framework, key libraries). Focus
on the "Open Questions for Engineering" or similar sections in the PRD if present.

Present trade-offs and ask key decisions via AskUser (e.g. language, framework, runtime, key
libraries). Record decisions.

### Task: Write Project Overview

Write `docs/01-PROJECT-OVERVIEW.md` covering: what it is, why the chosen technology stack (with
alternatives considered and reasons for rejection), core design principles, target platforms.

### Task: Write Architecture

Write `docs/02-ARCHITECTURE.md` covering: system diagram (ASCII or Mermaid), component/module
structure, data flows, dependency version pins, and the build/lint/test commands for this stack.

Make the build/lint/test commands explicit -- they will be extracted for template substitution
in Epic 5. Format them clearly, e.g.:

```
## Build Commands
- Build: `cargo build --workspace`
- Lint: `cargo clippy --workspace`
- Test: `cargo test --workspace`
```

Close all tasks, close the epic.

**Review pause:**

> Architecture written. Review `docs/01-PROJECT-OVERVIEW.md` and `docs/02-ARCHITECTURE.md`, then run `/bootstrap-project` to continue.

---

## Epic 3: Implementation Planning

**Goal:** Produce `docs/03-IMPLEMENTATION-PLAN.md`.

### Task: Write Implementation Plan

Read `docs/00-PRD.md`, `docs/01-PROJECT-OVERVIEW.md`, `docs/02-ARCHITECTURE.md`.

Follow the methodology and structure defined in
`skills/bootstrap-project/templates/implementation-methodology.md` (in the plugin).

The plan must be phased. Each phase must include:
- Goal statement
- Tasks list
- Skills Required (knowledge a Worker needs before implementing)
- Acceptance Criteria
- Validation Contract (preconditions, automated assertions, behavioral assertions, performance
  assertions, regression guards)

Close the task and epic.

**Review pause:**

> Implementation plan written to `docs/03-IMPLEMENTATION-PLAN.md`. Review each phase carefully -- this is what the autonomous agents will build from. Run `/bootstrap-project` when ready.

---

## Epic 4: Dev Environment Setup

**Goal:** Produce `.devcontainer/devcontainer.json` + `.devcontainer/Dockerfile`, verified to build.

### Task: Verify Docker is available

```bash
docker --version
docker compose version
```

If either fails, present install instructions via AskUser. Wait for confirmation, re-check.
Do not proceed until Docker is confirmed working.

### Task: Generate devcontainer config

Read `docs/02-ARCHITECTURE.md` to extract the technology stack.

Generate `.devcontainer/Dockerfile` with:
- Base image appropriate to the stack (e.g. `rust:1`, `node:22`, `python:3.12`, `golang:1.22`)
- All core runtimes and build tools the project needs (exact versions if pinned in architecture doc)
- Dev tools: `git`, `ripgrep`, beads CLI installation
- Any platform-specific system dependencies (e.g. WebKitGTK for Tauri on Linux)

Generate `.devcontainer/devcontainer.json` with:
- `"build": { "dockerfile": "Dockerfile" }`
- `"workspaceFolder": "/workspace"`
- `"workspaceMount": "source=${localWorkspaceFolder},target=/workspace,type=bind"`
- Relevant VS Code/Cursor extensions for the stack (optional)
- `"postCreateCommand"` for any one-time setup

Create the `.devcontainer/` directory if needed.

### Task: Verify devcontainer builds

```bash
docker build -f .devcontainer/Dockerfile .devcontainer/
```

If the build fails, diagnose the error, fix the Dockerfile, and retry until it succeeds.

Close all tasks, close the epic.

**Review pause:**

> Dev environment ready. `.devcontainer/` created and verified. Run `/bootstrap-project` to scaffold the agent infrastructure.

---

## Epic 5: Scaffold Infrastructure

**Goal:** Generate all operational files from templates.

First, retrieve the project name and beads prefix from the Epic 1 description:
```bash
bd show <epic1-id> --json
```

Extract from the description: `project_name` and `beads_prefix`.

Extract build/lint/test commands from `docs/02-ARCHITECTURE.md` (look for the "Build Commands"
section).

Template parameters:
- `{{PROJECT_NAME}}` -- project name from Epic 1
- `{{PROJECT_SLUG}}` -- project name lowercased with spaces replaced by hyphens (e.g. "My App" → "my-app")
- `{{BEADS_PREFIX}}` -- prefix from Epic 1
- `{{BUILD_COMMAND}}` -- from architecture doc
- `{{LINT_COMMAND}}` -- from architecture doc
- `{{TEST_COMMAND}}` -- from architecture doc
- `{{ORCHESTRATOR_MODEL}}` -- default: `claude-opus-4-5`

The templates live in the plugin directory alongside this skill. Read each template, substitute
all `{{PLACEHOLDER}}` markers, and write the output file.

### Task: Generate Operational Plan

Read `skills/bootstrap-project/templates/operational-plan.md`, substitute parameters, write to
`docs/05-OPERATIONAL-PLAN.md`.

### Task: Generate Beads Reference

Read `skills/bootstrap-project/templates/beads-reference.md` (no substitution needed), write to
`docs/04-BEADS-REFERENCE.md`.

### Task: Create Orchestrator droid

Create `.factory/droids/` if needed.
Read `skills/bootstrap-project/templates/droids/orchestrator.md`, substitute, write to
`.factory/droids/orchestrator.md`.

### Task: Create Worker droid

Read `skills/bootstrap-project/templates/droids/worker.md`, substitute, write to
`.factory/droids/worker.md`.

### Task: Create Validator droid

Read `skills/bootstrap-project/templates/droids/validator.md`, substitute, write to
`.factory/droids/validator.md`.

### Task: Create Init skill

Create `.factory/skills/init/` if needed.
Read `skills/bootstrap-project/templates/skills/init.md`, substitute, write to
`.factory/skills/init/SKILL.md`.

### Task: Create start.sh

Write `start.sh`:

```bash
#!/usr/bin/env bash
# Launch the {{PROJECT_NAME}} Orchestrator session.
exec droid --droid orchestrator
```

Make it executable: `chmod +x start.sh`

### Task: Generate AGENTS.md

Read `skills/bootstrap-project/templates/agents-md.md`, substitute all `{{PLACEHOLDER}}` markers
using the same template parameters as the other Epic 5 tasks (including `{{PROJECT_SLUG}}`,
derived from the project name lowercased with spaces replaced by hyphens), and write the result
to `AGENTS.md` at the project root.

### Task: Generate droid hooks

Create `.factory/` directory if it does not already exist. Write `.factory/settings.json` with
hooks that run `bd prime` on session start and before context compaction:

```json
{
  "hooks": {
    "PreCompact": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "bd prime"
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "bd prime"
          }
        ]
      }
    ]
  }
}
```

---

Close all tasks, close the epic.

Make an initial git commit:
```bash
git add -A
git commit -m "chore: bootstrap project infrastructure"
```

**Completion message:**

> Bootstrap complete!
>
> Your project is ready for autonomous development.
>
> Files created:
> - docs/00-PRD.md          -- Product spec
> - docs/01-PROJECT-OVERVIEW.md -- Technical framing
> - docs/02-ARCHITECTURE.md -- System design
> - docs/03-IMPLEMENTATION-PLAN.md -- Phased build plan
> - docs/04-BEADS-REFERENCE.md -- Beads CLI reference
> - docs/05-OPERATIONAL-PLAN.md -- Agent protocol
> - .devcontainer/          -- Reproducible dev environment
> - .factory/droids/        -- Orchestrator, Worker, Validator agents
> - .factory/skills/init/   -- Orchestrator init sequence
> - start.sh                -- Launch orchestrator
> - AGENTS.md               -- Agent briefing packet (build commands, conventions, roles)
> - .factory/settings.json  -- Droid hooks (bd prime on session start and compact)
>
> Run `./start.sh` to launch the Orchestrator and begin Phase 1.
> Track progress at https://beadbox.app/
