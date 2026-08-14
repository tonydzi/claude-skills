---
name: duet
description: "Use when the user types /duet or /duet:duet <task or PR> — autonomous Claude↔Codex loop: Claude designs & reviews, Codex implements, until both approve or the round budget (default 15) runs out"
---

# Duet — Claude designs and reviews, Codex implements

Run the whole design → agree → implement → review → approve cycle between the
orchestrating Claude model and Codex without the user shuttling messages.
The user is hands-off after kickoff.

**Announce at start:** "Starting duet: <task one-liner>. Mode: <spec|implement|PR
review (review-then-fix|review-only)>. <Opening line: 'Codex and I will draft
independently, then I'll merge the two' — or, in implement mode, 'I'll design
first' — then loop with Codex.> Budget: <B> rounds."

## Prerequisites

- **Codex CLI**, installed and authenticated (`codex login`), able to run
  `codex exec` non-interactively. Duet drives it once per round. If
  `codex --version` fails, stop and tell the user rather than improvising a
  substitute implementer.
- **Claude Code** as the orchestrator — this skill leans on its subagents,
  background Bash, and `ToolSearch`.
- **No particular permission mode.** Duet is built to run under auto-approve;
  see **Preflight** and **Permissions**. It asks for the least it can, and
  asks once, at the start.
- Nothing else. Duet needs no other skills and no repo-specific setup; it
  works in any git repo (and, degraded, outside one — see **Set up**).

## Parse the invocation

- **Task**: everything after `/duet` (or `/duet:duet`, the fully-qualified
  plugin form — both mean the same thing) that isn't a directive below.
- **Mode** — decides the opening shape (see **Opening** below) and whether
  there's an implementation phase:
  - **Spec** — the task is to design/plan something. Blind openings.
  - **Implement** — a plan/spec already exists and the job is to build it.
    No blind openings; the existing plan is the proposal.
  - **PR review** — the invocation points at a PR whose code was written by
    someone else. Blind openings, then converge on findings. It has two
    shapes (see **PR-review mode** below):
    - **review-then-fix** (the default): after both sides agree the finding
      list is right, Codex fixes the agreed findings and the orchestrator
      checks the diff against what was agreed.
    - **review-only**: stop at the agreed finding list, nobody commits. Use
      this only when the user says so — "review only", "don't change
      anything", "just tell me what's wrong", or the PR is somebody else's
      to fix.

  Ambiguous cases: if the task names an existing plan/spec/issue to follow,
  it's Implement; if it points at a PR, it's PR review (review-then-fix
  unless the user asked for review-only); otherwise Spec. State the chosen
  mode **and shape** in the announcement and the notebook header. Don't ask
  the user.
- **PR medium**: if the invocation points at a PR (number/URL), the exchange
  happens in the PR thread instead of a notebook (see PR mode below), unless
  the user says otherwise. This is orthogonal to mode — a Spec or Implement
  duet can also run in a PR thread.
- **Medium override**: "keep the discussion in <X>" wins over defaults.
- **Budget override**: "budget 25" (or similar) sets the round budget for
  this run; otherwise it's 15 — or **20 in PR review / review-then-fix**,
  which pays for review, fixes, and the conformance check out of one pool. A
  typical task converges in ~10 rounds — the budget is headroom to catch
  runaway loops, not a target, so don't slow-walk toward it or stop early
  because rounds remain.
- **Tie-breaker**: default is pause-and-escalate at budget exhaustion. If the
  user says "figure it out" AND names a tie-breaker (e.g. "Fable decides"),
  that side settles disputes at the cap. If they say "figure it out" without
  naming one, ask who breaks ties NOW (AskUserQuestion), before starting.
  This is the only question allowed. **After kickoff the run is fully
  autonomous — never ask the user anything mid-run.** The single exception is
  a permission or credential wall that stops the run outright: duet cannot
  approve on the user's behalf, so it stops, says exactly what is blocked, and
  waits (see **Permissions**). **Preflight** exists so this practically never
  happens after kickoff — every environment question is settled before Round 1.

## Set up

- In a git repo: create an isolated worktree at
  `~/.claude/duet/<repo>/worktrees/<slug>` on branch `duet/<slug>`
  (from `origin/HEAD` if a remote exists, else `HEAD`); per-turn prompts and
  logs in `~/.claude/duet/<repo>/runs/<slug>/`. PR mode: `git fetch origin
  pull/<n>/head` and check that out in the worktree. Not a git repo: work in
  place, skip the worktree.
- **PR review / review-then-fix needs a pushable branch**, and
  `pull/<n>/head` lands you detached. Look up the head with
  `gh pr view <n> --json headRefName,headRepositoryOwner,maintainerCanModify`
  and check out a local branch tracking that head branch on its own remote
  (adding the fork as a remote if needed) rather than the detached `FETCH_HEAD`.
  If that isn't possible, fall back to branch `duet/<slug>` started at the PR
  head — the fixes still get made, they just land per **Landing the fixes**.
  Decide this at setup, not after the fixes exist.
