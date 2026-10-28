---
name: delegate
description: Run a task as an orchestrator over Sonnet workers with one bounded escalation to a top-tier model. Use when the user invokes /delegate, or asks to "delegate this", "orchestrate this", "run this with workers", or "use the advisor pattern". Decomposes the task, fans out parallel workers that cannot guess and cannot self-escalate, and spends at most one call to a top-tier model at the single hardest moment.
---

# delegate

You are the orchestrator. You plan, dispatch, and synthesize. You do not read files, grep, edit, or run commands to accomplish the task — that is worker work. Your context is the scarce resource; an orchestrator that has read thirty files reasons worse than one that has read none.

Three roles, three models:

| Role | Model | Can it act? | How often |
|---|---|---|---|
| Orchestrator (you) | whatever the session is on | plans, dispatches, synthesizes | continuously |
| `delegate-worker` | `sonnet` | reads, edits, runs, verifies | once per subtask, in parallel |
| `delegate-advisor` | top tier only: the strongest model you have access to | nothing — read-only judgment | **at most once per run** |

## The rule that makes this work

**The advisor is called at most once.** Not once per subtask. Once per run.

This is the whole discipline. The top-tier model has to stay rare. That rarity is the entire reason the pattern is worth running. An advisor consulted at every checkpoint is just an expensive way to think out loud. If you find yourself wanting a second advisor call, that means the first one was spent at the wrong moment or the decomposition was wrong. Fix that instead.

Routine tasks get **zero** advisor calls. Most tasks are routine.

A second advisor call requires asking the user first, with a one-line reason. Never fire it silently.

## Step 1 — Scout, inline

Do this yourself. `Glob` and `Grep` only. No subagents, no `Read` of whole files unless a file is short and load-bearing. You are building enough of a map to decompose, nothing more.

Stop when you can name the files that will change and the shape of the change. If you cannot get there in a handful of tool calls, the task is ambiguous — that is a signal, and Step 2 tells you what to do with it.

## Step 2 — Decide where the hard part is

This is the only real decision you make, and it determines when — or whether — the advisor fires. Pick exactly one:

**The hard part is planning.** The decomposition is genuinely unclear: competing architectures, no obvious seam, or the task touches a design you do not understand. → Call the advisor now, in `plan` mode. Give it the task and your scout. Use its `items` and `constraints` as your dispatch. **Advisor budget now spent.**

**The hard part is execution.** The decomposition is obvious but a subtask may hit something you cannot foresee. → Decompose yourself. Dispatch. Hold the advisor call in reserve for the first worker that returns `blocked`.

**The hard part is risk.** The change is small and clear but destructive, security-sensitive, irreversible, or alters a contract other code depends on. → Decompose yourself, dispatch, and hold the advisor call for a `review` pass on the finished diff.

**There is no hard part.** → Decompose, dispatch, synthesize. No advisor. Say so in your summary; it is a good outcome, not a missed opportunity.

If two of these seem to apply, the task is too big. Split it and run `/delegate` on the pieces.

## Step 3 — Dispatch workers

Send every independent worker in **one message with multiple `Agent` tool calls**. Sequential dispatch of independent work is the most common waste in orchestrated sessions. Cap at 3-5 concurrent; coordination overhead beats parallel speedup past that.

Each dispatch to `delegate-worker` carries:

- the subtask `goal` and its `done_when` condition
- the constraints that keep it coherent with sibling subtasks
- any advisor `constraints` / `do_not` from a `plan` call
- an explicit note that it must return `blocked` rather than guess

Workers have no `Agent` tool. They cannot escalate around you. Every escalation lands here, visible and countable.

## Step 4 — Handle blocks

A worker returns `{"status": "blocked", "blocked": {question, options_considered, evidence}}`.

First, try to answer it yourself. You are running on a strong model and you hold the plan; many blocks are questions the worker could not see the answer to because it lacked the whole picture. If you know the answer, re-dispatch with the decision attached. This costs nothing.

If you genuinely do not know — and only if your advisor budget is unspent — call `delegate-advisor` in `unblock` mode with the block, the evidence, and the plan. Re-dispatch the worker with `decision`, `constraints`, and `do_not` attached. **Advisor budget now spent.**

**One re-dispatch per subtask.** A subtask that blocks a second time stops. Surface it to the user with both blocks and what you tried. Do not loop.

**If more than one subtask blocks, the plan was wrong.** Halt. Do not grind through blocks one at a time. Report the blocks, say the decomposition failed, and propose a different split.

## Step 5 — Synthesize

Collect the worker summaries. Report to the user:

- what changed, in prose, as one narrative rather than N stitched summaries
- what was actually verified, and what was not — a worker returning `verified: null` means nothing was checked, and you must say so rather than let the absence read as success
- whether the advisor was called, in which mode, and what it said
- anything a worker flagged that falls outside its subtask

If Step 2 selected **risk**, call `delegate-advisor` in `review` mode on the finished diff before you write this summary. A `correct` verdict is **advisory**: report the issues and stop. Do not auto-re-execute. Auto-correction loops are how a fifteen-minute task becomes an overnight one.

## What this does and does not buy you

It buys structured delegation, bounded escalation, and — when the advisor fires — a judgment from a top-tier model in a context that has not been anchored by everything you already decided. If your main thread is already on that tier, the advisor is a fresh-context reviewer rather than a step up in capability. Call it that when you report it.

It does not cut the cost of the main thread itself. Your main thread stays whatever model it is. What you save is orchestrator context; what you gain is one escalation you could not otherwise reach.

If the task is small, `/delegate` is overhead. Do it directly.
