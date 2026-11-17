---
name: answer-github-issues
description: "Answer live GitHub threads in your niche with your own measured experience: hunt the busiest live thread first, gate for quality, reply with facts and one runnable piece. Triggers: /answer-github-issues, find threads for our experience, grow reputation on GitHub."
---
<!-- llm-backend-ok: MVP public recipe for the reader's own agent; documents gh commands, not a sender -->

# answer-github-issues: be the person whose comment helped

Reputation on GitHub comes from comments that saved someone an afternoon. This lane
looks for live threads where your measured experience is the missing piece.

## Step 0. Where to look first

1. **Shelf x request:** a maintainer explicitly asks for something you already have
   (a benchmark, a repro, a tool). Highest value, rarest.
2. **The busiest live thread of your niche.** One answer in a thread followed by 70 people
   beats ten answers in empty threads.
3. **Stop signals:** a maintainer said "please stop", locked the thread, or asked for no
   AI-written comments. That beats any mention or invitation.

```bash
gh search issues --state open --sort comments --limit 20 -- topic1 topic2
gh search issues --state open --sort updated --limit 30 --label "help wanted" -- topic1
```

## Step 1. Read the whole thread before answering

Read every comment, not the first and the last. Half the time the answer is already in
comment 14, or the maintainer narrowed the scope in comment 9. Answering what was already
answered is noise.

## Step 2. Quality gate (all must pass)

- You have a fact from your own run: a number, a log, a date. Not an opinion.
- The thread is live (human activity in the last weeks).
- Your case matches theirs, and you can say how it differs.
- If you link an executable artifact, you cloned it fresh from its public home and ran it
  in the same session. We once almost sent 74 people to a script that crashed on real data.
- No template sentences, no praise openers, no "great question".

## Step 3. The answer

1. Language of the thread.
2. If an agent writes it, the first line says who is writing.
3. Where your case matches and where it differs.
4. Your experience as facts, with dates and numbers.
5. One runnable piece (command, snippet) the reader can apply in a minute.
6. At most one link, and only if it carries the evidence. In repos above ~1000 stars,
   inline instead of linking to your own work.
7. A question back to the author.

The status of your own fix is a number too. If the bug came back this morning, write
"not proven yet, regressed today". That honesty was the strongest line in one of our
best-received answers.

## Step 4. Record

URL, date, thread size, and later whether anyone replied. Without that log you cannot
tell a working lane from a habit.

---
Part of the `github-growth` plugin by Anton Dziatkovskii and Mycroft (his synthetic AI cofounder). More: github.com/tonydzi
