---
name: plan
description: >
  Thin conductor for the planning half of the pipeline: clears fog with /wayfinder
  when the destination is unclear, explores with superpowers:brainstorming when the
  requirements are ambiguous, drafts a ticket-shaped plan that carries decisions
  rather than code, then runs it through gstack /autoplan's CEO + eng + design + DX
  gauntlet. Emits one approved plan file and stops, it never implements. Trigger on
  "plan <feature>", "plan this out", "write a plan for X", "get this ready to build",
  /plan, and whenever /build reaches its planning stage with no plan file supplied.
  Do NOT trigger for: reviewing an existing plan (that is /autoplan alone) or for
  building (that is /build).
---

# /plan — planning pipeline conductor

This skill **invokes** other skills, `superpowers:brainstorming` and gstack
`/autoplan`, and consumes their summaries. It drafts the plan itself, in the format
below. Its single output is an approved plan file that `/build <path>` consumes
directly.

Planning lives here and only here. `/build` delegates its planning stage to this
skill rather than repeating it, so there is exactly one place planning behaviour can
drift.

Main thread conducts only. Drafting a plan is thinking, not bulk reading: dispatch
codebase reconnaissance to subagents and keep their summaries, don't read the
codebase yourself.

**This skill never writes implementation code.** It ends at an approved plan.

## Stage 0 — Is a plan warranted?

If the change is one file, one sentence, and touches none of `/build`'s risk surfaces
(billing, auth/entitlements, publish paths, schema/migrations, client-facing
production, deploy/infra), say so and point at `/build` directly. A plan for a
one-line change is the overhead this pipeline exists to avoid.

## Stage 1 — Clear the fog (conditional, user-run)

Judge whether the **destination** is clear, not just the route. If you cannot state
in one or two lines what reaching the end looks like, the work is foggy.

Foggy → recommend `/wayfinder` and **stop**. It charts the open decisions as grilling
tickets on the issue tracker and resolves them one at a time with the user, using
`/grilling` and `/domain-modeling`. Its own frontier step stops and says so when it
finds no fog, so the recommendation is cheap even when you are unsure.

`/wayfinder` is `disable-model-invocation: true` — **this skill cannot chain into
it.** The user runs it, then re-enters `/plan` with the decisions settled. Do not
attempt to invoke it; do not paraphrase its grilling yourself. An agent that answers
its own grilling questions has defeated the point.

Clear destination → skip to Stage 2.

## Stage 2 — Brainstorm (conditional, checkpoint)

Invoke `superpowers:brainstorming` when the requirements are ambiguous or the user
asked to explore. Skip it when the user handed over a clear spec or a PRD. That skill
gates on written user approval of the design; treat the wait as a checkpoint, not a
failure.

Fog and ambiguity are different problems. Stage 1 handles not knowing the
destination. This stage handles knowing the destination but not the shape of the
thing that gets you there.

## Stage 3 — Draft

Write the plan yourself, to `.context/build-<feature-slug>-plan.md` in the primary
checkout. That is the same path `/build` reads, so the handoff needs no translation.

Feed in whatever upstream context exists: the wayfinder map's settled decisions, a
PRD, the user's spec, the brainstorm's approved design. Record which of those the
plan came from, so a later reader knows what was decided and what was assumed.

A plan carries decisions, not code. The implementer is a capable agent with the
repository open in front of it, so it can find the files and write the tests itself.
What it cannot recover on its own is what you decided and why you rejected the
alternative.

**Do not invoke `superpowers:writing-plans`.** It requires a code block for every
step and full test bodies inline, which is what turns a plan file into a wall of
stale code that nobody rereads.

### Format

A short header, then one section per vertical slice.

```markdown
# <Feature> plan

**Goal:** one sentence on what this builds.
**Source:** wayfinder decisions / PRD / brainstorm / user spec, and which parts of
this plan were assumed rather than decided.
**Constraints:** project-wide rules with exact values, one line each.
**Out of scope:** what this deliberately does not do.

## Ticket N: <title>

**Delivers:** the end-to-end behaviour this makes work, in the user's terms.
**Blocked by:** ticket numbers, or "none".

- [ ] Acceptance criterion
- [ ] Acceptance criterion
```

### Rules

- Each ticket cuts a narrow but complete path through every layer it touches and is
  demoable or verifiable on its own. Never one layer at a time.
- Size each ticket to fit a single fresh context window.
- No code blocks, no file paths, no line numbers. One exception: a snippet that
  encodes a decision prose cannot state precisely, such as a schema, a state machine,
  or a type shape. Trim it to the decision.
- Acceptance criteria describe observable behaviour, not implementation steps.
- Blocking edges name only tickets that genuinely gate. A ticket with none starts now.
- For a mechanical change whose blast radius crosses the whole repository, sequence it
  as expand, then migrate in batches, then contract, instead of forcing it into one
  slice.

### Self-review

Read the draft once against its source. Is every requirement covered by a ticket, is
any acceptance criterion vague enough to be read two ways, does any ticket depend on
a ticket that does not gate it. Fix inline and move on.

## Stage 4 — Review gauntlet

Run gstack `/autoplan` against that path. It reads the CEO, eng, design, and DX
review skills from disk, runs them sequentially with auto-decisions by its 6
principles, and surfaces only taste calls at a final approval gate. It writes its
report back into the plan file and keeps a restore point.

`/autoplan` reviews a plan; it never authors one. Stage 3 is therefore not optional
and not skippable by handing it an empty file.

**Severity mapping**: minor findings fold into the plan silently. Anything that
changes scope or architecture goes to the user as a checkpoint before proceeding.

## Stage 5 — HARD STOP

Report the approved plan file path and stop. Do not create a worktree, do not
dispatch implementers, do not edit source. The next command is the user's to run:

```
/build .context/build-<slug>-plan.md
```

**When `/build` is the caller**, hand the path back and let it continue. The stop
applies to a user-invoked run, not to the delegated one.

For work too large for one `/build` — a full PRD's worth — the next command is
`/build-loop <prd-path>` instead, which decomposes into its own ledger.

## Autonomous mode

When `/build` invokes this skill in autonomous mode, the stages are unchanged but no
stage waits for a human:

- Stage 0 is skipped. Autonomous `/build` disables its trivial shortcut, so every
  autonomous item gets a plan here and never bounces back.
- Stage 1 does not recommend `/wayfinder`. Fog about *what* to build exits
  `blocked: needs-human` instead.
- Stage 2 is skipped. The supplied spec counts as the approved design.
- Stage 4 runs `/autoplan` with its final approval gate suppressed, deciding by its 6
  principles. Record every auto-decision in the plan file under an `## Auto-decisions`
  heading: the decision, the assumption behind it, the alternative rejected.
- A finding that changes scope or architecture exits `blocked: needs-human`. Never
  resolve product ambiguity by picking.

## Output format

```
Plan: <feature slug>
Fog: <cleared via /wayfinder | none found>
Brainstorm: <run and approved | skipped, spec was clear>
Plan file: .context/build-<slug>-plan.md
Gauntlet: <autoplan verdict — open majors, or none>
Next (user-invoked only): /build .context/build-<slug>-plan.md
```
