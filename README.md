# morphenius/claude-skills

A small [Claude Code](https://claude.com/claude-code) plugin marketplace. Everything
here is **opt-in** — nothing installs itself, and nothing changes for you until you
run the two commands below.

## Install

In any Claude Code session:

```
/plugin marketplace add morphenius/claude-skills
/plugin install duet@morphenius
```

To pick up later updates: `/plugin marketplace update morphenius`.
To back out: `/plugin uninstall duet@morphenius`.

## Plugins

| Plugin | What it does |
| ------ | ------------ |
| `duet` | `/duet <task or PR>` — Claude designs and reviews, Codex implements, hands-off until they agree |

### duet

Two models work a task together while you go do something else. Claude (the
orchestrator) writes the design; Codex critiques it and writes the code; Claude
reviews the diff; they go around until Claude approves and Codex agrees — or
until the round budget (default 15) runs out, at which point it stops and
explains where they got stuck in plain language.

The whole exchange is written down as it happens — a notebook file in the
worktree, or PR comments if you point it at a PR — so you can read the argument
afterward rather than taking the result on faith.

**Three modes**, chosen automatically from how you invoke it:

- **Spec** — designing something new. Both sides draft their opening *blind*
  (neither sees the other's first), then a third fresh context merges them.
  Nobody gets anchored by whoever typed first.
- **Implement** — a plan already exists; skip straight to building it.
- **PR review** — review someone else's PR. Blind openings again, no code
  written, and disputed findings are reported *as* disputed rather than
  papered over.

**Examples**

```
/duet design a retry policy for the ingest worker
/duet implement the plan in docs/rfcs/retry-policy.md
/duet review PR 412
/duet budget 25 refactor the scheduler; figure it out, Opus breaks ties
```

**Requirements**

- The [Codex CLI](https://developers.openai.com/codex/cli), installed and logged in
  (`codex login`). Duet calls `codex exec` once per round.
- Claude Code as the orchestrator.

**One setup knob.** The Codex command is prefixed with
`env -u CODEX_API_KEY -u OPENAI_API_KEY`, which strips inherited API keys so Codex
uses your subscription login instead of billing the API. If you authenticate Codex
by **API key only**, delete that prefix from `SKILL.md` or the first Codex turn will
fail on auth. Everything else works out of the box.

**Safety note.** Codex runs with `--dangerously-bypass-approvals-and-sandbox`, but
confined to a throwaway git worktree under `~/.claude/duet/` on its own branch —
never your working tree. Read the source before you run it if that trade isn't one
you want.

## Layout

```
.claude-plugin/marketplace.json    the marketplace manifest
duet/
  .claude-plugin/plugin.json       the plugin manifest
  skills/duet/SKILL.md             the skill itself
  skills/duet/references/          the protocol both sides follow each turn
```

Adding another plugin means a new top-level directory plus one entry in
`.claude-plugin/marketplace.json`.
