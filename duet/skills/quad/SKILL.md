---
name: quad
description: "Use when the user types /duet:quad <thing to build> — runs the full design-through-shipped-code lifecycle for one thing as four chained /duet:duet runs: design, design review+merge, implementation, implementation review+merge. Adds a shared budget ledger across all four, a standing gate for decisions that need a human rather than agent agreement, a liveness watchdog for the long dispatcher runs, and cleanup once everything lands."
---

# Quad — design it, review the design, build it, review the build

Four `/duet:duet` runs chained end to end for one piece of work, so a design
gets built without someone manually kicking off each stage and checking in on
it. Requires the `duet` skill (same plugin — quad is a thin sequencing layer
on top of it, not a reimplementation). Everything about how an individual
duet run actually works (models, verdicts, rounds, review tiers, Fable
checkpoints) is unchanged; quad only adds what doesn't exist at the
single-run level: sequencing, a cross-run budget, a place for decisions that
need the user rather than agent agreement, watching four long-running
dispatchers instead of one, and tidying up after all four.

**Announce at start:** "Starting quad: <thing to build>. Four stages: design
→ design review → implementation → implementation review. Shared budget:
<ledger totals>. I'll check in at each stage, and immediately if anything
needs you."

## Prerequisites

Same as `duet` (Codex CLI authenticated, no particular permission mode) —
quad adds no new tooling requirement, it just runs `duet` four times.

## Parse the invocation

- **Task**: everything after `/duet:quad` — the thing to design and build.
  This becomes each stage's own duet task, phrased for that stage (see
  **Stages** below).
- **Slug**: derive one short slug from the task (e.g.
  `reporter-identity-fix`) and reuse it, suffixed per stage, for every
  worktree/runs-dir name this run creates — `<slug>-design`,
  `<slug>-design-review`, `<slug>-impl`, `<slug>-impl-review`. This is what
  makes cleanup and status-checking coherent across four otherwise-separate
  duet runs.
- **Budget**: default pooled totals mirror `duet`'s own per-mode defaults
  (SKILL.md's **Model assignment**) summed across all four stages — **40
  rounds** (8 Spec + 12 PR-review + 8 Implement + 12 PR-review) and **5
  Fable checkpoints** (2 + 1 + 1 + 1). "Budget N" overrides the round total.
  See **Budget ledger** below for how stages draw from it.
- **Everything else** (tie-breaker, PR medium, etc.) passes through
  unchanged to each stage's own duet invocation.

## Set up — the project ledger

One file, `~/.claude/duet/<repo>/quad/<slug>/ledger.md`, created before
Stage 1 and passed to every stage:

- **Round pool** and **Fable-checkpoint pool**: total/remaining (from
  **Budget** above).
- **Per-stage status**: not started / in progress / done, with its PR
  number once one exists and its own round spend once it reports in.
- **Open items**: anything a stage flagged that needs the user (see **The
  human-decision gate**) or that got deferred as a manual follow-up
  (a credential to provision, a rollout step) — carried forward to the
  final report so nothing quietly drops between stages.

Each stage is a *fresh* subagent (never a fork) that itself invokes the
`duet` skill and drives it to completion — same "thin dispatcher" reasoning
as `duet`'s own **Context economy** section, one level up: a quad run can
span many hours across four long dispatcher lifetimes, and nothing should
have to live in one context for all of it.

## Stages

Run sequentially — each stage's input is the previous stage's merged
output, so there's no parallelism to have here (unlike `duet`'s own
within-run rounds, which can background a Codex turn while other work
happens).

1. **Design** — Spec mode, task: "design <the thing>". Ends with a PR open,
   *not merged* — tell the dispatching subagent explicitly not to merge; a
   separate stage reviews it.
2. **Design review** — PR-review mode (review-then-fix) on Stage 1's PR.
   Merges on convergence — this is the only stage allowed to just merge
   without a further check, same as any duet PR-review run authorized to
   merge on its own agreement.
3. **Implementation** — Implement mode, task: "build what
   `<path to the merged design doc>` specifies". Ends with a PR open, not
   merged.
4. **Implementation review** — PR-review mode (review-then-fix) on Stage
   3's PR. Merges on convergence. This is quad's last stage — once this
   merges, the thing is shipped.

