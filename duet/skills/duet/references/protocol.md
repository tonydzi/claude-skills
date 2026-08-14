# Duet protocol

Two agents share one task. The **architect/reviewer** (a Claude model,
orchestrating) designs and reviews; the **implementer** (Codex) critiques the
design and writes the code. They converse by appending entries to a shared
notebook (or, in PR mode, posting PR comments). This file is the contract;
both sides receive it every turn.

## Entry format

```markdown
## Round <N>/<budget> — <Architect|Implementer>

<body>

VERDICT: <verdict>
— <actual model name> (<harness>, <architect|implementer>)
```

- **Append-only.** Never edit or delete another entry (or your own past ones).
  This includes the Round 1 blind drafts once posted — the merged proposal
  supersedes them in force, but their text stays as written.
- While PR-review mode is still converging on findings (Phase A), nobody is
  implementing yet: the roles read `<Reviewer A|Reviewer B>`, with the
  orchestrator as A. Once the work order is frozen and fixing begins, the
  roles revert to `<Architect|Implementer>` for the rest of the run. In the
  **review-only** shape they stay Reviewer A/B throughout.
- The `VERDICT:` line is the last line before the signature, exactly
  `VERDICT: <WORD>` at line start (it is machine-parsed).
- **Sign as the model you actually are** — e.g. `— Fable 5 (Claude Code,
  architect)`, `— GPT-5.1-codex (codex exec, implementer)`. If a different
  model is in the seat (say Opus orchestrating), the signature says so.
- One **round** = one architect entry + one implementer entry. The round
  budget is stated in the notebook header (default 15) and is shared across
  design and review. State your round number. The budget is a runaway-loop
  backstop, not a pace to fill — converge as fast as correctness allows.

## Verdicts

| Verdict | Who | Meaning |
|---|---|---|
| `PROPOSE` | either | Opening design or review — including a blind opening. |
| `REVISE` | either | I want changes — each one listed concretely. |
| `AGREE` | either | I sign off on the current plan/state. |
| `IMPLEMENTED` | implementer | Coding pass done: what changed, how verified. |
| `APPROVE` | architect | The code is good. |

Design phase ends when **both** sides have said `AGREE` on the same plan.
The duet ends when the architect says `APPROVE` and the implementer closes
with `AGREE`.

In **PR review**, both sides first `AGREE` on the finding list, then `AGREE`
on the frozen work order (the agreed remedy for each finding), then the
implementer fixes (`IMPLEMENTED`) and the architect checks the diff against
that work order, finding by finding — `REVISE` if anything is unfixed, fixed
in a way that doesn't hold up, or out of scope; `APPROVE` when it all matches.
In the **review-only** shape there is no work order and no `APPROVE` — nobody
is approving code — and it ends when both sides `AGREE` on the finding list.

## The blind opening (Round 1)

When a turn prompt says you are drafting **independently**, whoever writes
first would anchor the other, so the usual shared-notebook flow is suspended
for that one round:

- **Do not read the notebook or the PR thread.** There is nothing there yet
  that you're meant to see. Read the repo, the task, and the diff freely.
- **Write to the file the prompt names**, not to the shared surface.
- **"Respond point-by-point to the previous entry" does not apply** — there is
  no previous entry. Say what *you* think the design (or the review) should be.
- Both drafts and a merged proposal are posted together afterward, so your
  draft is on the record verbatim under your own signature. Write it to stand
  on its own; don't hedge toward what you guess the other side will say.
- From Round 2 on, the normal rules resume and both sides critique the merger.

## Rules of engagement

- Respond point-by-point to the previous entry; number your points.
- Be concrete: file paths, function names, exact commands — not vibes.
- Don't relitigate settled points; the budget is shared and small.
- Concede when the other side is right — say so and move on.
- Keep entries tight. Disagreement is welcome; padding is not.

## Implementer constraints

- **Before the plan (or the work order) is agreed: no code edits.** Reading
  the repo to inform critique is encouraged; your only write is your notebook
  entry. In PR review that means no edits until the work order is frozen —
  and none at all in the review-only shape.
- Implementation happens only inside the assigned worktree. Never leave it,
  never `git stash`, never touch the notebook's history.
- Every `IMPLEMENTED` entry states what changed (files), what was run to
  verify (lint/tests + results), and any known gaps — gaps hidden from the
  reviewer count as defects. When fixing against a work order, answer it
  **item by item, by ID**, including items you chose not to do and why;
  silently skipping an agreed item is a defect. Don't change things no agreed
  item asks for — if you think something else needs doing, say so in your
  entry and let it be added to the work order first.
- In PR mode, post your entry as a PR comment via `gh pr comment`, opening
  with an attribution line naming your model, harness, and role, e.g.
  `*[GPT-5.1-codex · codex exec · duet implementer · <org>/<repo>:]*`
