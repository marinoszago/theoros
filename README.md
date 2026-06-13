# theoros

A Claude Code plugin that acts as a meta-orchestrator for complex tasks. It guides your work through structured problem discovery, model selection, cost estimation, step-by-step execution with visual progress, and context health monitoring.

*Named for the ancient Greek concept of careful observation before action.*

## Install

```bash
claude plugin install github:marinoszagkotsis/theoros
```

## Usage

```
/theoros [optional task description]
```

Examples:
```
/theoros
/theoros refactor the authentication module to use JWT
/theoros add a real-time notification system to the dashboard
```

## What It Does

Invoking `/theoros` starts a structured 9-phase workflow:

### Phase 1 — Problem Discovery Interview
Asks targeted questions grouped by topic (core problem, scope, success criteria, risks). Adapts based on what you've already described in the argument.

### Phase 2 — Problem Synthesis
Summarizes your answers into a structured problem statement and asks for confirmation before proceeding.

### Phase 3 — Model Selection (Planning)
Presents a model comparison table with current pricing and asks which model should create the plan.

| Model | Input $/1M | Output $/1M | Best for |
|-------|-----------|------------|---------|
| Claude Opus 4.8 | $5 | $25 | Complex, ambiguous tasks |
| Claude Sonnet 4.6 | $3 | $15 | Most tasks (recommended) |
| Claude Haiku 4.5 | $1 | $5 | Simple, well-defined tasks |
| Claude Fable 5 | $10 | $50 | Maximum capability |

### Phase 4 — Plan Creation
Creates a numbered plan with step sizes (S/M/L) and risk notes. If you selected Opus or Fable 5, spawns a dedicated planning agent using that model.

### Phase 5 — Implementation Model Selection
Shows projected cost for each model and lets you choose a (potentially cheaper) model for implementation, separate from the planning model.

### Phase 6 — Token Budget Dashboard
Displays a full cost projection before execution begins:

```
╔══════════════════════════════════════════════════════════════════╗
║  THEOROS v1.0  │  Phase: Planning       │  Opus 4.8             ║
╠══════════════════════════════════════════════════════════════════╣
║  WORKFLOW                                                        ║
║  [✓] DISCOVER  →  [●] PLAN  →  [ ] BUILD  →  [ ] VERIFY         ║
╠══════════════════════════════════════════════════════════════════╣
║  ACTIVE: Creating implementation plan                           ║
╠══════════════════════════════════════════════════════════════════╣
║  TOKEN BUDGET                                                    ║
║  Discovery:      14,500 tkn   $0.08                             ║
║  Planning:      ~22,000 tkn   $0.28                             ║
║  Build:         ~56,000 tkn   $0.52                             ║
║  TOTAL EST:     ~92,500 tkn   $0.88                             ║
╠══════════════════════════════════════════════════════════════════╣
║  CONTEXT  ████████░░░░░░░░  47%  ◉ Good                         ║
╚══════════════════════════════════════════════════════════════════╝
```

### Phase 7 — Execution
Runs each step, printing progress and updating the dashboard after every step.

### Phase 8 — Context Health Monitoring
Automatically warns when context usage approaches dangerous levels:
- **60–79%:** Warning with instructions to run `/compact`
- **≥80%:** Critical — pauses execution and instructs you to compact now

Session state is saved to `.theoros/session.json` in your working directory. After running `/compact`, just run `/theoros` again to resume from where you left off.

### Phase 9 — Completion & Retrospective
Shows a final summary with actual step count, token estimates, and cost. Then asks:

> *"Is this a task you'll need to do regularly?"*

If yes, theoros offers to create a reusable Claude Code skill so next time it's a single command instead of a full workflow.

## Session Resumption

Theoros saves session state continuously to `.theoros/session.json`. If your context gets compacted or the session ends mid-task:

```
/theoros
```

It will detect the existing session and offer to resume, picking up exactly where it left off.

The `.theoros/` directory is local to each project. You may want to add it to `.gitignore`:

```
.theoros/
```

## Plugin Structure

```
theoros/
├── .claude-plugin/
│   └── plugin.json        # Plugin metadata
└── skills/
    └── theoros/
        └── SKILL.md       # The orchestrator skill
```

## License

MIT
