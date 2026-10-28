---
name: build
description: >
  Thin conductor that chains existing skills (/plan, simplify, /qa-only,
  review-and-ship) into one gated end-to-end feature pipeline, sized to the task and
  stopping hard before merge. Accepts an already-approved plan file to skip straight
  to implementation, and delegates planning to /plan when none was supplied.
  Trigger on "build <feature>", "implement X fully",
  "take this end to end", "full pipeline", or /build. Do NOT trigger for: "the build
  is failing" / docker build / npm run build errors / CI build failures (route to
  normal debugging or /investigate) or bug reports (route to a debugging skill) — those are
  troubleshooting, not feature construction.
---

# /build — feature pipeline conductor

This skill **invokes** other skills — `/plan`, `simplify`, gstack `/qa-only`,
`review-and-ship` — and consumes their summaries. It never reimplements their logic.
If a stage skill's checklist changes, this conductor inherits the change
automatically; do not copy stage-skill detail into this file.

**Planning is not this skill's job.** `/plan` owns the fog check, brainstorming, the
draft, and the `/autoplan` gauntlet. Stage 1 delegates to it. Likewise, Stage 5
delegates the whole review-and-ship loop, so never run that loop again after this
skill finishes; it already ran.

Main thread orchestrates only: no bulk reading, no direct implementation edits on the
main thread. Implementation happens in an isolated worktree via Sonnet-model
subagents. Never auto-merge — see Stage 6.

## Worker lifecycle

Workers run on the Sonnet model (`model: "sonnet"`); the main thread stays on the
session model (your strongest model) and thinks, dispatches, synthesizes — nothing else.

Workers are background agents and **stay addressable after they report**. When a
later stage produces follow-up work on something a live worker built — review
findings against its files (Stage 5), a smoke failure in its flow (Stage 4) — send
the follow-up to **that worker** via SendMessage instead of spawning a fresh agent.
It already holds the context of what it wrote and why; a fresh agent pays full
catch-up cost to reconstruct it. Spawn new workers only for genuinely new,
independent scopes. Record live worker IDs and their scopes in the state file so a
resumed session knows who is still addressable.

## Orchestrator discipline — think before dispatching

Orchestration is a design act, not a reflex. Before every dispatch batch, answer
three questions to yourself — and if you can't answer one, that's the signal to stop
and think, not to dispatch anyway:

1. **Goal check** — does this stage/dispatch still move toward the feature the user
   asked for, or has the pipeline drifted toward something adjacent?
2. **Decomposition check** — is this the right split? Are the pieces genuinely
   independent, sized so one worker can own one coherent scope, and free of hidden
   ordering dependencies? A wrong split costs more than a slow start.
3. **Waste check** — what would make this dispatch worthless (missing prerequisite,
   ambiguous instruction, work another live worker already owns)? Fix that first.

When a decomposition or design choice is genuinely uncertain — competing approaches,
unclear seam, a call you keep flip-flopping on — use gstack `/codex` in **consult
mode as a thinking partner** to challenge your own approach *before* burning worker
tokens on it. Codex is not just the Stage 5 review gate: consult it on architecture
seams, decomposition options, or "am I overcomplicating this?" — a cross-model
second opinion is cheapest at the planning moment, most expensive after the code
exists. Keep it bounded: one consult per genuine uncertainty, not a reflex before
every stage.

## Gate states

Every stage resolves to exactly one of:

- **green** — auto-proceed to the next stage.
- **checkpoint** — pipeline paused, waiting on user input. This is a normal resting
  state, not a failure (Stage 1 is interactive by design, via `/plan`).
- **red** — stop and report. A hung stage, a timed-out subagent, or an unresolved
  blocker all count as red, not "still running."

## Durable state

At every stage transition, write/update `.context/build-<feature-slug>.md` in the
**primary checkout** (not the worktree — it must survive worktree removal and exists
before Stage 3 creates one): stage reached, worktree path, branch, plan file path, PR URL (once it
exists), open blockers. A resumed session (after compaction or a new turn) reads this
file first before asking the user anything. This is the only way long builds survive
context loss.

## Stage 0 — Entry mode and size (always)

**Entry mode first.** If the caller supplies an approved plan file
(`/build <path-to-plan.md>`, or a caller that names one), Stage 1 is already done:
record the path in the state file and start at Stage 3. Never re-plan a plan the user
already had reviewed. `/plan` is the upstream skill that produces these, at
`.context/build-<slug>-plan.md`. Otherwise run the full Stage 1–6 pipeline.

