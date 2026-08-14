---
name: duet
description: "Use when the user types /duet <task or PR> — autonomous Claude↔Codex loop: Claude designs & reviews, Codex implements, until both approve or the round budget (default 15) runs out"
---

# Duet — Claude designs and reviews, Codex implements

Run the whole design → agree → implement → review → approve cycle between the
orchestrating Claude model and Codex without the user shuttling messages.
The user is hands-off after kickoff.

**Announce at start:** "Starting duet: <task one-liner>. Mode: <spec|implement|PR
review>. <Opening line: 'Codex and I will draft independently, then I'll merge
the two' — or, in implement mode, 'I'll design first' — then loop with Codex.>
Budget: <B> rounds."

## Prerequisites

- **Codex CLI**, installed and authenticated (`codex login`), able to run
  `codex exec` non-interactively. Duet drives it once per round. If
  `codex --version` fails, stop and tell the user rather than improvising a
  substitute implementer.
- **Claude Code** as the orchestrator — this skill leans on its subagents,
  background Bash, and `ToolSearch`.
- Nothing else. Duet needs no other skills and no repo-specific setup; it
  works in any git repo (and, degraded, outside one — see **Set up**).

## Parse the invocation

- **Task**: everything after `/duet` that isn't a directive below.
- **Mode** — decides the opening shape (see **Opening** below) and whether
  there's an implementation phase:
  - **Spec** — the task is to design/plan something. Blind openings.
  - **Implement** — a plan/spec already exists and the job is to build it.
    No blind openings; the existing plan is the proposal.
  - **PR review** — the invocation points at a PR whose code was written by
    someone else and the ask is to review it. Blind openings, no
    implementation phase (see **PR-review mode** below).

  Ambiguous cases: if the task names an existing plan/spec/issue to follow,
  it's Implement; if it points at a PR without asking for changes to be
  written, it's PR review; otherwise Spec. State the chosen mode in the
  announcement and the notebook header. Don't ask the user.
- **PR medium**: if the invocation points at a PR (number/URL), the exchange
  happens in the PR thread instead of a notebook (see PR mode below), unless
  the user says otherwise. This is orthogonal to mode — a Spec or Implement
  duet can also run in a PR thread.
- **Medium override**: "keep the discussion in <X>" wins over defaults.
- **Budget override**: "budget 25" (or similar) sets the round budget for
  this run; otherwise it's 15. A typical task converges in ~10 rounds — the
  budget is headroom to catch runaway loops, not a target, so don't slow-walk
  toward it or stop early because rounds remain.
- **Tie-breaker**: default is pause-and-escalate at budget exhaustion. If the
  user says "figure it out" AND names a tie-breaker (e.g. "Fable decides"),
  that side settles disputes at the cap. If they say "figure it out" without
  naming one, ask who breaks ties NOW (AskUserQuestion), before starting.
  This is the only question allowed. **After kickoff the run is fully
  autonomous — never ask the user anything mid-run.**

## Set up

- In a git repo: create an isolated worktree at
  `~/.claude/duet/<repo>/worktrees/<slug>` on branch `duet/<slug>`
  (from `origin/HEAD` if a remote exists, else `HEAD`); per-turn prompts and
  logs in `~/.claude/duet/<repo>/runs/<slug>/`. PR mode: `git fetch origin
  pull/<n>/head` and check that out in the worktree. Not a git repo: work in
  place, skip the worktree.
- **Notebook**: `.duet/<YYYY-MM-DD>-<slug>.md` inside the worktree. Add
  `.duet/` to `.git/info/exclude` (never commit it unless asked). Header:
  task, mode, date, participants (ACTUAL model names), budget state.
- Read `references/protocol.md` (next to this file). It is the contract both
  sides follow; hand its full text to Codex with every turn prompt.

## Context economy — orchestrate rounds through fresh subagents (default)

The orchestrating context should stay a **thin dispatcher**; the heavy
per-round work runs in fresh subagents. This is the default shape, not an
optimization for long runs — it's rarely obvious ahead of time which runs
turn long, the runs dir + PR thread/notebook preserve everything anyway, and
a single long-lived orchestrator context costs more every round: the
transcript only grows, so round N re-reads everything rounds 1..N-1 produced.
Prompt caching softens that but doesn't remove it — cached reads still scale
with a transcript that keeps growing, a 60-minute Codex turn can outlast even
the 1h cache TTL, and the context eventually needs compaction regardless.
(Caching is handled by the harness, not by duet — there is no TTL knob here.)

