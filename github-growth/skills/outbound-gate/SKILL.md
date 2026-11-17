---
name: outbound-gate
description: "The PR is written: should it go out? Checks whose request it closes, duplicates, your open-PR count in that repo, fix vs feature, bot review threads, sign-off. Triggers: /outbound-gate, PR ready should I send, one more PR into this repo. Not for choosing where to start (use issue-match)."
---

# outbound-gate: ship it, hold it, or turn it into a comment

Run this on a finished change, right before `gh pr create`. Each check is one
question with a yes/no answer. Any "no" means hold, rework, or downgrade to a comment.

## 0. Interruption check (always first)

Does a human already wait on you somewhere? A maintainer reply on one of your open PRs
outranks any new PR.

```bash
gh search prs --author @me --state open --json url,commentsCount,updatedAt \
  -q 'sort_by(.updatedAt) | reverse | .[:15][] | "\(.updatedAt[:10]) \(.commentsCount) \(.url)"'
```

Open the ones updated after your last push. Answer first (see `reply-pr-reviewers`), ship later.

## 1. Anchor: whose live request does this close?

Write the anchor down as a link: an issue opened or confirmed by a maintainer, a
"PR welcome", an assignment, or a maintainer comment asking for exactly this.

- No anchor → this is a cold PR. Turn it into an issue comment with your evidence and a
  plan, and wait for a yes.
- Work bigger than one PR → title each part "Part k of N of #issue", and put any risky
  flag or default switch in the **last** part only.

## 2. Fix or feature?

Judge by what the diff does, not by the title. New behavior is a feature even if the
title says "fix".

Our measurement over two months of outside contributions: features merged 3 of 40,
fixes merged 30 of 67. Features go only where a maintainer asked for them, and in large
repos only on explicit invitation.

## 3. Duplicate check (ask the threads, not only the index)

```bash
gh pr list -R owner/repo --state all --search "keyword in:title" --limit 20
gh search prs --repo owner/repo -- keyword1 keyword2
```

Search can miss fresh PRs. Open the anchor issue and read its linked PRs and cross-references.
Someone else's open PR for the same bug → review theirs instead of sending yours.

## 4. How many of yours are already open there?

```bash
gh search prs --author @me --repo owner/repo --state open --json url -q length
```

- Two or more open in one large repo → hold. Get one merged, then send the next.
- Never fan out a batch of same-shaped PRs into one repo. One, merged, then the rest.

## 5. Proof it works

- Test is red before your change and green after, run in *their* toolchain. Put the log
  in the PR body.
- Every bot reviewer thread (CodeRabbit, Greptile, Copilot, Cursor Bugbot and so on) is
  closed by a commit or a reply before you ask a human to look. Maintainers do ask for this.
- A question to the maintainer is phrased as "Option 1 / Option 2, I suggest 1 because...".

## 6. Paperwork

- Commit author is a real account with a verified email; DCO `Signed-off-by` if required;
  CLA signed if the bot asks.
- In repos above ~1000 stars or corporate ones: no links to your own projects in the PR.
  Only what a reviewer needs to verify the change (repro, upstream issue, spec).
- If an AI wrote the change, say who wrote it in the PR body. Do not hide it and do not
  write "no human reviewed this".

## Ship and watch

After `gh pr create`, record the URL somewhere you will read tomorrow, and run
`contrib-watch` and `reply-pr-reviewers` daily. A PR without a watcher is a PR that
dies on the first review comment.

---
Part of the `github-growth` plugin by Anton Dziatkovskii and Mycroft (his synthetic AI cofounder). More: github.com/tonydzi
