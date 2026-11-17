---
name: contrib-watch
description: "Who came to your own repos and is waiting for you: issues, PRs, inline review comments, promises you made in threads. 24-hour rule. Triggers: /contrib-watch, who wrote to us on GitHub, inbound debt, unanswered issues on my repos."
---

# contrib-watch: people who came to you first

A stranger who opens an issue or a PR on your repo has already spent their time on you.
That is the warmest contact you will ever get, and it closes from silence, not from
disagreement. Threshold: **24 hours**. Longer is red.

We added this lane after measuring a four-day silence. The cause was not "busy". The
cause was that nothing told us anyone had written.

## Step 1. Take the picture (three surfaces, not one)

```bash
ME=$(gh api user -q .login)
# 1. Issues and PRs on your repos where the last word is not yours
gh search issues --owner $ME --state open --sort updated --limit 50 \
  --json url,updatedAt,author,commentsCount
gh search prs --owner $ME --state open --sort updated --limit 50 \
  --json url,updatedAt,author,commentsCount
# 2. Inline review comments live behind a different endpoint
for r in $(gh repo list $ME --limit 200 --json name -q '.[].name'); do
  gh api "repos/$ME/$r/pulls/comments?sort=updated&direction=desc&per_page=5" \
    -q '.[] | select(.user.login != "'"$ME"'") | "\(.updated_at) \(.html_url)"' 2>/dev/null
done
```

For each hit, open the thread and check who spoke last. Count bots separately: a bot
comment does not hand the ball to you, and it does not hand it back either.

Three traps we hit:
- **The conversation moved.** "Last word is theirs" in the issue, but the answer is in a
  linked PR. Check cross-references before you call it debt.
- **The mirror case.** The last word is yours, and it is a promise ("I will run the
  experiment tomorrow"). That is debt too. Only a done fact closes it, not a second promise.
- **A click, not a reply.** "I will merge when CI is green" and CI went green hours ago.
  The move is yours, and it is a button.

## Step 2. Check the motive before you celebrate

| Looks like | What it means | Your move |
|---|---|---|
| Few repos, many touches in yours | Real user | Answer today, offer triage rights or co-authorship |
| 40+ PRs across 20 repos, patches are real | Reputation pipeline | Answer the same day; they go deep where people answer |
| 40+ PRs, text generated, facts do not match | Someone's agent on random targets | Honest no with a short technical reason |
| Bio sells their product, questions lead to "you need our layer" | Vendor | One answer on substance |
| 100+ public repos and under 5 followers | PR farm | Short code-level answer, no rights, no promises |

First question about everyone: **did we knock on their door first?** A reply to your own
outreach is not an inbound signal.

## Step 3. Verify the claim, do not trust it

For a PR to your repo, check three things:
1. The patch does what it says.
2. Its tests fail when you revert the fix. If they stay green, they test nothing.
3. The patch is not stale. Your repo moved while it waited; rebase and rerun.

## Step 4. Reply

1. If you are late, say so and say why, concretely.
2. Name what exactly they caught, in your own words. That proves you read the code.
3. Verdict: merged / needs a rebase (say how) / not taking it, and why.
4. Next step with a date, and one open question.
5. If an agent writes the reply, the first line says who is writing. Never "no human read this".

## Bounds

Replies on your own repos are yours to send. Merging other people's code into your repo
goes through your normal tests and a leak scan.

---
Part of the `github-growth` plugin by Anton Dziatkovskii and Mycroft (his synthetic AI cofounder). More: github.com/tonydzi
