---
name: duet
description: "Use when the user types /duet or /duet:duet <task or PR> — autonomous Claude↔Codex loop: Fable/Sonnet/Codex draft independently and merge, Codex implements, tiered Sonnet/Opus review, Fable checkpoints, until both approve or the round budget (default 8) runs out"
---

# Duet — Claude designs and reviews, Codex implements

Run the whole design → agree → implement → review → approve cycle between
Claude models and Codex without the user shuttling messages. The user is
hands-off after kickoff. Every agent duet spawns gets an explicit model — see
**Model assignment** — so cost tracks the work, not whatever model happens to
be orchestrating.

**Announce at start:** "Starting duet: <task one-liner>. Mode: <spec|implement
|PR review (review-then-fix|review-only)>. <Opening line — spec/PR review:
'Fable and Codex draft blind, then Sonnet+Codex merge until converged (≤3
iterations)'; implement: 'The existing plan is the proposal — straight to the
loop.'> Fable checkpoints: <N for this mode>. Budget: <B> rounds."

## Prerequisites

- **Codex CLI**, installed and authenticated (`codex login`), able to run
  `codex exec` non-interactively. Duet drives it once per round. If
  `codex --version` fails, stop and tell the user rather than improvising a
  substitute implementer.
- **Claude Code** as the orchestrator — this skill leans on its subagents
  (with explicit `model:` on every one), background Bash, and `ToolSearch`.
- **No particular permission mode.** Duet is built to run under auto-approve
  — references/preflight.md.
- **Self-contained.** Duet depends on no other skill and no repo-specific
  setup — Worktree setup/teardown are spelled out in full below (**Set up**,
  **Finish**); don't reach for other skills or repo docs to create or retire
  the worktree, or assume any exist on this machine. Works in any git repo
  (and, degraded, outside one — see **Set up**).

## Model assignment

Every subagent duet spawns gets an explicit `model:` — never inherited from
whatever model happens to be orchestrating.

| Work | Model |
|---|---|
| Blind spec draft (Claude side); blind PR review (Claude side); end-of-run "what did the simpler models miss" check; adjudication of disputes left after the merge cap | **Fable** (explicit `model: "fable"`; if unavailable in the harness, opus) |
| Merge iterations, per-round design/review work, evidence audits, conformance checks, work-order drafting | **Sonnet** |
| Comparison matrices, log/test collection, state-file updates, usage rollups | **Haiku** (or plain scripting where no judgment is needed) |
| Blind draft B, its own merge iterations, implementation, testing, detailed self-review, evidence packets | **Codex** (subscription-billed; near-free at the margin — shift mechanical verification burden toward it) |

Fable checkpoints per mode (ceilings, not targets — unused checkpoints are not
spent):

- **Spec**: 2 — blind draft + end-check/adjudication (the end-check also
  settles disputes left when the merge cap hits).
- **Implement**: 1 — final architectural pass after the loop converges.
- **PR review**: 1 — Fable is the blind Claude-side reviewer. No separate
  Fable end-check: the independent read is where its judgment finds what
  others miss. Disputes still open after the merge cap are recorded as
  disputed, never built or forced to agreement.

**`full-duet`**: if the user says this, restore v1.2.0 behavior —
orchestrator-model everywhere, budgets 15/20, single-merge opening. It exists;
don't re-document it here — see git history.

## Parse the invocation

- **Task**: everything after `/duet` (or `/duet:duet`, the fully-qualified
  plugin form — both mean the same thing) that isn't a directive below.
