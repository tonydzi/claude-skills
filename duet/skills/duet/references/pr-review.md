# PR-review mode

Reviewing a PR: the two sides review independently, converge on what's wrong,
and then — in the default **review-then-fix** shape — Codex actually fixes it
and a fresh Sonnet conformance check verifies the fixes against what was
agreed. Agreeing on a list of complaints is the *middle* of this mode, not the
end.

## Phase A — converge on the findings

- **Set up** as **PR mode** below (fetch and check out the PR head in the
  worktree). Nobody commits during Phase A; the checkout is read-only until
  the work order is frozen.
- **Round 1** is the blind opening: a fresh **Fable** subagent is the
  Claude-side reviewer — this is the mode's one Fable checkpoint (see
  SKILL.md's model table), because Fable's *independent* read finds what a
  read anchored on a converged list would miss. Codex reviews independently
  too. Both review at the standard of the code-review skill at high effort,
  each fanning out its own review angles and adversarially verifying findings
  before writing. Neither reads the PR thread while drafting.
- **Merge iterations** (protocol.md's blind-opening/merge-iteration rules
  apply, operating on finding lists instead of plans): per iteration
  i = 1, 2, 3, a fresh **Sonnet** agent merges the two current finding lists
  into A′ᵢ; **Codex** produces B′ᵢ blind to A′ᵢ until both exist. A **Haiku**
  agent builds a mechanical comparison matrix (agreements, substantive
  disagreements, one-sided findings — no judgment calls; mark unsure cases
  substantive). **Converged** = no substantive disagreements in the matrix;
  the more complete list becomes the finding list (dispatcher picks, noting
  the choice). Findings both sides found are deduped, noting the agreement as
  signal; findings only one side raised are kept; everything is ordered
  most-severe-first. Findings the two sides *contradict* each other on are
  marked disputed, not silently dropped.
- **Cap: 3 merge iterations.** There is **no Fable end-check in PR review**
  (see SKILL.md's model table — the checkpoint here is the blind read, not a
  converged-list adjudication). Whatever's still disputed when the cap hits
  is carried straight into Phase B's **still disputed** bucket below.
- All drafts and mergers post to the PR thread verbatim, signed, once each
  pair exists — same honesty rule as everywhere else.
- Phase A ends when the merge converges, or the cap hits and the remainder is
  marked disputed.

## Phase B — freeze the work order

Agreeing a problem is real is **not** agreeing what to do about it, and an
unwritten intent can't be checked later. So before any code is written, the
architect posts a **work order** and Codex must `AGREE` to it (one round):

- Every surviving finding gets a stable ID (`F1`, `F2`, …), and for each:
  **file:line**, the defect in one sentence, the **agreed remedy** in concrete
  terms (what the code should do afterwards, not "fix it"), and how it will
  be **verified** (specific test, command, or observation).
- Each finding is marked **fix now** / **out of scope** (why — e.g.
  pre-existing, or a separate PR) / **still disputed**. Only *fix now* items
  get built. Disputed items — including anything still disputed when the
  Phase A merge cap hit — are never silently built or silently dropped.
- The work order is the contract Phase D checks against. Store it in the runs
  dir as `work-order.md` as well as posting it, and record it in
  `architect-state.md` — a fresh per-round subagent must be able to check
  conformance from that file alone.

## Phase C — Codex fixes

- Codex implements the *fix now* items in the worktree, exactly as the
  implement loop's coding step describes (repo ground rules, validation
  gates, never leave the worktree, never `git stash`; long runs in
  background).
- **One commit per finding where practical**, message starting with the ID
  (`F3: reject negative page sizes`). This is what makes Phase D cheap.
- Its entry is `VERDICT: IMPLEMENTED` and carries the evidence packet
  (protocol.md's implementer constraints): annotated diff, exact commands run
  and their output, a per-ID status mapping every ID to what changed and how
  it was verified — including any it chose *not* to do and why.

## Phase D — conformance check

A fresh **Sonnet** subagent (the same per-round-agent shape as the implement
loop's reviewer — not the long-lived dispatcher) re-reviews the new commits
with the work order in hand and posts a per-ID verdict table. For each ID
exactly one of:

| Verdict | Meaning |
|---|---|
| **as agreed** | Fixed, matches the agreed remedy, verification passes |
| **fixed differently** | Defect is gone but by another route — judge it on merit; if the new route is fine, say so and accept; if not, it's a `REVISE` item |
| **not fixed** | Untouched, or the remedy doesn't actually close the defect |
| **new problem** | The fix introduced something that wasn't there before |

- **Audits the evidence packet and re-runs the verification named in the work
  order itself.** Never accept "fixed" on Codex's say-so — check against the
  actual new commit.
- Also reviews the fix diff as a whole for scope creep: changes with no
  finding ID behind them get called out, and either justified or reverted.
- Anything not **as agreed** (or an accepted *fixed differently*) →
  `VERDICT: REVISE`, back to Phase C. All clean → `VERDICT: APPROVE`, and
  Codex closes with `AGREE`.

## Landing the fixes

- **Preferred**: push the fix commits to the PR's head branch, so the PR the
  user pointed at gets better. This only works if you can write to it —
  same-repo PRs, or fork PRs with maintainer-edits enabled.
- **If the push is rejected** (fork PR, no write access, protected branch):
  do not force anything. Push the `duet/<slug>` branch to your own remote and
  open a stacked PR against the original PR's branch, and say so plainly in
  the wrap-up comment and the final report. If even that fails, leave the
  commits local and report the worktree path and branch name so the user can
  pick them up.
- Either way, post the wrap-up comment on the original PR: the settled
  findings (which were unanimous, which were one side's call), what was
  fixed, what was deliberately left, and where the commits are.

## review-only shape

When the user asked for review-only, stop at the end of Phase A: no work
order, no commits, no `APPROVE` — nobody is approving code. Post the settled
review as the wrap-up comment, saying plainly which findings were unanimous
and which were one side's call (and which stayed disputed).

## Budget exhausted

- **In Phase A**, findings still disputed: post the agreed findings and list
  the disputed ones as disputed, showing both positions. A split review is a
  fine outcome; a fake consensus is not.
- **In Phase C/D**, fixes incomplete: commit and push what *is* verified as
  agreed, and report per-ID exactly what landed, what didn't, and what state
  each unfinished one is in. Never report a partially-applied work order as
  done.

## PR mode

- The PR thread replaces the notebook: every entry is a `gh pr comment`,
  opening with an attribution line per the user's global GitHub rules, e.g.
  `*[Fable 5 · Claude Code · duet orchestrator · <org>/<repo>:]*` — Codex
  posts its own comments via `gh` and attributes likewise (the turn prompt
  must spell this out). Verdict + signature rules unchanged.
