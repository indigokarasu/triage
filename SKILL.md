---
name: ocas-triage
version: 1.1.0
description: >
  System scheduler and priority queue manager. Determines what gets attention
  next across all pending work. Use when prioritizing competing tasks, checking
  queue state, preempting active work, or auditing execution order. Does not
  route tasks, select skills, or decompose work.
visibility: private
type: system
---

# Triage

Triage is the system scheduler. Its only job is to determine what gets
attention next. It maintains a durable priority queue, scores work
deterministically, emits pickup signals, and handles interrupts.

## When to use

- Prioritize competing tasks: "prioritize X over Y"
- Check queue state: "what are you working on", "what's pending"
- Interrupt active work: "stop what you're doing and do this first"
- Add work to the queue: "add this to the queue"
- Manage queue: "cancel that", "pause", "resume"

## When not to use

- Project orchestration — use Mentor
- Message drafting or inbox triage — use Dispatch
- Skill execution requests — route directly to the target skill
- Pattern analysis — use Corvus

## Core promise

One active task at a time. Deterministic scoring. Durable queue. Pickup
signals emitted for all consumers. Every task logged.

## Meta commands

These execute immediately and bypass the queue entirely:

```
status / what are you working on
stop / cancel that
pause / resume
```

Meta commands never appear in `queue.jsonl`.

## Task model

See `references/schemas.md` for full schema.

A task is created for every actionable input. Not created for: thanks, ok,
casual conversation, clarification questions, or meta queries.

**States:** `queued` → `active` → `completed` or `cancelled`
Lateral: `blocked` (retries exceeded), `waiting_external` (awaiting response)

**Debounce:** 1-second window groups rapid messages for relationship detection.

**Update relationships:** `override`, `correction`, `additional_context`, `duplicate`

**Duplicate rule:** similarity > 0.90 within 5 minutes → merge, +5 priority
to existing, no new task created.

**Deadline:** explicit (user states time) or inferred (task references a known
event). Null if no reliable deadline exists.

**Estimated completion:** heuristic buckets (<10s, <1m, 1–10m, 10–30m, >30m).
Null defaults to 1800s for scoring. Historical completions refine estimates.

## Priority scoring

Full model in `references/scoring_model.md`. Formula:

```
priority_score = urgency + deadline_proximity + consequence_weight +
                 interruption_intent + quick_completion_bonus + queue_aging
Clamp: 0–100
```

| Signal | Condition | Points |
|---|---|---|
| Urgency phrase | now / urgent / right away / before [time] | +40 |
| Interruption intent | actually / wait / change that / stop | +35 |
| Deadline proximity | within 1 hour | +35 |
| Consequence weight | reservation / financial / coordination | +30 |
| Deadline proximity | same day | +20 |
| Quick completion | < 10 seconds | +40 |
| Quick completion | < 1 minute | +25 |
| Quick completion | < 5 minutes | +15 |
| Quick completion | < 30 minutes | +5 |
| Deadline proximity | within week | +10 |
| Queue aging | per hour waiting, max +20 | +2/hr |

`waiting_external` tasks do not accrue queue aging.

## Task selection and preemption

**Selection formula:** `task_score = priority_score / max(estimated_completion_seconds / 60, 1)`

**Tie-breaking:** earlier deadline → shorter estimated time → older `created_at`

**Preemption:** if new task priority exceeds active task by > 25 points:
- checkpoint active task (preserve `last_execution_time`, `partial_result`,
  `progress_marker`)
- return to `queued` with original score + +10 resume bonus
- run higher-priority task
- emit journal entry + DecisionRecord

**Queue capacity:** `max_queue_size = 50`. Overflow: merge duplicates first,
then drop lowest-priority `queued` task.

**Recalculation triggers:** task created/updated/completed/cancelled,
reactivation from `waiting_external`, deadline change, every 60 seconds.

**Task expiration:** 7-day TTL for tasks with no update. Explicit deadlines
expire when the deadline passes and execution is no longer useful.

**History:** completed/cancelled tasks appended to `history.jsonl`,
limit 100.

## Queue pickup

Triage writes to `.triage/signals.jsonl`. Consumers poll this file.

`task_ready` — emitted when a task becomes active
`task_completed` — emitted on completion or cancellation
`task_acknowledged` — written by consumer to prevent double-pickup

See `references/schemas.md` for signal schemas.
See `references/boundary_contracts.md` for consumer pickup behavior.

## Stall detection and recovery

Stall threshold: 120 seconds of no progress.
Retry limit: 3, backoff: exponential.
On limit exceeded: `state → blocked`, `blocking_reason` logged.

**Queue integrity:** if storage is corrupted or queue is unexpectedly empty,
rebuild from `queue.jsonl` + `history.jsonl`. If reconstruction fails, clear
invalid entries and preserve highest-confidence records.

## Boundary contracts

See `references/boundary_contracts.md` for full rules.

**vs Dispatch** — Dispatch produces actionable items from communications.
Those enter the Triage queue with `origin: dispatch`. Triage never touches
raw messages.

**vs Mentor** — Mentor schedules within a project. Triage schedules across
all work. Tasks with `routing_hint: mentor` are picked up by Mentor on its
heartbeat pass. Mentor never re-implements queue aging or urgency scoring.

**vs base agent** — Short-path tasks handled directly still enter the queue
with `routing_hint: direct` for audit continuity.

**Triage never:** selects skills, routes tasks, decomposes work, interprets
domain intent, or invokes execution systems.

## Journal output

Emits Action Journals (spec-ocas-journal.md §2.1, journal_spec_version: 1.2).

Events journaled: task created, task scored, task selected, preemption,
task completed/cancelled, meta command, stall detected, retry, queue rebuild.

See `references/journal_spec.md` for full entry structure and OKRs.

## Support file map

- `references/schemas.md` — Task, signal, and DecisionRecord schemas
- `references/scoring_model.md` — Full scoring formula with signal examples
- `references/journal_spec.md` — Journal entry structure and OKR definitions
- `references/boundary_contracts.md` — Consumer pickup behavior and hard boundaries

## Storage layout

```
.triage/
  config.json       ConfigBase + triage settings
  queue.jsonl       all task records, append-only; state transitions are
                    new records with same task_id, not mutations
  signals.jsonl     task_ready, task_completed, task_acknowledged records
  decisions.jsonl   DecisionRecord entries for preemption and scoring
  history.jsonl     completed/cancelled tasks, last 100
  journals/         action journal entries per run
  reports/
```

Current queue state: read `queue.jsonl`, take latest record per `task_id`.

## Validation rules

- No two tasks simultaneously in `active` state
- Every active task has exactly one `active` record in `queue.jsonl`
- Meta commands never appear in `queue.jsonl`
- Preemption always produces a journal entry and DecisionRecord
- `task_ready` signal precedes `active` state for same `task_id`
- Every `waiting_external` task has a non-null `blocking_reason`
- `history.jsonl` does not exceed 100 records
- Queue does not exceed `max_queue_size`
- `journal_completeness` = 1.0 for all runs
- `okr_evaluation` block present in every journal entry
