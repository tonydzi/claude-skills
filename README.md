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

Several models work a task together while you go do something else. Every
agent gets an explicit model, tiered to the work: Fable drafts blind and
makes the judgment-call checkpoints, Sonnet does the routine per-round
design/review/merge work, Haiku handles mechanical chores, and Codex designs,
critiques, and writes the code. They go around until the architect approves
and Codex agrees — or until the round budget (default 8; 12 for PR
review-then-fix) runs out, at which point it stops and explains where they
got stuck in plain language.

The whole exchange is written down as it happens — a notebook file in the
worktree, or PR comments if you point it at a PR — so you can read the
argument afterward rather than taking the result on faith.

**Three modes**, chosen automatically from how you invoke it:

- **Spec** — designing something new. Both sides draft their opening *blind*
  (neither sees the other's first), then Sonnet and Codex iterate up to 3
  independent merge attempts until they converge, and a Fable end-check asks
  "what did the simpler models miss?" before issuing the charter. Nobody
  gets anchored by whoever typed first.
- **Implement** — a plan already exists; skip straight to building it, with
  a Fable architectural pass once the loop converges.
- **PR review** — review someone else's PR. Blind openings again (Fable is
  the blind Claude-side reviewer), findings converge the same iterated-merge
  way, and disputed findings are reported *as* disputed rather than papered
  over.

A parent project can also split a large piece of work into parts and run each
as a nested duet against a shared round/Fable-checkpoint budget, so a
multi-part job can't silently drain the account one part at a time.

**Examples**

```
/duet design a retry policy for the ingest worker
/duet implement the plan in docs/rfcs/retry-policy.md
/duet review PR 412
/duet budget 25 refactor the scheduler; figure it out, Opus breaks ties
/duet project ingest-rewrite part 3/6, spec docs/rfcs/ingest-rewrite.md
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
  skills/duet/SKILL.md             the skill core (model table, the loop, budgets)
  skills/duet/references/          protocol, preflight, PR-review, multi-part
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