- **Mode** — decides the opening shape and whether there's an implementation
  phase:
  - **Spec** — design/plan something. Blind openings.
  - **Implement** — a plan/spec already exists; build it. No blind openings;
    the existing plan is the proposal.
  - **PR review** — the invocation points at a PR someone else wrote. Blind
    openings, then converge on findings. Full phase sequence, work-order
    freeze, and landing rules: references/pr-review.md. Two shapes:
    **review-then-fix** (default) or **review-only** (only when the user
    says so — "review only", "don't change anything", "just tell me what's
    wrong", or the PR is somebody else's to fix).
  - **Multi-part**: `project <id> part <i>/<n>` invocations resolve to
    Implement mode by definition, name a governing spec, and pass mode
    explicitly — see references/multipart.md.

  Ambiguous cases: names an existing plan/spec/issue → Implement; points at
  a PR → PR review (review-then-fix unless review-only was asked for);
  otherwise Spec. State the chosen mode **and shape** in the announcement
  and the notebook header. Don't ask the user.
- **PR medium**: a PR-pointing invocation runs the exchange in the PR thread
  instead of a notebook (references/pr-review.md's **PR mode**), unless told
  otherwise. Orthogonal to mode — a Spec or Implement duet can run in a PR
  thread too. "Keep the discussion in <X>" overrides the default.
- **Budget**: default round budget is **8** standalone, **12** for PR
  review-then-fix (v1's 15/20 padded for ceremony v2 removed — the budget
  stays a runaway backstop, not a pace). "Budget 25" (or similar) overrides
  it for this run. Blind pair = Round 1; each merge iteration = one round;
  every architect+implementer exchange = one round; all phases share the
  one pool.
- **Tie-breaker**: default is pause-and-escalate at budget exhaustion. "Figure
  it out" + a named tie-breaker (e.g. "Fable decides") makes that side settle
  disputes at the cap. "Figure it out" with no name → ask who breaks ties NOW
  (AskUserQuestion), before starting. This is the only question allowed.
  **After kickoff the run is fully autonomous — never ask the user anything
  mid-run**, except a permission/credential wall that stops the run outright
  (duet can't approve on the user's behalf, so it stops, says what's
  blocked, and waits — references/preflight.md's **Permissions**). Preflight
  exists so this practically never happens after kickoff.

## Set up

- **Prune old duet branches first.** Local branches matching `duet/*` in this
  repo that are already merged into `origin/HEAD` (or `HEAD` if no remote) are
  leftovers from prior runs — GitHub's delete-on-merge setting, if on, only
  cleans up the remote side, never local refs, and duet's own Tidy-up step
  deliberately never deletes a branch (the PR may still be open when a run
  ends). Before creating this run's worktree: `git fetch --prune` (drops
  remote-tracking refs for branches GitHub already deleted), then `git branch
  -d $(git branch --list 'duet/*' --merged origin/HEAD)` (falls back to
  `--merged HEAD` with no remote). Safe unconditionally — lowercase `-d`
  refuses anything not merged, and refuses a branch checked out in another
  worktree, so a run still in flight on a different slug is untouched.
- In a git repo: create an isolated worktree at
  `~/.claude/duet/<repo>/worktrees/<slug>` on branch `duet/<slug>`
  (from `origin/HEAD` if a remote exists, else `HEAD`); per-turn prompts and
  logs in `~/.claude/duet/<repo>/runs/<slug>/`. PR mode: `git fetch origin
  pull/<n>/head` and check that out in the worktree. Not a git repo: work in
  place, skip the worktree.
- **PR review / review-then-fix needs a pushable branch** — `pull/<n>/head`
  lands you detached. Look up the head with `gh pr view <n> --json
  headRefName,headRepositoryOwner,maintainerCanModify` and check out a local
  branch tracking that head branch on its own remote (adding the fork as a
  remote if needed) rather than the detached `FETCH_HEAD`. If that isn't
  possible, fall back to branch `duet/<slug>` started at the PR head — fixes
  still get made, they just land per references/pr-review.md's **Landing
  the fixes**. Decide this at setup, not after the fixes exist.
- **Notebook**: `.duet/<YYYY-MM-DD>-<slug>.md` inside the worktree. Add
  `.duet/` to `.git/info/exclude` (never commit it unless asked). Header:
  task, mode, date, participants (ACTUAL model names), budget state.
- Read references/protocol.md — send the slice its "what to send when" note
  calls for: Core every turn, blind-opening/merge only during Round 1 and
  merge iterations, PR-review only in PR mode.

**Preflight**: run auth checks, the session-economy probe, and the
permission-handshake smoke test before Round 1 — see references/preflight.md,
which also has the exact round command shape **The loop** uses.

## Context economy — orchestrate rounds through fresh subagents (default)

The orchestrating context should stay a **thin dispatcher**; the heavy
per-round work runs in fresh subagents — the default shape, not a long-run
optimization, since it's rarely obvious ahead of time which runs turn long
and a long-lived orchestrator costs more every round (the transcript only
grows; caching softens that but a 60-minute Codex turn can outlast the cache
TTL regardless). The runs dir + PR thread/notebook preserve everything, so
nothing is lost by staying thin.

- **State file**: at kickoff, create `<runs>/<slug>/architect-state.md` — mode,
  protocol essentials + paths, verification bar, settled design rulings
  (do-not-relitigate list), Fable-checkpoint and round accounting, the
  session-economy probe result, open findings, review-methodology notes.
  Whoever completes a round updates it; a fresh context needs only this file
  plus the latest notebook/PR entries to act as the architect.
- **Per-round agent**: run each design/review round as a *fresh* subagent
  with an explicit model per the table above (never a fork — forks inherit
  the full transcript, defeating the purpose), prompted with the state file
  path, worktree, runs dir, and round task. It re-runs the verification bar
  itself, fans out its own adversarial review agents for substantive diffs,
  drafts and posts the entry, updates the state file, and returns only a
  compact summary + verdict.
- **Dispatcher keeps**: launching Codex turns, watching CI, round/budget and
  Fable-checkpoint accounting, tie-breaker decisions, usage-log writes, and
  anything requiring the user — growing by roughly a paragraph per round.

## Opening — iterated independent merge (spec and PR-review modes)

Whoever writes first anchors the other, so spec/PR-review modes open blind
and converge through up to 3 independent merge attempts, not one merge.
Skip in **implement** mode — the plan already exists; go to **The loop**.
(PR review runs this over finding lists, not plans — references/pr-review.md.)

**Round 1 — blind openings** (references/protocol.md's blind-opening
rules): dispatch Codex in the background (Bash `run_in_background`) and a
fresh **Fable** subagent in parallel, no extra wall-clock. Codex writes
`<runs>/round-1-codex-open.md`, Fable `<runs>/round-1-claude-open.md`;
neither reads the notebook/PR thread or gets any opinion of yours. Post
both verbatim, labeled and signed by their real authors, `VERDICT: PROPOSE`
each.

**Merge iterations, i = 1, 2, 3 (cap 3; each = one round):**

1. Both sides get both current drafts (A/B, then A′ᵢ₋₁/B′ᵢ₋₁).
2. Fresh **Sonnet** produces merger A′ᵢ; **Codex** produces B′ᵢ blind to A′ᵢ
   until both exist (Codex may resume its own session — its history is
   fine, A′ᵢ early is not). Each merger takes the stronger option point by
   point (never splits differences), listing every disagreement — side
   taken, why, what was rejected. Unjustified total agreement in one hop is
   a defect, not convergence.
3. **Haiku** builds a mechanical comparison matrix of A′ᵢ vs. B′ᵢ:
   agreements, substantive disagreements, one-sided ideas, scope
   differences — no judgment calls; unsure means substantive.
4. **Converged** = no substantive disagreements in the matrix (wording
   doesn't count) — the dispatcher picks the more complete draft as the
   plan, noting the choice.
5. **Cap: 3 iterations.** Still disputed at the cap → the Fable end-check
   (spec mode) or recorded disputed (PR review).

Drafts and mergers post verbatim and signed once each pair exists — same
honesty rule as Round 1.

**Spec-mode finish — Fable end-check** (checkpoint 2/2): reads the
converged plan and full disagreement history, asks "what did the simpler
models miss?", settles capped disputes, issues the **charter** — goals,
non-goals, settled decisions (do-not-relitigate list), invariants,
acceptance checks, open items. Lands in `architect-state.md` and the
notebook.

**Budget:** the blind pair is Round 1; each merge iteration is one round.

## The loop

One **round** = one architect entry + one implementer entry, or one merge
iteration during Opening. **The budget covers all phases together** — design,
merge, implementation, and review draw from the same pool. Record it in the
notebook header and label every entry `Round N/<B>`.

In PR-review mode these steps are re-shaped into Phases A–D —
references/pr-review.md.

1. **Design**: in spec/PR-review modes, Round 1 is the blind opening above
   and the loop picks up once the merge converges. In implement mode, write
   the initial design in the notebook, `VERDICT: PROPOSE`. Sign as the model
   actually running the round — verify identity, don't assume.
2. **Codex turn**: write a prompt file in the runs dir (round number, phase,
   absolute paths, protocol slice, "read the notebook, append your entry"),
   then run the round command in the exact shape settled at preflight
   (references/preflight.md) — only `-o` and the prompt differ, which is
   what makes one approval cover the whole run. Always a fresh `codex exec`
   per turn with a protocol slice — `codex exec resume` is not usable for
   this (references/preflight.md's **Codex session economy**). Verify Codex
   appended a well-formed entry (`grep
   '^VERDICT:'` on the tail); if it answered in `-o` output but didn't
   append, append it yourself, marked "(appended by orchestrator)".
3. Alternate until both sides say `VERDICT: AGREE` on the design. Respond
   point-by-point; concede when Codex is right.
4. **Implement**: Codex codes in the worktree (its half of a round). Prompt
   includes the agreed design, repo ground rules (AGENTS.md/CLAUDE.md),
   validation gates, "never leave the worktree, never `git stash`." Long
   runs: Bash `run_in_background`, then monitor.
5. **Review**, tiered by how substantive the diff is:

   | Tier | Depth |
   |---|---|
   | First substantive diff | Full review — multiple angles, findings adversarially verified, fresh **Sonnet** subagents |
   | Revision rounds | Delta-only — verify the named findings' fixes plus a scope-creep scan of the whole diff |
   | Trivial diff (small, mechanical, no interface/behavior change) | Single Sonnet reviewer, no fan-out |

   Every `IMPLEMENTED` entry carries an evidence packet
   (references/protocol.md); the reviewer audits it and re-runs the named
   verification commands instead of re-deriving the review — "fixed" without
   artifacts is never accepted. Findings get stable IDs (F1, F2, …); Codex
   answers item by item, by ID; one commit per finding where practical.
   **Escalate that round to Opus** (never Fable — reserved for named
   checkpoints; not sticky) when Codex disputes a `REVISE` finding, the diff
   is architectural (module boundary, public interface, dependency, or
   concurrency/persistence/security semantics), or the round would change a
   do-not-relitigate ruling. If Opus isn't available in the harness (model
   overrides can fail per environment), the Sonnet round proceeds but must
   flag the tripped trigger in its entry so the Fable final pass looks there
   first — don't burn a Fable checkpoint as a substitute. `VERDICT: REVISE` (file:line, most severe
   first) or `APPROVE`. **Done when the reviewer writes APPROVE and Codex's
   closing entry is AGREE.**
6. **Fable final pass** (implement mode's one checkpoint): once the above is
   done, one Fable round reviews charter + evidence + full diff at
   architectural altitude — intended shape preserved, details distorting
   the design, unnecessary machinery, missed concerns, repo coherence.
   `APPROVE` ends the run. `REVISE` → Codex fixes → **Sonnet** verifies; the
   final report says plainly "verified by Sonnet, not re-reviewed by
   Fable." Never loop Fable again by default.

## Instrumentation

- Every spawned agent gets a line appended to `<runs>/usage.jsonl`:
  `{"round": N, "phase": "...", "role": "...", "model": "...", "started":
  "...", "wall_secs": N, "tokens": {...}|null}` — tokens filled when the
  harness exposes them, else null; wall time and model always recorded. The
  dispatcher writes the line for its own subagents, and for Codex turns from
  the `-o` transcript where Codex reports usage.
- The final report includes a rollup: agents spawned per model, Fable
  checkpoints used vs. ceiling, rounds used vs. budget, token totals when
  available. Multi-part runs copy each part's rollup into the project
  ledger — references/multipart.md.

## Finish

- Commit in the worktree; push/PR only if the task calls for it (PR mode:
  push to the PR branch, post a wrap-up comment — in PR-review mode follow
  references/pr-review.md's **Landing the fixes**, including the fallback
  when the push is rejected).
- Final report to the user in plain language: what was built, how it was
  verified, notebook path, worktree path, diff summary, and the
  **Instrumentation** rollup. In PR-review mode, include the per-ID
  conformance table — which findings were fixed as agreed, which
  differently, which deliberately left.
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
  gone for good. The branch and its commits survive; only the checkout
  folder goes away. If the command refuses because of uncommitted changes,
  say so in the final report and leave the worktree alone — `--force` only
  if the user explicitly asks to discard that work. Not a git repo: nothing
  to clean up.
- Load PushNotification (`ToolSearch "select:PushNotification"`) and ping the
  user that it's done. Same on any early stop.

## Budget exhausted

- **No tie-breaker named**: stop. Write "Where we're stuck" — in the notebook
  and as the final message — in words a 10-year-old could follow: what both
  sides agree on, what they're stuck on, what each side wants. No jargon, as
  brief as clarity allows. Ping the user.
- **Tie-breaker named**: that side decides each open dispute (documented in
  the notebook), then up to 3 extra rounds to finish under those decisions.
  Still not done → stop and summarize as above.
- Never report a partially-completed run as done, and never manufacture
  mutual approval because the budget ran out.