- **Notebook**: `.duet/<YYYY-MM-DD>-<slug>.md` inside the worktree. Add
  `.duet/` to `.git/info/exclude` (never commit it unless asked). Header:
  task, mode, date, participants (ACTUAL model names), budget state.
- Read `references/protocol.md` (next to this file). It is the contract both
  sides follow; hand its full text to Codex with every turn prompt.

## Preflight — settle auth and permissions before Round 1

Duet is hands-off after kickoff, so every environment question gets answered
**now**, in one handshake, instead of surfacing as a denied command in round 7
with a half-built worktree on the floor. Run these before the announcement.
Steps 1–2 can run while the worktree is being set up; step 3 needs it.

1. **Codex is there**: `codex --version`. If it fails, stop and say so — never
   improvise a substitute implementer.
2. **How Codex authenticates**: `codex login status`.
   - `Logged in using ChatGPT` → a stored subscription login exists: **keep**
     the `env -u CODEX_API_KEY -u OPENAI_API_KEY` prefix, so a long
     implementation run bills the subscription instead of the API.
   - Not logged in (an API key in the environment is doing the work) →
     **drop** the prefix, or `codex exec` fails to authenticate once the keys
     are stripped.

   Decide once, here, and use the same prefix for every turn of the run.
   **Never read `~/.codex/auth.json`** or any other credential file to answer
   this. `codex login status` reports the same thing without touching a
   secret, and "read the file holding my API keys" is exactly the shape a
   permission classifier denies — correctly.
3. **Smoke test — this doubles as the permission handshake.** Once the
   worktree and runs dir exist, run the *real* command shape once with a
   throwaway prompt:

   ```bash
   env -u CODEX_API_KEY -u OPENAI_API_KEY \
     codex exec -s workspace-write --add-dir "<runs>" \
     -c sandbox_workspace_write.network_access=true \
     -C "<worktree>" -o "<runs>/smoke.md" \
     "Reply with exactly: DUET_SMOKE_OK" </dev/null
   ```

   **Flag for flag, this is the command the rounds use** — only `-o` and the
   prompt differ. That is the whole point: the approval granted here covers
   every round that follows. A smoke test in a *simpler* shape is worse than
   none, because it proves the rounds will be allowed when they won't.

   - **`DUET_SMOKE_OK`** → go. Announce and start Round 1.
   - **Auth error** → step 2 read the environment wrong. Flip the `env -u`
     decision, retry once, and use whichever worked for the rest of the run.
   - **Denied by the permission system** → stop *here*, before any design
     work. Show the user the allowlist under **Permissions**, ask once, then
     resume. Do **not** quietly retry in a weaker shape to slip past the
     denial — a Codex that cannot write the worktree produces a run that looks
     alive and builds nothing. If the user declines, remove the worktree and
     stop.

**Never wrap a Codex call in `timeout`** — it is not present on stock macOS, so
the whole command dies with `env: timeout: No such file or directory` and reads
like a Codex failure. Long turns go to Bash `run_in_background` and get
monitored, which is the rule for implement turns anyway.

## Permissions

Duet asks for the least it can, and asks for it once. A colleague on
auto-approve should see at most one prompt, at the smoke test.

| Command | Why duet needs it |
|---|---|
| `codex login status` | which auth Codex uses — reads no secret |
| `codex exec -s workspace-write …` | the implementer's turn, once per round |
| `git worktree add` / `remove` | the isolated checkout duet works in |
| `gh pr view` / `gh pr comment` | PR mode only — the thread is the notebook |

If the smoke test comes back denied, hand the user exactly this, say in one
line what each entry buys, and let them add it via `/permissions` or
`.claude/settings.json`:

```json
"permissions": {
  "allow": [
    "Bash(codex login status)",
    "Bash(env -u CODEX_API_KEY -u OPENAI_API_KEY codex exec:*)",
    "Bash(codex exec:*)",
    "Bash(git worktree:*)",
    "Bash(gh pr view:*)",
    "Bash(gh pr comment:*)"
  ]
}
```

Report the denial as what it is — the harness's permission gate, not a refusal
by the model and not a broken Codex. Users read "denied" as "Claude won't do
it" and argue with the wrong party.

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
default 15; 20 for PR review / review-then-fix) covers all phases together** —
design, implementation, and review draw from the same pool. Record B in the
notebook header and label every entry `Round N/<B>`.

