# theoros

> *Structured problem discovery, model selection, cost estimation, and execution tracking for complex Claude Code tasks.*

Named for the ancient Greek concept of careful observation before action.

---

## Install

```bash
npx skills@latest add marinoszago/theoros
```

---

## Usage

```text
/theoros [optional task description]
```

Examples:

```text
/theoros
/theoros refactor the authentication module to use JWT
/theoros add a real-time notification system to the dashboard
```

---

## What It Does

`/theoros` runs a structured workflow before touching any code. Rather than diving straight into implementation, it interviews you, selects the right model for each phase, estimates cost upfront, then executes with live progress tracking.

```text
╔══════════════════════════════════════════════════════════════════╗
║  THEOROS v1.0  │  Phase: Build          │  Sonnet 4.6           ║
╠══════════════════════════════════════════════════════════════════╣
║  WORKFLOW                                                        ║
║  [✓] DISCOVER  →  [✓] PLAN  →  [●] BUILD  →  [ ] VERIFY         ║
╠══════════════════════════════════════════════════════════════════╣
║  ACTIVE: Step 3/7 — Migrate session storage to Redis            ║
╠══════════════════════════════════════════════════════════════════╣
║  TOKEN BUDGET                                                    ║
║  Discovery:      14,500 tkn   $0.06                             ║
║  Planning:      ~22,000 tkn   $0.18                             ║
║  Build:         ~56,000 tkn   $0.47                             ║
║  TOTAL EST:     ~92,500 tkn   $0.88                             ║
╠══════════════════════════════════════════════════════════════════╣
║  CONTEXT  ████████░░░░░░░░  47%  ◉ Good                         ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## Workflow Phases

| Phase | Name | What happens |
| ----- | ---- | ------------ |
| 0 | Initialization | Checks for a saved session to resume; shows welcome banner |
| 1 | Problem Discovery | Grouped interview — core problem, scope, constraints, success criteria |
| 2 | Synthesis | Structured problem statement; you confirm before anything is planned |
| 3 | Model Selection — Planning | Pricing table; choose the model that will create the plan |
| 4 | Plan Creation | Numbered steps with S/M/L sizes; Opus/Fable 5 spawned as a dedicated agent |
| 5 | Model Selection — Implementation | Cost comparison across models; pick a cheaper one if the plan is clear |
| 6 | Token Budget | Full cost projection displayed before execution begins |
| 7 | Execution | Step-by-step execution; dashboard updated after every step |
| 8 | Context Monitoring | Warns at 60%, halts at 80% with `/compact` + auto-resume instructions |
| 9 | Retrospective | Final summary; offers to generate a reusable skill if the task repeats |

---

## Model Pricing

Theoros shows this table during model selection so you can make an informed choice:

| Model | ID | Input $/1M | Output $/1M | Best for |
| ----- | -- | ---------- | ----------- | -------- |
| Claude Fable 5 | `claude-fable-5` | $10 | $50 | Maximum capability |
| Claude Opus 4.8 | `claude-opus-4-8` | $5 | $25 | Complex, ambiguous tasks |
| Claude Sonnet 4.6 | `claude-sonnet-4-6` | $3 | $15 | Most tasks |
| Claude Haiku 4.5 | `claude-haiku-4-5` | $1 | $5 | Simple, well-defined tasks |

You can choose **different models for planning and implementation** — for example, use Opus to design the plan, then Sonnet to execute it.

---

## Session Persistence & `/compact`

Theoros saves state to `.theoros/session.json` after every phase. If the context gets compacted or the session ends mid-task, just run `/theoros` again — it detects the saved session and resumes from the next pending step.

Add `.theoros/` to your project's `.gitignore`:

```gitignore
.theoros/
```

---

## Skill Creation

At the end of each session, theoros asks if the task was repetitive. If it was, it offers to generate a new Claude Code skill — a `/your-command` shortcut that skips the full discovery flow next time.

---

## Plugin Structure

```text
theoros/
├── .claude-plugin/
│   └── plugin.json        # Plugin metadata
└── skills/
    └── theoros/
        └── SKILL.md       # The orchestrator skill (~300 lines)
```

---

## Contributing

Issues and pull requests are welcome. The plugin logic lives entirely in `skills/theoros/SKILL.md` — no build step or dependencies required.

## License

MIT
