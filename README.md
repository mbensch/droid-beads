# droid-beads

A [Factory](https://factory.ai) plugin that bootstraps any software project from idea to autonomous multi-agent development.

Uses [beads](https://beadbox.app/) for issue tracking — all pipeline state lives in beads, making the process fully resumable. Visit [beadbox.app](https://beadbox.app/) to visualize your beads issues in a web UI.

## What it does

Walks you through a five-stage pipeline:

1. **PRD** — Interview-driven product requirements document, or ingest an existing one
2. **Technical Discovery** — Research technology options, make architecture decisions
3. **Implementation Planning** — Phased plan with validation contracts per phase
4. **Dev Environment Setup** — Generate a `.devcontainer/` tailored to your stack, verify build
5. **Scaffold Infrastructure** — Generate operational plan, beads issue structure, custom droids, init skill, and launch script

Each stage produces concrete artifacts in `docs/` and `.factory/`. All progress is tracked in beads — re-invoke at any time to resume from where you left off.

## Prerequisites

- [Factory CLI](https://factory.ai) (`droid`)
- [beads](https://beadbox.app/) (`bd`) with a running Dolt server
- Docker (for devcontainer setup)
- `git`

## Install

```bash
droid plugin marketplace add https://github.com/mbensch/droid-beads
droid plugin install droid-beads@droid-beads
```

## Usage

Create and enter your project directory, then start a droid session:

```bash
mkdir my-project && cd my-project && droid
```

Inside the session:

```
/bootstrap-project
```

The skill detects your current state and either starts fresh or resumes the pipeline. It will ask questions and pause at each stage for your review before proceeding.

To resume after a pause:

```
/bootstrap-project
```

## Output structure

After completing all stages, your project will contain:

```
docs/
  00-PRD.md                    Product requirements
  01-PROJECT-OVERVIEW.md       Tech stack and architecture decisions
  02-ARCHITECTURE.md           Architecture detail
  03-IMPLEMENTATION-PLAN.md    Phased plan with validation contracts
  05-OPERATIONAL-PLAN.md       Agent roles, protocols, beads structure

.devcontainer/
  devcontainer.json
  Dockerfile

.factory/
  droids/
    orchestrator.md            Autonomous orchestration loop
    worker.md                  Task implementation agent
    validator.md               Validation execution agent
  skills/
    <project>-init/
      SKILL.md                 Orchestrator initiation skill

start.sh                       Launch the orchestrator session
AGENTS.md                      Agent briefing packet (build commands, conventions, roles)
.factory/settings.json         Droid hooks (bd prime on session start and compact)
```

## Templates

The `skills/bootstrap-project/templates/` directory contains reusable templates used during scaffolding:

- `prd-structure.md` — PRD interview guide and output format
- `implementation-methodology.md` — Methodology section and phase template for implementation plans
- `operational-plan.md` — Full operational plan template with `{{PLACEHOLDER}}` markers
- `beads-reference.md` — Beads CLI reference for agents

## How it works

Each pipeline stage maps to a beads epic. Stages are dependency-chained (stage N+1 is blocked until stage N is closed). The skill reads epic status on every invocation to determine where to resume.

The orchestrator droid produced by stage 5 reads the implementation plan, creates beads tasks per phase, and spawns worker and validator agents as subagents via the Factory Task tool.

## License

MIT
