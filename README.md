# ocas-triage

**Version:** 1.1.0  
**Author:** Indigo Karasu  
**Type:** system  
**License:** MIT

System scheduler and priority queue manager for OpenClaw agent systems. Triage's only job is to determine what gets attention next. It maintains a durable priority queue, scores work deterministically, emits pickup signals, and handles interrupts.

## What it does

- Maintains a persistent priority queue across all pending work
- Scores tasks deterministically using urgency, deadline proximity, consequence weight, and queue aging
- Preempts active work when a higher-priority task arrives (threshold: +25 points)
- Emits pickup signals (`task_ready`, `task_completed`) for downstream consumers
- Handles debouncing, duplicate detection (>0.90 similarity = merge), and stall recovery
- Logs every state transition and decision to durable storage

## What it does not do

- Route tasks to skills (that is Dispatch)
- Decompose or plan work (that is Mentor)
- Interpret domain intent or select execution paths
- Triage raw messages or communications

## When to invoke

- "Prioritize X over Y"
- "What are you working on" / "What's pending"
- "Stop what you're doing and do this first"
- "Add this to the queue"
- "Cancel that" / "Pause" / "Resume"

## Meta commands

These execute immediately and bypass the queue entirely:

```
status / what are you working on
stop / cancel that
pause / resume
```

Meta commands never appear in `queue.jsonl`.

## Priority scoring

```
priority_score = urgency + deadline_proximity + consequence_weight +
                 interruption_intent + quick_completion_bonus + queue_aging
Clamp: 0–100
```

| Signal | Condition | Points |
|---|---|---|
| Urgency phrase | now / urgent / right away | +40 |
| Interruption intent | actually / wait / change that | +35 |
| Deadline proximity | within 1 hour | +35 |
| Consequence weight | reservation / financial / coordination | +30 |
| Quick completion | < 10 seconds | +40 |
| Quick completion | < 1 minute | +25 |
| Queue aging | per hour waiting, max +20 | +2/hr |

Full scoring model in `references/scoring_model.md`.

## Storage layout

```
.triage/
  config.json       ConfigBase + triage settings
  queue.jsonl       all task records, append-only
  signals.jsonl     task_ready / task_completed / task_acknowledged
  decisions.jsonl   DecisionRecord entries for preemption and scoring
  history.jsonl     completed/cancelled tasks, last 100
  journals/         action journal entries per run
  reports/
```

## Reference files

| File | Contents |
|---|---|
| `references/schemas.md` | Task, signal, and DecisionRecord schemas |
| `references/scoring_model.md` | Full scoring formula with signal examples |
| `references/journal_spec.md` | Journal entry structure and OKR definitions |
| `references/boundary_contracts.md` | Consumer pickup behavior and hard boundaries |

## Installation

Drop the `ocas-triage/` directory into your OpenClaw skills folder and reload your agent instance.

## License

MIT
