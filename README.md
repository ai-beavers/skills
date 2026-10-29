# skills

The Claude Code skills I use every day (vlady)

My rule: if you do something three times, make it a skill. Most of mine come from other people. I wrote a few myself, and those mostly chain the open source ones together.

## Quick start

You need about ten minutes.

1. Install [gstack](https://github.com/garrytan/gstack) and [superpowers](https://github.com/obra/superpowers). `/build` calls their skills by name.
2. Install the Codex command line tool with `npm install -g @openai/codex`, then run `codex login`. gstack's `/codex` review needs it.
3. Sign in to GitHub with `gh auth login`. `/build` opens pull requests through `gh`.
4. Copy `skills/build`, `skills/plan` and `skills/review-and-ship` into `~/.claude/skills/`. Add `skills/build-loop` if you want the loop.
5. Open a repo whose `CLAUDE.md` names its test command, start your app locally, and type `/build` followed by a small feature.

[ponytail](https://github.com/DietrichGebert/ponytail) and [mattpocock/skills](https://github.com/mattpocock/skills) are optional but worth it. `/build` passes ponytail's rules to every worker anyway, and `/plan` points you to Matt Pocock's `/wayfinder` when you don't know yet what to build.

## /build: one feature, end to end

I brainstorm with Claude until the feature is clear. Then I type `/build` and describe it. It asks a few questions and runs on its own until a pull request is open. It never merges. I always merge myself.

`/build` does almost no work itself. It calls other skills in order and checks the result after each step:

| Step | Runs | Result |
|---|---|---|
| Plan | my `/plan`, which calls superpowers `brainstorming` and gstack `/autoplan` | a reviewed plan file |
| Build | Sonnet subagents in their own git worktree, with ponytail rules | small code, written in parallel |
| Simplify | `simplify`, built into Claude Code | the new code, cut down |
| QA | gstack `/qa-only` against your running app | a click-through in a real browser |
| Review and ship | my `/review-and-ship` | reviews, fixes, tests, an open pull request |

Every step ends one of three ways. Green moves on. Checkpoint waits for me. Red stops and reports, with no retry loop.

A change can skip the pipeline when it touches one file, fits in one sentence, and stays away from billing, auth, database migrations, production and deploy files. Then `/build` edits directly, runs `simplify` and `/qa-only`, and offers `/ship`. That path has no planning and no specialist review.

Only the word "merge" merges. "Ship it" does not count.

## /build-loop: a whole product spec

`/build-loop <prd.md>` takes a product requirements document and runs it to the end.

1. It splits the document into small items, orders them by dependency, and writes a ledger file.
2. It checks that every requirement belongs to an item. A requirement with no item pauses the run.
3. It waits about 15 minutes, so you can fix the split before work starts.
4. On each tick of Claude Code's `/loop`, it picks the next item and runs `/build` in autonomous mode.
5. It merges each finished item into a separate integration branch, never into main.
6. At the end it runs the tests and `/qa-only` against the acceptance criteria. Then it opens one final pull request to main and stops.

Say "pause the loop" to stop it. By default it stops after 25 item builds.

The loop expects a `main` branch, a remote called `origin`, and merge commits allowed on GitHub. A repo that only allows squash merges will fail at the first merge.

## The skills underneath

`/plan` works out what to build, drafts a plan, and runs it through gstack `/autoplan`. It writes one approved plan file and never writes code.

`/review-and-ship` runs three to five Sonnet reviewers in parallel, plus gstack `/codex` for a second opinion from another model. It ranks the findings, fixes the serious ones, re-reviews each fix, runs your tests, and calls gstack `/ship`.

## /delegate: orchestration on demand

`/delegate` is separate from `/build`. Use it for any big task, code or not. The main session plans and hands out work, Sonnet workers do it in parallel, and the strongest model you have gets called at most once per run, at the hardest moment.

Copy `skills/delegate` into `~/.claude/skills/` and both files in `agents/` into `~/.claude/agents/`.

## How the skills stack

```
/build-loop  (one tick of /loop per item)
   |
   v
/build
   |- /plan ............ superpowers brainstorming, gstack /autoplan
   |- Sonnet workers ... ponytail rules, own git worktree
   |- simplify
   |- gstack /qa-only
   '- /review-and-ship . parallel reviewers, gstack /codex, gstack /ship
   |
   v
pull request, waiting for you to say "merge"
```

Building this taught me one thing: write a thin skill that calls the good skills other people already wrote. One giant skill drifts the moment its parts change.

## Other people's skills I use

Install each one from its own repo. The authors keep their install steps current, so I link instead of copying them.

| Skill or pack | Author | What I use it for |
|---|---|---|
| [mattpocock/skills](https://github.com/mattpocock/skills) | Matt Pocock | `grill-me` and `grilling` ask me hard questions until a plan holds. `tdd` writes the test first. `to-spec` turns a chat into a spec. `wayfinder` maps the open decisions when I don't know what to build. |
| [gstack](https://github.com/garrytan/gstack) | Garry Tan | `/autoplan` reviews a plan from the CEO, engineering, design and developer-experience angles. `/qa-only` tests in a real browser, `/codex` asks another model, and `/ship` opens the pull request. I also use `/office-hours` and `/investigate`. |
| [superpowers](https://github.com/obra/superpowers) | Jesse Vincent | `brainstorming` before any new feature. It also makes Claude check for a fitting skill before every task. |
| [ponytail](https://github.com/DietrichGebert/ponytail) | Dietrich Gebert | Makes the agent write the smallest code that works. |
| [skill-creator](https://github.com/anthropics/claude-plugins-official) | Anthropic | Writes and tests new skills. I use it whenever a task hits the three-times rule. |
| [stop-slop](https://github.com/hardikpandya/stop-slop) | Hardik Pandya | Removes the usual AI writing tells from anything I publish. |
| [humanizer](https://github.com/blader/humanizer) | Siqi Chen | Runs after stop-slop and rewrites what still sounds like a chatbot. This README went through both. |
| [i-have-adhd](https://github.com/ayghri/i-have-adhd) | Ayoub Ghriss | Makes Claude lead with the next action and keep answers short. |
| [context-mode](https://github.com/mksglu/context-mode) | Mert Köseoğlu | Keeps long tool output out of the chat, so long sessions stay sharp. It uses the Elastic License 2.0, which is source-available but not open source. |
| [GSD Core](https://github.com/open-gsd/gsd-core) | Open GSD | A spec-driven system for bigger projects with phases and milestones. I run the older get-shit-done release, which now lives on as GSD Core. |

## License

My skills in `skills/` and `agents/` use the MIT license. See [LICENSE](LICENSE). Every project in the table above keeps its own license.

Made by [AI BEAVERS](https://ai-beavers.com).