- **State file**: at kickoff, create `<runs>/<slug>/architect-state.md` — the
  distilled working state: mode, protocol essentials + paths, verification bar,
  settled design rulings (do-not-relitigate list), delivery/checkpoint state,
  round accounting, open findings, and review-methodology notes (e.g. lessons
  about the implementer's failure modes). Whoever completes a round updates
  it. A fresh context needs only this file plus the latest notebook/PR
  entries to act as the architect.
- **Per-round agent**: run each design/review round as a *fresh*
  general-purpose subagent (never a fork — forks inherit the full transcript,
  defeating the purpose) prompted with: the state file path, the worktree and
  runs dir, and the round task. It re-runs the verification bar itself, fans
  out its own adversarial review agents for substantive diffs, drafts and
  posts the architect entry (notebook or `gh pr comment`), updates the state
  file, and returns only a compact summary + verdict.
- **Dispatcher keeps**: launching Codex turns, watching CI, round/budget
  accounting, tie-breaker decisions, and anything requiring the user. Its
  context should grow by roughly a paragraph per round.

## Opening — independent first (spec and PR-review modes)

Whoever writes first anchors the other. In **spec** and **PR-review** modes the
two sides open **blind**: each produces its opening without having seen the
other's, and a third, fresh context merges them. Skip this entirely in
**implement** mode — the plan already exists, so there's nothing to converge on;
go straight to the loop with the existing plan as the proposal.

Blindness is the whole point and the notebook/PR thread is shared, so the drafts
stay **out of the shared surface** until both exist.

1. **Dispatch both in parallel.** Launch the Codex turn in the background
   (Bash `run_in_background`) and the Claude draft at the same time; this costs
   no extra wall-clock over doing one.
   - **Codex**: prompt says explicitly *do not read the notebook / do not read
     the PR thread — no entry has been written yet and you are drafting
     independently*; write the draft to
     `<runs>/round-1-codex-open.md` and nowhere else.
   - **Claude**: a *fresh* general-purpose subagent (never a fork — a fork
     inherits this transcript) writes to `<runs>/round-1-claude-open.md`.
     Give it the task, worktree, and state file — not any opinion of yours.
2. **Merge.** A *fresh* subagent — no `model` override, so it inherits whichever
   Claude model is orchestrating; fresh matters more than which model, since a
   context that wrote one of the drafts can't merge impartially — reads both
   drafts and produces the merged proposal. It must: take the stronger option
   point by point rather than splitting differences, name every place the two
   disagreed and say which it took and why, and flag anything only one side
   raised. Where they genuinely conflict and neither is clearly better, it says
   so and picks one, marking it open for the loop to settle.
3. **Post all three together** as Round 1 — both original drafts verbatim
   (clearly labeled and signed by their actual authors) plus the merged
   proposal, `VERDICT: PROPOSE`, signed by the merging model. Posting the
   originals keeps the record honest about what each side actually thought.
4. Round 2 opens the normal loop, with both sides critiquing the *merger*.

**Budget:** the blind pair plus the merge together count as **Round 1** — one
round, not three.

## The loop

One **round** = one orchestrator entry + one Codex entry. **The budget (B,
default 15) covers all phases together** — design and review draw from the
same pool. Record B in the notebook header and label every entry
`Round N/<B>`.

1. **Design**: in spec and PR-review modes, Round 1 is the blind opening above
   and the loop picks up at Round 2. In implement mode, write the initial
   design in the notebook, `VERDICT: PROPOSE`. Either way, sign as the model
   actually running (if Opus is orchestrating, sign Opus — verify identity,
   don't assume).
2. **Codex turn**: write a prompt file in the runs dir — it states the round
   number, phase, absolute paths of notebook + repo, includes the protocol
   text verbatim, and says "read the notebook, append your entry to it".
   Then run:

   ```bash
   env -u CODEX_API_KEY -u OPENAI_API_KEY \
     codex exec --dangerously-bypass-approvals-and-sandbox \
     -C "<worktree>" -o "<runs>/round-<n>-codex.md" \
     "$(cat "<runs>/round-<n>-prompt.md")" </dev/null
   ```

   **On the `env -u` prefix — keep it or drop it, once, at setup.** It strips
   inherited API keys (and defeats shell wrappers that inject one), so Codex
   falls back to whatever `codex login` stored. Keep it if Codex is signed in
   with a subscription and you'd rather not bill the API for long
   implementation runs — that is the default assumption here. **Drop it if
   your Codex authenticates by API key only**, since with the keys stripped
   `codex exec` will fail to authenticate. If the first Codex turn dies on an
   auth error, that prefix is the first thing to check.

   The `</dev/null` is required: with an open non-tty stdin, `codex exec`
   blocks forever waiting for stdin to close (verified 2026-08-11).

   Verify Codex appended a well-formed entry (`grep '^VERDICT:'` on the
   tail). If it answered in `-o` output but failed to append, append it
   yourself, marked "(appended by orchestrator on Codex's behalf)".
3. Alternate until both sides have said `VERDICT: AGREE` on the design.
   Respond point-by-point; concede when Codex is right.
4. **Implement**: Codex codes in the worktree in an exec run (its half of a
   round). Prompt includes the agreed design, repo ground rules
   (AGENTS.md/CLAUDE.md), validation gates (lint/tests), "never leave the
   worktree, never `git stash`". Long runs: Bash `run_in_background`, then
   monitor; don't let a 10-minute foreground timeout kill it.
5. **Review**: review the full diff at the standard of the code-review skill
   at high effort — for substantive diffs, multiple independent review
   angles with findings adversarially verified (including empirical checks)
   before posting; review judgment stays with the orchestrator. Findings go
   in the notebook, `VERDICT: REVISE` (concrete, file:line, most severe
   first) or `VERDICT: APPROVE`. Codex must answer each finding with its own
   judgment (confirmed / disputed and why), and the next orchestrator turn
   evaluates those judgment calls — don't accept "fixed" without
   re-verifying against the new commit. Repeat. **Done when the orchestrator writes APPROVE and Codex's closing
   entry is AGREE.**
- Mechanical chores (collecting logs, running test suites, summarizing long
  output) may go to lower-cost subagents (haiku); design and review judgment
  may not.

## Finish

- Commit in the worktree; push/PR only if the task calls for it (PR mode:
  push to the PR branch, post a wrap-up comment).
- Final report to the user in plain language: what was built, how it was
  verified, notebook path, worktree path, diff summary.
- **Tidy up.** Only after the final report, and only once the work is
  committed (or the user asked to throw it away). Duet created the worktree,
  so it already knows both paths — do not go discovering them:

  ```bash
  cp -r "<worktree>/.duet" "<runs>/<slug>/notebook"
  git -C "<main-repo>" worktree remove "<worktree>"
  ```

  The notebook lives inside the worktree and is untracked by design (excluded
  so it's never committed), so `git worktree remove` deletes it silently
  along with the checkout — copy `.duet/` into the runs dir first or it's
  gone for good. The branch `duet/<slug>` and every commit on it survive;
  only the checkout folder goes away. If the command refuses because of
  uncommitted changes, say so in the final report and leave the worktree
  alone — add `--force` only if the user explicitly asks to discard that
  work. If duet worked in place (not a git repo), there is nothing to clean
  up.
- Load PushNotification (`ToolSearch "select:PushNotification"`) and ping the
  user that it's done. Same on any early stop.

## Budget exhausted

- **No tie-breaker named**: stop. Write "Where we're stuck" — in the
  notebook and as the final message — in words a 10-year-old could follow:
  what both sides agree on, what they're stuck on, what each side wants. No
  jargon, as brief as clarity allows. Ping the user.
- **Tie-breaker named**: that side decides each open dispute (documented in
  the notebook), then up to 3 extra rounds to finish under those decisions.
  Still not done → stop and summarize as above.

## PR-review mode

Reviewing a PR somebody else wrote: there is no implementation phase, and the
deliverable is the review itself.

- **Set up** as PR mode below (fetch and check out the PR head in the
  worktree), but nobody commits — treat the checkout as read-only.
- **Round 1** is the blind opening above: both sides review the PR
  independently, each at the standard of the code-review skill at high effort,
  each fanning out its own review angles and adversarially verifying findings
  before writing. Neither reads the PR thread while drafting.
- **The merge** dedupes findings that both sides found (noting agreement —
  that's signal), keeps findings only one side raised, and orders everything
  most-severe first. Findings the two sides *contradict* each other on go in
  marked as disputed, not silently dropped.
- **Rounds 2+** settle the disputed findings and anything either side wants to
  add: each responds point-by-point with its own judgment (confirmed /
  refuted and why). Verify contested claims against the actual code rather
  than deferring to whoever sounds more confident.
- **Done** when both sides `AGREE` the finding list is right — no `APPROVE`,
  since nobody here is approving code. Post the settled review as the wrap-up
  comment, and say plainly which findings were unanimous and which were one
  side's call.
- **Budget exhausted** with findings still disputed: post the agreed findings
  and list the disputed ones as disputed, showing both positions. A split
  review is a fine outcome; a fake consensus is not.

## PR mode

- The PR thread replaces the notebook: every entry is a `gh pr comment`,
  opening with an attribution line per the user's global GitHub rules, e.g.
  `*[Fable 5 · Claude Code · duet orchestrator · <org>/<repo>:]*` — Codex
  posts its own comments via `gh` and attributes likewise (the turn prompt
  must spell this out). Verdict + signature rules unchanged.

## Integration

**Self-contained by design.** Duet depends on no other skill. Worktree setup
and teardown are spelled out in full above (**Set up**, **Finish**) — do not
reach for other skills or repo docs to create or retire the worktree, and do
not assume any exist on this machine.
