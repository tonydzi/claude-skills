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

To back out: `/plugin uninstall duet@morphenius`.

Then invoke it as **`/duet <task or PR>`**. Installed from a plugin it is formally
named `duet:duet` (plugin name, then skill name), and that fully-qualified form
works too — but plain `/duet` routes to it.

## Updating

Installed plugins are **copied into a cache and pinned to a git commit**, so
refreshing the marketplace alone doesn't move an already-installed plugin. To
actually pick up new changes, reinstall:

```
/plugin marketplace update morphenius
/plugin uninstall duet@morphenius
/plugin install duet@morphenius
```

Same three lines whether you're a teammate pulling from GitHub or the author
pulling from a local clone.

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

**No setup knobs.** Duet runs a short preflight before it starts: it checks Codex is
installed, asks `codex login status` whether you're on a subscription or an API key,
and adjusts the command it uses accordingly. Then it runs one throwaway Codex call as
a smoke test. If anything is wrong, you hear about it in the first few seconds rather
than forty minutes in.

**Permissions.** Duet is built to run under auto-approve. Codex is called with
`-s workspace-write`, which lets it edit the throwaway worktree and nothing else —
so it reads as a normal sandboxed command, not a request to switch safety off. The
smoke test uses the exact command shape the real rounds use, so approving it once
covers the whole run. If your setup denies even that, duet stops right there and
prints the allowlist to paste into `/permissions` — it will not quietly downgrade
itself into a run that looks alive and builds nothing.

**Safety note.** Codex writes only inside a throwaway git worktree under
`~/.claude/duet/`, on its own branch — never your working tree. It has network
access (implementation needs `gh`, `git fetch`, package installs). Duet escalates to
`--dangerously-bypass-approvals-and-sandbox` only if a turn genuinely can't proceed
without it, and only after telling you why.

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

## Authoring (for me)

This machine has the marketplace registered as a **directory** source pointing at
the local clone (`~/Projects/claude-skills`), not at GitHub — so the edit loop is
edit → commit → the three update lines above, with no push required to test.
Push when it is ready for everyone else.

The other machine and teammates register it as a GitHub source:
`/plugin marketplace add morphenius/claude-skills`.