If the previous turn ran `/plan` but no path was passed in, ask for the path rather
than planning again from scratch.

Then classify:

- **trivial**: one file changed AND a one-sentence description AND touches none of
  the risk surfaces below. Skip the pipeline — implement directly, run `simplify`
  then gstack `/qa-only`, and offer `/ship`.
- **standard/large**: everything else. Run the full Stage 1–6 pipeline.

**Risk surfaces force the full pipeline regardless of diff size**: billing and payments,
auth/entitlements, anything that publishes or sends on its own, database schema or migrations, anything
client-facing in production, deploy/infra files. When uncertain, size up — a skipped
pipeline on a risky change costs far more than extra review on a small one.

## Stage 1 — Plan (delegated)

Invoke `/plan`. It runs the fog check, brainstorming when requirements are ambiguous,
the draft, and the `/autoplan` gauntlet, and returns the path to an approved plan file
at `.context/build-<slug>-plan.md` in the primary checkout. Record that path in the
state file before Stage 3, because every implementer is pointed at it.

Do not draft or review a plan here. One owner for planning means the two skills
cannot drift apart, and it is why this stage is four lines instead of forty.

**Green** = an approved plan file exists at a recorded path, with no unresolved
majors.

**Checkpoint** = `/plan` is waiting on the user, at its brainstorm gate or its
gauntlet gate, or it recommended `/wayfinder` and stopped. Both are normal resting
states, not failures.

## Stage 3 — Implement

Create the isolated worktree first (via `EnterWorktree` or `git worktree`, branched
from the caller-supplied base branch — default: current main — named after the
feature slug); confirm it is clean before
dispatching. Never implement on main or the primary checkout.

Dispatch Sonnet-model subagents, independent pieces in one parallel message. Point
every implementer at the **plan file path**, never a re-told summary — prevents
drift between what was approved and what gets built.

Every dispatch prompt must explicitly instruct the subagent to follow **ponytail
minimal-code discipline**: smallest working solution, YAGNI, no new env vars, no
speculative abstraction — while never simplifying away validation, error handling,
security, or accessibility. Ponytail does not auto-activate inside fresh subagents;
state it every time.

**Green** = implementers report done, each with an explicit file list.

## Stage 3.5 — Simplify (always)

Invoke `simplify` on the worktree diff. It reviews the changed code for reuse,
simplification, efficiency, and altitude, then **applies** the fixes. Quality only —
it does not hunt for bugs, so it never substitutes for Stage 5.

This stage is not redundant with Stage 5's overengineering reviewer. That reviewer
only *flags*, and overengineering findings rarely rank P1, so in practice they are
reported and never fixed. Simplify is the stage that actually removes the speculative
abstraction, the one-caller interface, the config for a value that never changes.
Running it before Stage 4 also means QA and review see the simplified code rather
than code that is about to change underneath them.

Never let this stage strip input validation, error handling, security controls, or
accessibility — simplify is about less code, not a weaker system.

**Green** = simplify reports its fixes applied, with the resulting file list.

## Stage 4 — QA (smoke/UX only)

Invoke gstack `/qa-only` against the running app. It drives a real browser, walks the
changed flow end-to-end, checks console errors and interactive states, and returns a
health score, screenshots, and repro steps.

Use `/qa-only`, never `/qa`. `/qa` fixes bugs and commits them atomically, which
would land changes that Stage 5 never reviews and put two fixers on one diff.
`/qa-only` has no `Edit` tool at all, so it cannot. Carry its findings forward — they
enter Stage 5's ranked queue alongside the code-review findings.

This stage does **not** run the test gate — that belongs to Stage 5 via
`review-and-ship`; running it twice wastes tokens.

If the repo's CLAUDE.md defines a local stack close-out (for example a Docker
compose rebuild), follow it and report the real local URL. Any verification that
could touch real users or customers runs in the repo's test mode; never real
client effects.

**Green** = `/qa-only` ran to completion against the running app. Findings,
including a broken flow, do not make this stage red: they go to Stage 5 as input,
where the fix happens. **Red** = QA could not run at all (app will not start,
browser unavailable, timeout).

## Stage 5 — Review and ship

Invoke `review-and-ship` (parallel specialist review, `/codex` challenge, ranked
P1/P2/P3 findings, P1 fixes, re-review, test gate, gstack `/ship` which opens the
PR). The conductor runs no review pass of its own — that would duplicate the stage
skill. Hand Stage 4's QA findings in as additional input so they are ranked with
everything else instead of being fixed on a separate track.