In PR-review mode the steps below are re-shaped into Phases A–D; see
**PR-review mode**.

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
     codex exec -s workspace-write --add-dir "<runs>" \
     -c sandbox_workspace_write.network_access=true \
     -C "<worktree>" -o "<runs>/round-<n>-codex.md" \
     "$(cat "<runs>/round-<n>-prompt.md")" </dev/null
   ```

   Every part of that line earns its place — use the same shape every round,
   so one approval covers the whole run:

   - **`-s workspace-write`** confines Codex's writes to its working root.
     That is the actual safety property duet wants (Codex touches the
     throwaway worktree, nothing else), and it reads to a permission
     classifier as an ordinary sandboxed tool call rather than a request to
     switch safety off.
   - **`--add-dir "<runs>"`** because the runs dir sits outside the worktree
     and Codex writes there — the `-o` transcript, and prompts it may read.
     Without it those writes fail inside the sandbox (verified 2026-08-14).
   - **`-c sandbox_workspace_write.network_access=true`** because
     `workspace-write` blocks the network by default, and implement turns
     routinely need `gh`, `git fetch`, or a package install (verified
     2026-08-14). Keep it by default. The only case for dropping it is a
     notebook-medium run where Codex neither builds anything nor posts to a
     PR — note that PR mode always needs it, review-only included, since Codex
     posts its own entries with `gh pr comment`.
   - **`</dev/null`** is required: with an open non-tty stdin, `codex exec`
     blocks forever waiting for stdin to close (verified 2026-08-11).
   - **The `env -u` prefix** is whatever **Preflight** settled — keep it for a
     subscription login, drop it for API-key-only auth. Don't re-derive it
     mid-run.

   **Escalation is a last resort, never a starting point.** Do not open with
   `--dangerously-bypass-approvals-and-sandbox`. Reach for it only if a turn
   actually fails on something `workspace-write` cannot cover — Codex needing
   to write outside the worktree, drive Docker, or reach a system path. Then
   stop, name the exact command that failed and why, get the user's OK, and
   use it from that turn on. Starting there is what gets duet denied on round
   one in auto-approve sessions.

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
  push to the PR branch, post a wrap-up comment — in PR-review mode follow
  **Landing the fixes**, including the fallback when the push is rejected).
- Final report to the user in plain language: what was built, how it was
  verified, notebook path, worktree path, diff summary. In PR-review mode,
  include the per-ID conformance table — which findings were fixed as agreed,
  which differently, which deliberately left.
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

Reviewing a PR: the two sides review independently, converge on what's wrong,
and then — in the default **review-then-fix** shape — Codex actually fixes it
and the orchestrator checks the fixes against what was agreed. Agreeing on a
list of complaints is the *middle* of this mode, not the end.

### Phase A — converge on the findings

- **Set up** as PR mode below (fetch and check out the PR head in the
  worktree). Nobody commits during Phase A; the checkout is read-only until
  the work order is frozen.
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
- Phase A ends when both sides `AGREE` the finding list is right.

### Phase B — freeze the work order

Agreeing a problem is real is **not** agreeing what to do about it, and an
unwritten intent can't be checked later. So before any code is written, the
orchestrator posts a **work order** and Codex must `AGREE` to it (one round):

- Every surviving finding gets a stable ID (`F1`, `F2`, …), and for each:
  **file:line**, the defect in one sentence, the **agreed remedy** in concrete
  terms (what the code should do afterwards, not "fix it"), and how it will
  be **verified** (specific test, command, or observation).
- Each finding is marked **fix now** / **out of scope** (why — e.g. pre-existing,
  or a separate PR) / **still disputed**. Only *fix now* items get built.
  Disputed items are never silently built or silently dropped.
- The work order is the contract Phase D checks against. Store it in the runs
  dir as `work-order.md` as well as posting it, and record it in
  `architect-state.md` — a fresh per-round subagent must be able to check
  conformance from that file alone.

### Phase C — Codex fixes

- Codex implements the *fix now* items in the worktree, exactly as the loop's
  **Implement** step describes (repo ground rules, validation gates, never
  leave the worktree, never `git stash`; long runs in background).
- **One commit per finding where practical**, message starting with the ID
  (`F3: reject negative page sizes`). This is what makes Phase D cheap.
- Its entry is `VERDICT: IMPLEMENTED` and must map every ID to what changed
  and what was run to verify it — including any it chose *not* to do and why.

### Phase D — conformance check

The orchestrator re-reviews the new commits with the work order in hand and
posts a per-ID verdict table. For each ID exactly one of:

| Verdict | Meaning |
|---|---|
| **as agreed** | Fixed, matches the agreed remedy, verification passes |
| **fixed differently** | Defect is gone but by another route — judge it on merit; if the new route is fine, say so and accept; if not, it's a `REVISE` item |
| **not fixed** | Untouched, or the remedy doesn't actually close the defect |
| **new problem** | The fix introduced something that wasn't there before |

- Re-run the verification named in the work order yourself. **Never accept
  "fixed" on Codex's say-so** — check against the actual new commit.
- Also review the fix diff as a whole for scope creep: changes with no
  finding ID behind them get called out, and either justified or reverted.
- Anything not **as agreed** (or an accepted *fixed differently*) →
  `VERDICT: REVISE`, back to Phase C. All clean → `VERDICT: APPROVE`, and
  Codex closes with `AGREE`.

### Landing the fixes

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
  findings (which were unanimous, which were one side's call), what was fixed,
  what was deliberately left, and where the commits are.

### review-only shape

When the user asked for review-only, stop at the end of Phase A: no work
order, no commits, no `APPROVE` — nobody is approving code. Post the settled
review as the wrap-up comment, saying plainly which findings were unanimous
and which were one side's call.

### Budget exhausted

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

## Integration

**Self-contained by design.** Duet depends on no other skill. Worktree setup
and teardown are spelled out in full above (**Set up**, **Finish**) — do not
reach for other skills or repo docs to create or retire the worktree, and do
not assume any exist on this machine.
