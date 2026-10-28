---
name: build-loop
description: >
  Autonomous PRD executor: decomposes a big PRD into a dependency-ordered ledger and
  drives it to completion by running one autonomous /build per /loop tick, merging each
  finished sub-plan into a dedicated integration branch, and stopping at a single final
  human-merged PR. Trigger on "/build-loop <prd file>", "run this PRD to completion",
  "execute the whole PRD". Do NOT trigger for a single feature (route to /build) or for
  recurring non-build tasks (plain /loop).
---

# /build-loop — PRD-to-done orchestrator

One orchestrator (the main thread), one ledger, one autonomous /build at a time.
No nested orchestrators: never dispatch /build inside a subagent. /build's own
Sonnet workers are the only agent layer below this skill.

Invoke as `/build-loop <prd-path>` — pacing is self-armed: at the end of the setup
tick, invoke the `loop` skill with `/build-loop <prd-path>` as its prompt (dynamic
self-paced mode, no interval), so every subsequent tick fires via ScheduleWakeup
without the user typing anything. If the session is already in loop mode (a wakeup
re-invoked this skill), don't re-arm — just run the tick and schedule the next
wakeup. `/build-loop <prd-path> supervised` skips the arming and runs ticks
back-to-back in the current turn instead. Each tick relies on /build's
**Autonomous invocation contract** (see the /build skill) — spec in,
`ready-for-merge` or `blocked: <reason>` out, no questions.

## Branch model

- On first run, create the **integration branch** `build-loop/<prd-slug>` from main.
  If that branch (local or remote) or an open final PR for the slug already exists,
  do NOT reuse it silently — stop and ask the user: resume that run, or pick a new
  slug.
- Each item's /build worktree branches from **fetched `origin/build-loop/<slug>`**
  (fetch first — after every `gh pr merge` the remote is ahead of any local ref).
  Item branches are named `bl-item/<slug>/BL-<i>` (a separate namespace —
  `build-loop/<slug>/BL-<i>` is impossible in git because the integration branch ref
  is its prefix); PR titles start with `[BL-<i>]`
  — this naming is what lets reconciliation map PRs back to ledger items.
- Sub-plan PRs target the integration branch. When an item is `ready-for-merge`, the
  LOOP merges its PR with `gh pr merge --merge` (never squash/rebase). **Before
  merging, verify via `gh pr view` that the PR's base branch is exactly
  `build-loop/<slug>`, it is open and mergeable, and checks are green** — never
  trust that /build targeted the right base; a mis-based PR aimed at main must be
  re-targeted, never merged. If the merge command fails (conflict, protection,
  stale head), the item becomes `blocked: merge-failed` — never `merged`.
- **Bind the merge to the reviewed commit and confirm it landed.** Read the PR's head
  SHA (`gh pr view --json headRefOid`) in the same check that confirms green CI,
  merge with
  `gh pr merge --merge --match-head-commit <sha>`, then confirm with
  `gh pr view --json state,mergeCommit` that `state` is `MERGED` and that the merge
  commit is reachable from fetched `origin/build-loop/<slug>`. A queued or pending
  merge (merge queue, auto-merge) is not merged: leave the item `building`, keep
  its worktree, and re-check on the next tick. Never mark `merged`, clean up, or
  start a dependent item until both checks pass.
- The ONLY PR to main is the final integration PR, opened at completion and
  **never merged by the loop** — human merges. The never-auto-merge-to-main rule is
  absolute in every state, including "all items done".

## Ledger — `.context/prd-<slug>.md` (in the primary checkout, not a worktree)

Single source of truth across ticks and compaction. Sections:

1. **Header**: PRD path, PRD content hash (`shasum -a 256`), integration branch,
   `paused: false`, tick count / `max_ticks` (default 25, counts only item-build
   ticks — setup, merges, and finalization are always allowed), final PR URL (once
   it exists), start date.
2. **Items** — one per sub-plan, stable IDs `BL-1…` (also used for integration-fix
   items — no special ID scheme):
   `| id | title | deps | status | build state file | PR | note |`
   Status: `pending / building / ready-for-merge / merged / blocked / dropped`.
   `deps` is canonical ordering authority; row order is only a tiebreak among
   runnable items. Follow-up items record `follow-up of BL-<i>` in `note`; an item
   whose note already says follow-up may not spawn further items (its discoveries
   go to the icebox).
3. **Item specs** — one short section per item: what to build, acceptance criteria,
   and the PRD requirement IDs it covers. This section (plus the PRD itself) is the
   spec handed to autonomous /build; a bare title is not a spec.
4. **Coverage matrix**: every PRD requirement / acceptance criterion → ≥1 item ID.
   Requirements marked `[not covered — why]` force `paused: true` at end of setup —
   the user must waive or cover them before the loop proceeds.
5. **Icebox**: discoveries that don't trace to a PRD requirement. Never built by
   this loop.
6. **Log**: one line per tick (tick #, item, outcome).

The user may hand-edit the ledger between ticks: fix a spec, mark an item `dropped`
(its ID is removed from dependents' `deps` and its coverage rows need a new owner or
a waiver), or flip `paused`. After handling a PRD-change pause, the user (or the
resuming tick, on their explicit go-ahead) updates the stored hash to the new PRD
hash — otherwise every restart re-pauses on the same mismatch.

## First invocation (setup tick)

1. Read the PRD. Decompose into items sized for one /build each (independent,
   coherent scope, explicit deps), writing an item spec section for every item.
   If the PRD came out of a `/wayfinder` map, that map's decisions are settled input
   — read them, but decompose fresh here. The ledger is the only build-item list;
   never mirror tracker tickets into it, or the two drift.
2. Write the ledger with the coverage matrix and PRD hash. Create the integration
   branch and push it. Report the decomposition to the user.
3. Schedule the first work wakeup at ~15 minutes, saying so — that window is the
   user's real chance to correct the split before token spend starts. (Uncovered
   requirements instead pause the loop entirely, per the coverage rule.)