**Re-QA condition**: if the P1 fixes touched UI or user-flow files, run `/qa-only`
once more on the fixed tree before `/ship`. Stage 4 exercised the pre-fix code, and
the test gate does not catch a broken flow — that is the whole reason QA exists.
Skip the second pass when the fixes were backend-only; this is conditional, not
routine. Exception: if Stage 4 found a broken flow, the second pass is mandatory and
must show that flow working before `/ship`.

**Green** = review-and-ship reports shipped: PR open, test gate passed, findings
resolved.

## Stage 6 — HARD STOP

The pipeline ends here. Report status and suggested next commands the user can run
themselves — do not run any of them:

- `merge it` → then, in a later turn, gstack `/land-and-deploy`, `/canary <url>`,
  `/document-release`, `/learn`.

**"Ship it" is not merge consent.** After Stage 5 opens the PR, only the literal word
**"merge"** (e.g. "merge it") advances past this stop. When it comes, merge with
`gh pr merge --merge` — never `--squash` or `--rebase`. This skill does not chain
into post-merge steps even after a merge instruction; those are separate, user-
invoked skills.

## Autonomous invocation contract (for /build-loop and other non-interactive callers)

When a caller explicitly invokes /build in **autonomous mode** (the /build-loop skill
does; a user can too by saying "autonomous /build"), the stages are unchanged but every
human checkpoint resolves by policy instead of waiting:

- **Inputs from the caller**: a spec (e.g. a PRD-derived ledger item), a base branch
  (default: main), and a PR target branch (default: base). Worktrees branch from the
  base; Stage 5 opens the PR against the PR target.
- **Stage 0's trivial shortcut is disabled** — every autonomous item runs the full
  pipeline, because the caller needs a PR and a machine-readable exit, which the
  trivial path never produces.
- **Stage 1 invokes `/plan` in autonomous mode** — see that skill's own Autonomous
  mode section. The supplied spec counts as the approved design, so brainstorming is
  skipped and `/autoplan`'s final gate is suppressed. If the spec is ambiguous about
  *what* to build (not how), exit `blocked: needs-human` immediately; never resolve
  product ambiguity by picking. Every auto-decision is persisted in the plan file
  under an `## Auto-decisions` section: the decision, the assumption behind it, the
  rejected alternative. Findings that change scope or architecture do NOT checkpoint —
  they exit `blocked: needs-human`. Stage 3.5 (`simplify`) and Stage 4 (`/qa-only`)
  run unchanged; neither needs a human.
- **No AskUserQuestion anywhere.** Every would-be checkpoint maps to exactly one of:
  auto-decide (minor, logged), `blocked: needs-human` (scope/architecture/product),
  or an icebox note in the report (out-of-scope discovery).
- **Machine-readable exit** — autonomous /build ends in exactly one state:
  - `ready-for-merge` — PR open against the PR target, test gate green, no unresolved
    P1 findings. (Stricter than interactive Stage 6: an open PR with red CI or open
    P1s is NOT ready-for-merge.)
  - `blocked: <reason>` — includes what interactive mode would call checkpoint or red.
  Either way the exit report carries: exit state, PR URL + head branch (if opened),
  plan and state file paths, follow-ups (each tagged with the requirement it traces
  to, or "untraced"), and for blocked, whether the cause reads transient or
  deterministic. Suppress the interactive next-step suggestions (`merge it` →
  `/land-and-deploy` …) — they are wrong for a non-main PR target.
- **Stage 6 invariant holds in every mode: /build never merges anything.** A caller
  may merge the PR it received; /build itself may not, regardless of target branch.

## Failure / abort policy

On red: stop, report exactly what blocks, and wait — no auto-retry loops. The
worktree is **always preserved**, never deleted on failure; report its path so work
is recoverable.

If **two consecutive stages** go red, declare the task mis-sized and propose
re-scoping rather than grinding further.

## Handoff schema (per stage)

Each stage returns a minimal structured summary the conductor carries forward:

- what changed (files)
- commands run
- verification evidence
- open questions

## Output format

**Per-stage status line** (one line, posted after each stage resolves — never raw
subagent chatter):

```
Stage <n> <name>: <green|checkpoint|red> — <one-line reason/result>
```

**Final report** (at Stage 6, or on red/mis-sized abort):

```
Build: <feature slug>
Worktree: <path>  Branch: <branch>
Stages: 0=<state> 1=<state> 3=<state> 3.5=<state> 4=<state> 5=<state>
PR: <url or "not yet">
State file: .context/build-<slug>.md
Next (user-invoked only): merge it -> /land-and-deploy -> /canary <url> -> /document-release -> /learn
```
