---
name: reply-pr-reviewers
description: "Answer reviewers on your PRs in other people's repos: find where the ball is yours, do what they asked, reply short, and stop bumping dead threads. Triggers: /reply-pr-reviewers, answer reviewers, review debt, ball is in our court."
---
<!-- llm-backend-ok: MVP public recipe for the reader's own agent; documents gh commands, not a sender -->

# reply-pr-reviewers: answer the people who already agreed to talk

A live reviewer is worth more than a new PR. Silence in a live thread is a failure, not
caution. Threshold: 24 hours.

## Step 1. Where is the ball yours?

```bash
ME=$(gh api user -q .login)
gh search prs --author $ME --state open --sort updated --limit 50 \
  --json repository,number -q '.[] | "\(.repository.nameWithOwner) \(.number)"' |
while read R N; do
  last=$(gh pr view $N -R $R --json comments,reviews -q \
    '[(.comments[]|{t:.createdAt,a:.author.login}), (.reviews[]|{t:.submittedAt,a:.author.login})]
     | map(select(.a|test("\\[bot\\]|bot$")|not)) | sort_by(.t) | last | .a // "none"')
  [ "$last" != "$ME" ] && [ "$last" != "none" ] && echo "BALL OURS  $R#$N  (last: $last)"
done
```

Also read inline review threads (`gh api repos/$R/pulls/$N/comments`). They are a
separate surface and the one most often missed.

Traps:
- "Last speaker is not me" is not "they replied to me". It may be a bot or a third person.
- Before you answer about what your PR does, re-read your own PR body and diff. Memory of
  what you shipped drifts.
- The reviewer closed the conversation themselves ("closing in favor of #123"): that is
  not debt. Thank once, take the pointer, move on.

## Step 2. The three-day rule and the silence counter

Across 54 of our PRs, every merge happened within 0 to 3 days of the last exchange. After
that, never. So:

- Ball is theirs and silence is longer than 3 days: the thread is a grave. Do not bump it.
  At most one polite bump per PR while it is still alive, never a second one.
- Count your own comments after the last human from their side. 0: reply with the fix.
  1: the next comment is the last one and only with new value after 7+ days (a release
  changed things, your own claim went stale). 2 or more: stop. Either close it with a
  one-line thanks, or let it sit.

## Step 3. Do what they asked

- Fix in code, push, and reply with what changed and where (commit SHA, file:line).
- A number in your reply comes with its source and the date it was measured.
- Question back to them: "Option 1 / Option 2, I suggest 1 because...".
- Close every bot review thread too, by a commit or a reply. Maintainers ask for this.
- Before force-push, check the repo's rules. Some forbid rebasing an open PR.

## Step 4. Reply short and verify it landed

```bash
gh pr comment $N -R $R --body-file reply.md         # a reply in the conversation
gh pr review  $N -R $R --comment --body-file r.md   # when you review someone else's PR
```

`gh pr comment` does not create a Review object; on someone else's PR, a review is what
counts as review work. Read your comment back from the server after posting and compare
it to what you meant to send.

## Step 5. Your PR got closed

Before you file it away, read the closing comment for a pointer: a board, another repo,
another issue. That pointer is the next door, named by the person who merges there.
Feed it to `issue-match`.

---
Part of the `github-growth` plugin by Anton Dziatkovskii and Mycroft (his synthetic AI cofounder). More: github.com/tonydzi
