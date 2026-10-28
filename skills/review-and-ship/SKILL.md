---
name: review-and-ship
description: Runs a standard shipping loop — the simplify skill applied to the diff, then parallel Sonnet-subagent review (correctness/security, overengineering, adversarial production-failure, testing, and frontend/i18n when UI is touched) fanned out alongside gstack /codex, consolidated into ranked P1/P2/P3 findings, P1 fixes applied and re-reviewed, the repo test gate run, and gstack /ship invoked if everything is green. Use whenever the user says things like "review this branch and ship", "review and if green ship", "/review and if everything green /ship", "fix the findings then review again", "is it ready, ship it", "run review + codex", or invokes /review-and-ship. Runs its own reviewer fan-out and wraps gstack's /codex and /ship into one loop instead of running them ad hoc.
---

# Review and Ship

A recurring shipping loop, encoded so it runs the same way every time instead of being
re-improvised per session: parallel specialist review + an independent cross-model challenge from
codex, ranked findings, fixes, re-review, the test gate, then ship — only if everything is green.

## Why this shape

A single review pass misses things a second, targeted pass catches — that's why fixes always get
a dedicated re-review, not just a rerun of the original reviewers. And a green test suite is not
the same as intact production behavior: a past incident here was a safety gate
(`getattr(config, "test_mode", False)`) that could never fire because the field didn't exist —
tests were green, the guard was dead. So any safety/gating change gets an explicit trace, not just
a passing gate.

## Orchestration rule

You are the orchestrator only. Do not read diffs, run verbose greps, or fix code yourself — dispatch
all of that to Sonnet-model subagents and synthesize their summaries. This is a token-economy rule,
not a formality: an orchestrator that has read the whole diff itself is a worse orchestrator than one
that dispatched it. Launch every batch of independent agents (reviewers, or fixers for independent
findings) in a single message with multiple Agent calls — never sequentially.

## Workflow

1. **Scope the diff.** Identify the target branch/worktree and the merge-base against the
   base branch the caller supplied (for example `/build`'s PR target, which `/build-loop` sets
   to its integration branch). With no caller-supplied base, use `main` (or the repo's default
   branch). Carry that same branch through to step 9. Confirm with the user (briefly) which branch/diff you're
   reviewing if it's ambiguous — don't guess on a multi-branch or multi-worktree repo.

1.5. **Simplify first.** Invoke the `simplify` skill on that diff — it reviews the changed code for
   reuse, simplification, efficiency, and altitude and *applies* the fixes. Skip only if the caller
   already ran it on this exact diff (`/build` does, at its Stage 3.5); never skip because the
   overengineering reviewer in step 2 exists. That reviewer only flags, its findings land as P2, and
   P2s are optional in step 5 — which is exactly how overengineering survives a green review. This
   step is the one that deletes it, and running it here means the reviewers read the final code.

   Simplify is quality only; it does not hunt for bugs, so it never substitutes for step 2. It must
   not strip input validation, error handling, security controls, or accessibility.

2. **Fan out parallel reviewers.** Pick 3–5 Sonnet-model subagents based on what the diff actually
   touches, and launch them together in one message:
   - **Correctness + security** — logic bugs, injection, auth/authz gaps, unsafe defaults.
   - **Simplicity / overengineering** — apply this rubric verbatim: *correct, secure, most simple,
     most robust, maintainable, NOT overengineered*. Flag any new env var (the user rejects these on
     sight — stable values get hardcoded as constants instead), any file touched outside the actual
     feature/bug surface, and any speculative abstraction (interfaces/configs/flags with exactly one
     caller).
   - **Adversarial production-failure** — try to construct a realistic scenario where this change
     breaks production. Specifically: for every safety gate, flag, or guard touched in the diff,
     trace the value to its actual source and confirm it can fire. A `getattr(x, "field", default)`
     or similar fallback is a red flag if `field` is never actually set anywhere — that exact pattern
     has shipped a dead safety gate before.
   - **Testing / maintainability** — is there a regression test covering the changed behavior? Would
     a future editor understand why this code exists without spelunking git blame?
   - **Frontend / i18n / design** — only if UI files are touched. Check against the repo's UI style
     rules (commonly `.cursor/rules/ui-style-guide.mdc` and any design-baseline doc) and confirm
     user-facing copy goes through locale message files, not inline component literals.

   Each reviewer gets: the diff scope, its specific lens, and an instruction to return findings only
   (file:line, what's wrong, why it matters) — not a narrated walkthrough.

3. **Challenge with codex, in parallel with step 2.** Invoke the gstack `/codex` skill against the
   same diff scope as an independent cross-model review, in the same message batch as the reviewer
   agents where possible (or immediately alongside them) — it's a second opinion, not a follow-up.

4. **Consolidate.** Merge all findings, dedupe overlapping ones, and rank:
   - **P1** — must fix before shipping (bugs, security holes, dead/broken safety gates, breaks prod).
   - **P2** — should fix (maintainability, missing tests, moderate overengineering).
   - **P3** — note only, no action required now.

   Present the ranked list briefly — this is a status checkpoint, not the final report.

5. **Fix P1s (and cheap P2s).** Apply minimal, targeted diffs — no scope creep, no drive-by
   refactors. If the Sonnet agents that wrote the code are still alive in this session (e.g. this
   review runs as a pipeline stage right after implementation), send each finding back to the
   worker that owns those files via SendMessage — it already holds the context of what it built
   and why. Otherwise dispatch fresh Sonnet fixers. Independent fixes go in one parallel batch;
   dependent ones go sequentially.

6. **Re-review the fixes.** Dispatch a subagent per fix (or one subagent covering all fixes if
   they're small and related) whose only job is to verify each fix actually landed correctly and
   introduced nothing new. Never treat the original review pass as still valid after edits — a fix
   is not confirmed until it has its own dedicated check.

7. **Run the test gate.** Delegate to a subagent; ask for failures only, not a full log dump. Use
   the repo's standard test/CI command (check `CLAUDE.md`/`AGENTS.md`/`package.json` if unclear).

8. **If a safety/gating mechanism was touched anywhere in the diff**, explicitly state — with a
   file:line trace — whether real (non-test) production behavior is intact. A passing test gate is
   not sufficient evidence on its own for this point.

9. **Ship or stop.** If review findings are resolved and the test gate is green, invoke gstack
   `/ship` with the PR target branch from step 1. After the PR opens, confirm its base with
   `gh pr view --json baseRefName`; if it is wrong, re-target it with `gh pr edit --base` before
   reporting shipped. If anything is still red, report exactly what blocks and stop — do not ship partially
   green work.

## Standing constraints

- **Never auto-merge.** Merging happens only on the user's explicit word, and only as a normal merge
  commit (`gh pr merge --merge`) — never squash or rebase, even if recent history looks squashed.
- **Check worktree upstreams before pushing.** Branches in worktrees sometimes have upstream set to
  `origin/main`, so a bare `git push` can land on main. Always push with an explicit
  `<branch>:<branch>` refspec.
- **Token economy is not optional.** If a step can be answered from a subagent's summary, don't also
  read the raw diff yourself "just to check." Trust the delegated result; re-delegate a narrower
  question if something looks off rather than pulling the whole file into your own context.

## Output format

Keep the final report short and plain:

```
Findings: N P1, N P2, N P3 (ranked list, one line each)
Fixed: <what was fixed, one line each>
Re-review: <pass/fail per fix>
Test gate: <pass/fail, label>
Safety gates touched: <file:line trace, or "none">
Ship: <shipped (link) | blocked on: ...>
```