## Tick procedure (every subsequent wakeup)

1. **Read the ledger.** If `paused: true` or the user said stop → end the loop
   (ScheduleWakeup stop) with a status report. Sanity-check the ledger (unknown
   statuses, dep cycles, deps on dropped items, duplicate PR references, more than
   one `building` item) — on corruption, pause and report rather than acting.
2. **Reconcile before acting** (this replaces locks/idempotency machinery — the
   session is single-threaded, so drift only comes from crashes or human action):
   - `gh pr list --state all` for `[BL-*]` PRs on the integration base, plus the
     final PR if recorded. A PR merged on GitHub while its item isn't `merged` →
     mark it `merged` after confirming its merge commit is reachable from
     `origin/build-loop/<slug>` (never rerun /build for it). A PR closed without
     merge → `blocked: needs-rework`.
   - `git worktree list` vs items marked `building` (a `building` item with no
     worktree and no state-file progress → back to `pending`).
   - Re-hash the PRD. Changed → set `paused: true`, write a delta report (added /
     changed / removed / already-built requirements), stop the loop, wait for user.
3. **Finish before starting**: if an item is `ready-for-merge`, re-verify and merge
   it now (per Branch model) before anything else. If an item is `building`,
   resume THAT /build from its state file. Never start a second item while one is
   `building`. A resumed or merged item still flows through steps 6–8 (result
   handling, cleanup, follow-ups, tick close) — resuming doesn't skip them.
4. **Select**: if all items are `merged` (or `dropped`) → **completion** (below;
   finalization never counts against the budget). Otherwise, if `max_ticks` is
   exhausted → end state `paused-budget`: stop and report; never present budget
   exhaustion as completion. Otherwise pick among `pending` items whose deps are
   all `merged` (deps naming `dropped` items were already pruned), taking the
   earliest row. If none remain runnable → end state `incomplete-blocked`: stop
   the loop, report what's blocked on what.
5. **Mark `building` and update the ledger BEFORE invoking /build.** Then run one
   autonomous /build on the item (spec = item spec section + PRD; base and PR
   target = integration branch; worktree per item, branch `bl-item/<slug>/BL-<i>`).
6. **On `ready-for-merge`**: merge and confirm (per Branch model), mark `merged`
   only once confirmed, then clean up:
   copy the item's /build state file and plan into `.context/archive/` in the
   primary checkout (update the ledger pointer), `docker compose down` any compose
   project the build started in that worktree, remove the worktree and item branch.
   **On `blocked`**: apply the failure policy below.
7. **Route follow-ups** from the /build exit report: traces to a PRD requirement →
   append as a new item (ID, spec section, deps, `note: follow-up of BL-<i>`);
   otherwise → icebox.
8. **Close the tick**: update the ledger (status, log line, tick count), post one
   status line to the user
   (`Tick <n>: BL-<i> <title> — <merged|blocked: reason>; <k>/<total> merged`),
   then schedule the next wakeup (~60s — human input isn't needed between items).

## Failure policy

- `blocked` with a transient cause (network, rate limit, flaky infra) → set back to
  `pending` with `note: retry 1`; one retry only, then it blocks for good.
  Deterministic causes (needs-human, failing design, unresolvable P1) block
  immediately.
- Dependent blocking is **derived, never persisted**: an item is unrunnable while
  any dep is not `merged`; it becomes runnable the moment the dep merges. Don't
  write `waiting-on` statuses into the ledger.
- If blocked items gate more than half of the remaining work, stop the loop now
  (`incomplete-blocked`) instead of grinding the unblocked tail.
- Blocked is a resting state: the user unblocks by editing the ledger (answering
  the need, re-scoping, or dropping the item) and restarting the loop.
- Blocked items keep their worktrees (recoverable work); the final report lists
  every leftover worktree and branch for a user-approved sweep.

## Completion

All items `merged` or `dropped`:

1. Fetch and fast-forward a local checkout to `origin/build-loop/<slug>` (this
   exact tree is what gets tested AND shipped); if main has advanced, merge main
   into it, resolve, and **push** the result before anything else (a conflicted
   final PR is not "complete").
2. Run the repo test gate on that integration checkout, plus a gstack `/qa-only` pass
   against the coverage matrix's acceptance criteria (per-item gates ran on pre-merge
   states; this catches cross-item breaks). `/qa-only`, not `/qa` — an autonomous
   loop must not let a QA pass commit unreviewed fixes at the final gate. Red → open
   a normal new `BL-<next>` fix item with a spec and keep looping.
3. Open the final PR `build-loop/<prd-slug>` → main (or confirm the recorded one is
   open and mergeable) and wait for its CI to go green before declaring the end
   state — red or pending final-PR checks are not `complete`. Record its URL in
   the ledger header. Do not merge it.
4. End state: `complete` (or `complete-with-icebox` if the icebox is non-empty —
   list it). Stop the loop. Final report: ledger table, final PR URL, icebox,
   leftover worktrees, total ticks.

## Kill switch

At any time the user can say "pause the loop" or set `paused: true` in the ledger.
Both take effect at the next tick boundary; a mid-tick /build finishes its current
stage and writes its state file first, so nothing is lost.

<!-- ponytail: no leases, idempotency keys, or schema versioning. The only SHA
     tracking is the merge's --match-head-commit check; single-threaded session +
     step-2 reconciliation covers crash recovery. Add a real state controller only if ticks ever run in parallel
     sessions. -->