Between Stage 2 merging and Stage 3 starting, `main` may have moved (another
quad stage, or anything else landing in the meantime) — Stage 3 starts from
current `origin/HEAD`, same as any fresh duet worktree, so this is automatic.
Between Stage 4 starting and finishing, the same is true in reverse — its own
worktree may need a rebase before it can merge cleanly if `main` moved again;
that's ordinary, not a failure, and Stage 4's own dispatcher handles it as
part of landing the fix (same as `duet`'s **Landing the fixes**).

## The human-decision gate

Some designs contain a decision that isn't the agents' to make — not because
it's hard, but because it's not a correctness question: it changes who can
see something, spends real money irreversibly, or reflects a policy/risk
choice rather than an engineering one. A design duet that hits this should
say so explicitly, with a **standard, greppable marker** in the design
document itself:

```markdown
## HUMAN DECISION REQUIRED

<the decision, framed concretely: what's being chosen between, and why an
engineering-only choice isn't sufficient — see the "why gate this" test
below>
```

After Stage 2 merges, before dispatching Stage 3, **check the merged design
document for this heading**. If present: stop, report the decision to the
user exactly as the design phrased it, and do not dispatch Stage 3 until an
explicit answer arrives. If absent: proceed straight to Stage 3, same as
today.

**Why gate this and nothing else** (tell the design duet this test, so it
knows when to use the marker): would getting this wrong be fixable with a
follow-up patch, or would real damage already be done by the time anyone
noticed — data seen by people who shouldn't have seen it, money spent that
can't be clawed back? The first case is ordinary engineering judgment and
doesn't need the marker, even for a hard call. The second is what the marker
is for.

## Credential-handling convention

If a design's implementation needs a real secret (an API key, an access
token) to actually go live, the implementation stage builds code that
consumes it via a clearly-named, documented environment-variable or config
reference — **never invents or hardcodes a value**. Provisioning the real
credential (who owns it, its exact scope, rotation cadence) is a manual
follow-up recorded in the ledger's **Open items**, not something that blocks
Stage 3 or Stage 4 from finishing and merging.

## Liveness watchdog

A stage's dispatcher subagent can legitimately go quiet for a long stretch —
a single Codex round doing careful verification has run past 50 minutes more
than once — and separately, these subagents intermittently and harmlessly
drop out of `ListAgents` between their own async dispatches, which looks
identical to a stall from the outside. Don't wait blind:

- **While actively responding to the user**, a background cron/wakeup cannot
  fire (it only runs when the session is idle between turns) — so during any
  long stretch of conversation, periodically break in yourself and check
  rather than assuming a watchdog has it covered.
- **Otherwise**, poll roughly every 20 minutes: `ListAgents`, plus real
  liveness signs (`ps aux | grep -i codex`, file mtimes under the current
  stage's runs dir) rather than trusting `ListAgents` status alone. Recent
  file activity or a live `codex exec` process → say nothing, it's fine.
  Nothing for ~45+ minutes and no live process → `SendMessage` the stage's
  dispatcher directly asking for its real status; a genuinely dead process
  can't respond to this, so a reply (even "still working") is real signal a
  bare status flag isn't.
- If a nudge gets no response and file activity stays flat: check the
  stage's latest round for a visible error first. If none, this is worth
  telling the user directly rather than waiting further — it may be a real
  resource limit (e.g. a model access/credit problem, testable by spawning a
  trivial subagent on the model in question and seeing if it responds at
  all) rather than an ordinary stall.

## Cleanup

Only after Stage 4 merges. For each of the four stages' worktrees, run
`duet`'s own **Tidy up** step (copy `.duet/` into that stage's runs dir,
`git worktree remove`). If any refuses (uncommitted stray files unrelated to
the actual work — this has happened more than once with a leftover
`.factory/` directory) leave it and note the path in the final report
rather than forcing it. Then write one line per stage into the ledger
recording final status, PR number, and rounds spent, so the ledger alone
tells the whole run's story without anyone having to open all four stages'
individual notebooks.

## Final report

Short and big-picture by default — this is the one place quad should
deliberately diverge from `duet`'s own (appropriately detailed) per-run
report convention, because a quad final report is summarizing *four* of
those:

- What got built, in a sentence or two.
- The two merged PR links (design-review's merge, implementation-review's
  merge) — not a per-finding recap; anyone who wants that detail can open
  the stage's own notebook.
- Total rounds spent vs. the pooled budget.
- **Open items** from the ledger — anything still needing the user (a
  deferred credential, a rollout step, a disputed finding that was recorded
  rather than resolved).

## Budget exhausted

Same circuit-breaker shape `duet`'s own multi-part reference already solved
for a related problem (`references/multipart.md` in the `duet` skill — reuse
its reasoning, don't reinvent it): as the shared pool depletes, escalations
get more conservative before stages stop starting entirely. Concretely, at
60% of the pool consumed, drop per-round Opus escalation except for genuine
disagreement; at 80%, delta-only review everywhere and no new stage starts
once the current one finishes; at 95%, finish whatever's in flight and stop
cleanly rather than starting Stage 3 or 4. Never report a stage — or the
whole run — as done if it degraded partway; the final report says plainly
which stages ran at full depth and which didn't.
