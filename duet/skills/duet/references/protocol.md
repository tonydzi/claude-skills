# Duet protocol

Two agents share one task. The **architect/reviewer** (a Claude model — Fable,
Sonnet, or Opus depending on the round, see the model table in SKILL.md)
designs and reviews; the **implementer** (Codex) critiques the design and
writes the code. They converse by appending entries to a shared notebook (or,
in PR mode, posting PR comments).

**What to send when:**
- **Core** goes to both sides every turn — it's the standing contract.
- **Blind opening & merge iterations** goes out only during Round 1 and each
  merge iteration (spec and PR-review modes) — a routine implement-phase turn
  never carries it.
- **PR-review sequencing & attribution** goes out only when the medium is a
  PR (any mode) or the mode is PR review.

## Core (always)

### Entry format

```markdown
## Round <N>/<budget> — <Architect|Implementer>

<body>

VERDICT: <verdict>
— <actual model name> (<harness>, <architect|implementer>)
```

- **Append-only.** Never edit or delete another entry (or your own past ones).
  This includes the Round 1 blind drafts and every merge iteration's output
  once posted — a later merger supersedes an earlier one in force, but the
  text stays as written.
- The `VERDICT:` line is the last line before the signature, exactly
  `VERDICT: <WORD>` at line start (it is machine-parsed).
- **Sign as the model you actually are** — e.g. `— Fable 5 (Claude Code,
  architect)` for a blind draft or end-check, `— Sonnet 5 (Claude Code,
  architect)` for a routine round, `— Opus 5 (Claude Code, architect)` for an
  escalated round, `— GPT-5.1-codex (codex exec, implementer)` for Codex. The
  seat name (architect/implementer) never changes; the model filling it does,
  round to round — always sign the one that actually ran.
- One **round** = one architect entry + one implementer entry (or, during the
  merge phase, one merge iteration). The round budget is stated in the
  notebook header (default 8; 12 for PR review-then-fix) and is shared across
  design, merge, implementation, and review. State your round number. The
  budget is a runaway-loop backstop, not a pace to fill — converge as fast as
  correctness allows.

### Verdicts

| Verdict | Who | Meaning |
|---|---|---|
| `PROPOSE` | either | Opening design or review — including a blind opening or a merger. |
| `REVISE` | either | I want changes — each one listed concretely. |
| `AGREE` | either | I sign off on the current plan/state. |
| `IMPLEMENTED` | implementer | Coding pass done: what changed, how verified (evidence packet — see below). |
| `APPROVE` | architect | The code is good. |

Design phase ends when **both** sides have said `AGREE` on the same plan. The
duet ends when the architect says `APPROVE` and the implementer closes with
`AGREE`. (PR review sequences these differently — see the PR-review section
below.)

### Rules of engagement

- Respond point-by-point to the previous entry; number your points.
- Be concrete: file paths, function names, exact commands — not vibes.
- Don't relitigate settled points; the budget is shared and small.
- Concede when the other side is right — say so and move on.
- Keep entries tight. Disagreement is welcome; padding is not.

### Implementer constraints

- **Before the plan (or the work order) is agreed: no code edits.** Reading
  the repo to inform critique is encouraged; your only write is your notebook
  entry. In PR review that means no edits until the work order is frozen —
  and none at all in the review-only shape.
- Implementation happens only inside the assigned worktree. Never leave it,
  never `git stash`, never touch the notebook's history.
- You run **sandboxed**: the worktree and the runs dir are writable, the rest
  of the filesystem is not. That matches the rule above, so ordinary work
  never notices it. If something you genuinely need is refused by the sandbox,
  **say so in your entry** — name the command and what it needed — and let the
  architect widen it. Do not route around a sandbox denial, and do not silently
  drop the piece of work that hit it.
- **Every `IMPLEMENTED` entry carries an evidence packet**: an annotated diff
  (or diff path), the exact commands you ran, their captured output, a
  per-finding/per-ID status, any deviations from the charter or work order,
  and known gaps — gaps hidden from the reviewer count as defects. The
  reviewer audits this evidence and re-runs the *named* verification
  commands; it does not re-derive the whole review from scratch. "Fixed"
  without artifacts is still never accepted.
- When fixing against a work order or finding list, answer it **item by
  item, by stable ID** (F1, F2, …), including items you chose not to do and
  why; silently skipping an agreed item is a defect. One commit per finding
  where practical. Don't change things no agreed item asks for — if you think
  something else needs doing, say so in your entry and let it be added first.

## Blind opening & merge iterations (Round 1 and merge iterations only)

When a turn prompt says you are drafting or merging **independently**,
whoever writes first would anchor the other, so the usual shared-notebook
flow is suspended for that round:

- **Do not read the notebook or the PR thread.** There is nothing there
  you're meant to see yet. Read the repo, the task, and (for a merger) both
  current drafts freely — but never the other side's *current-iteration*
  output before both exist.
- **Write to the file the prompt names**, not to the shared surface.
- **"Respond point-by-point to the previous entry" does not apply** to a
  Round 1 draft — there is no previous entry. Say what *you* think the
  design (or the review) should be.
- This blindness rule extends to every merge iteration, not just Round 1: you
  may see your own prior history (a resumed Codex session, for instance) but
  never the other side's merger for the *current* iteration until both exist.
- Both drafts (or mergers) are posted together once both exist, verbatim
  under their own signatures, so the record is honest about what each side
  actually thought — don't hedge toward what you guess the other side will
  say.
- From the round after each pair posts, normal rules resume and both sides
  critique the result.

## PR-review sequencing & attribution (PR mode only)

In **PR review**, both sides first `AGREE` on the finding list (reached
through the same blind-opening + merge-iteration machinery above, applied to
finding lists instead of plans — see references/pr-review.md), then `AGREE`
on the frozen work order (the agreed remedy for each finding), then the
implementer fixes (`IMPLEMENTED`, with its evidence packet) and a fresh
Sonnet reviewer checks the diff against that work order, finding by finding —
`REVISE` if anything is unfixed, fixed in a way that doesn't hold up, or out
of scope; `APPROVE` when it all matches. In the **review-only** shape there is
no work order and no `APPROVE` — nobody is approving code — and it ends when
both sides `AGREE` on the finding list.

- While Phase A is still converging on findings, nobody is implementing yet:
  the roles read `<Reviewer A|Reviewer B>`, with the Claude side as A. Once
  the work order is frozen and fixing begins, the roles revert to
  `<Architect|Implementer>` for the rest of the run. In the **review-only**
  shape they stay Reviewer A/B throughout.
- Post your entry as a PR comment via `gh pr comment`, opening with an
  attribution line naming your model, harness, and role, e.g. `*[GPT-5.1-codex
  · codex exec · duet implementer · <org>/<repo>:]*`. Both sides do this —
  the turn prompt must spell it out for Codex.
