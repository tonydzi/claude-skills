---
name: share-fix
description: "You fixed something at home: find 3-5 live threads where other people hit the same symptom and hand them the verified fix, one tailored comment each. Triggers: /share-fix, share this fix, who else has this bug, give the fix to the world."
---
<!-- llm-backend-ok: MVP public recipe for the reader's own agent; documents gh commands, not a sender -->

# share-fix: a fixed bug at home is a gift for strangers

Every bug you fix in your own setup was probably hit by someone else first. They wrote
an issue, nobody answered, and they are still stuck. Your fix, with evidence, is the most
useful comment that thread will ever get.

Our numbers over the lane's lifetime: 379 comments in 313 threads. A person replied
directly in about 11.5% of threads, the maintainer personally in about 4.5%. The reply
rate held steady as volume grew (12% at ~60 comments a week, 10% later). What killed
volume was not people ignoring us. It was our own candidate queue running dry.

## Step 0. Two gates, both must be yes

1. **Portable:** can a stranger reproduce the symptom without your private scripts,
   infrastructure or accounts?
2. **Clean:** the fix you share carries no secrets, private names or internal paths. A
   script for the world is a new, clean file written for the world.

Either is no → stop here and say so.

Also: if the bug is in someone else's tool and you have no fix yet, file one bug report
first (search open *and* closed issues for duplicates before you do).

## Step 1. Search by the exact symptom

```bash
gh search issues --state open --sort updated --limit 30 -- "exact error text" toolname
gh search issues --state open --sort comments --limit 30 -- word1 word2 word3
```

Search gotchas we paid for:
- One long quoted argument is an exact phrase. Pass words separately or you get a false zero.
- Always filter `--state open`. Without it, results drift into the archive.
- A bare number in the query is read as an issue number. Give it context.
- `--sort comments` combined with `repo:` inside the query string returns junk. Use `--repo`.

## Step 2. Pick the 3-5 best live threads

- Live = different human authors commenting recently. The comment counter lies both ways;
  look at who wrote, not how many.
- Your fix removes *their* symptom, judged by *their* logs. A similar-looking error with a
  different cause is a skip.
- A thread where you already wrote is a skip. Dedup by opening the thread, not by the
  search index; the index misses fresh comments.
- At most one comment per foreign repo per day. Never paste the same text into many threads.

## Step 3. One comment per thread, tailored

1. If an agent writes it, the first line says who is writing.
2. How your case matches theirs, and how it differs. Honestly.
3. The fix, as something they can apply in a minute: a command, a config line, a patch.
4. The evidence: before/after, a log line, a version.
5. One question back to the author. Threads live on questions.

On a PR thread, use `gh pr review <N> -R owner/repo --comment --body-file c.md`.
On an issue, `gh issue comment <N> -R owner/repo --body-file c.md`.
In repos above ~1000 stars, no links to your own projects: inline what they need.

## Step 4. Watch and record

Write down each URL. Come back in 1-3 days. A reply is a conversation to continue
(`reply-pr-reviewers`), silence is fine. Record the reply rate per lane; it tells you
whether the lane works, and a falling volume tells you to fix your queue, not the people.

## Feeding the queue

The cheapest source of candidates is your own merged fixes: every merged `fix:` PR from
the last 30 days is a symptom someone else probably has. Pull them in nightly so the
lane never starves.

```bash
gh search prs --author @me --merged --merged-at ">=$(date -v-30d +%F 2>/dev/null || date -d '-30 days' +%F)" \
  --json title,url -q '.[] | select(.title|test("fix";"i")) | "\(.url) \(.title)"'
```

---
Part of the `github-growth` plugin by Anton Dziatkovskii and Mycroft (his synthetic AI cofounder). More: github.com/tonydzi
