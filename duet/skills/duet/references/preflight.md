# Preflight, the round command, and permissions

Duet is hands-off after kickoff, so every environment question gets answered
**now**, in one handshake, instead of surfacing as a denied command in round 7
with a half-built worktree on the floor. Run these before the announcement.
Steps 1–2 can run while the worktree is being set up; step 3 needs it.

## Preflight — settle auth and permissions before Round 1

1. **Codex is there**: `codex --version`. If it fails, stop and say so — never
   improvise a substitute implementer.
2. **How Codex authenticates**: `codex login status`.
   - `Logged in using ChatGPT` → a stored subscription login exists: **keep**
     the `env -u CODEX_API_KEY -u OPENAI_API_KEY` prefix, so a long
     implementation run bills the subscription instead of the API.
   - Not logged in (an API key in the environment is doing the work) →
     **drop** the prefix, or `codex exec` fails to authenticate once the keys
     are stripped.

   Decide once, here, and use the same prefix for every turn of the run.
   **Never read `~/.codex/auth.json`** or any other credential file to answer
   this. `codex login status` reports the same thing without touching a
   secret, and "read the file holding my API keys" is exactly the shape a
   permission classifier denies — correctly.
3. **Smoke test — this doubles as the permission handshake.** Once the
   worktree and runs dir exist, run the *real* command shape once with a
   throwaway prompt:

   ```bash
   env -u CODEX_API_KEY -u OPENAI_API_KEY \
     codex exec -s workspace-write --add-dir "<runs>" \
     -c sandbox_workspace_write.network_access=true \
     -C "<worktree>" -o "<runs>/smoke.md" \
     "Reply with exactly: DUET_SMOKE_OK" </dev/null
   ```

   **Flag for flag, this is the command the rounds use** — only `-o` and the
   prompt differ. That is the whole point: the approval granted here covers
   every round that follows. A smoke test in a *simpler* shape is worse than
   none, because it proves the rounds will be allowed when they won't.

   - **`DUET_SMOKE_OK`** → go. Announce and start Round 1.
   - **Auth error** → step 2 read the environment wrong. Flip the `env -u`
     decision, retry once, and use whichever worked for the rest of the run.
   - **Denied by the permission system** → stop *here*, before any design
     work. Show the user the allowlist under **Permissions**, ask once, then
     resume. Do **not** quietly retry in a weaker shape to slip past the
     denial — a Codex that cannot write the worktree produces a run that looks
     alive and builds nothing. If the user declines, remove the worktree and
     stop.
4. **Session-economy probe** (see **Codex session economy** below): with the
   version/auth already known, check once whether the installed Codex CLI
   supports resuming a session non-interactively — try `codex exec resume
   --help`, and check `codex exec --help` for a `resume`/`--last` affordance.
   Record the answer in `architect-state.md`; every later turn reads it
   instead of re-probing.

**Never wrap a Codex call in `timeout`** — it is not present on stock macOS, so
the whole command dies with `env: timeout: No such file or directory` and reads
like a Codex failure. Long turns go to Bash `run_in_background` and get
monitored, which is the rule for implement turns anyway.

## The round command shape

Every round after the smoke test uses this exact shape — only `-o` and the
prompt change:

```bash
env -u CODEX_API_KEY -u OPENAI_API_KEY \
  codex exec -s workspace-write --add-dir "<runs>" \
  -c sandbox_workspace_write.network_access=true \
  -C "<worktree>" -o "<runs>/round-<n>-codex.md" \
  "$(cat "<runs>/round-<n>-prompt.md")" </dev/null
```

Every part of that line earns its place — use the same shape every round, so
one approval covers the whole run:

- **`-s workspace-write`** confines Codex's writes to its working root. That
  is the actual safety property duet wants (Codex touches the throwaway
  worktree, nothing else), and it reads to a permission classifier as an
  ordinary sandboxed tool call rather than a request to switch safety off.
- **`--add-dir "<runs>"`** because the runs dir sits outside the worktree and
  Codex writes there — the `-o` transcript, and prompts it may read. Without
  it those writes fail inside the sandbox (verified 2026-08-14).
- **`-c sandbox_workspace_write.network_access=true`** because
  `workspace-write` blocks the network by default, and implement turns
  routinely need `gh`, `git fetch`, or a package install (verified
  2026-08-14). Keep it by default. The only case for dropping it is a
  notebook-medium run where Codex neither builds anything nor posts to a PR —
  note that PR mode always needs it, review-only included, since Codex posts
  its own entries with `gh pr comment`.
- **`</dev/null`** is required: with an open non-tty stdin, `codex exec`
  blocks forever waiting for stdin to close (verified 2026-08-11).
- **The `env -u` prefix** is whatever **Preflight** settled — keep it for a
  subscription login, drop it for API-key-only auth. Don't re-derive it
  mid-run.

**Escalation is a last resort, never a starting point.** Do not open with
`--dangerously-bypass-approvals-and-sandbox`. Reach for it only if a turn
actually fails on something `workspace-write` cannot cover — Codex needing to
write outside the worktree, drive Docker, or reach a system path. Then stop,
name the exact command that failed and why, get the user's OK, and use it
from that turn on. Starting there is what gets duet denied on round one in
auto-approve sessions.

## Codex session economy

- The probe at preflight step 4 decides this for the whole run.
- **If resuming is supported**: run one persistent Codex session across
  critique → implementation → fixes; per-turn prompts send only the new task
  + state delta, not the full protocol + history.
- **If not supported**: fall back to fresh `codex exec` per turn, but send a
  protocol *slice* — Core, plus whichever of the blind-opening/merge-iteration
  or PR-review slices apply this turn — not the whole `protocol.md` file (see
  the "what to send when" note at the top of that file).
- Either way the smoke test above still runs in the exact per-turn command
  shape that will actually be used.

## Permissions

Duet asks for the least it can, and asks for it once. A colleague on
auto-approve should see at most one prompt, at the smoke test.

| Command | Why duet needs it |
|---|---|
| `codex login status` | which auth Codex uses — reads no secret |
| `codex exec -s workspace-write …` | the implementer's turn, once per round |
| `git worktree add` / `remove` | the isolated checkout duet works in |
| `gh pr view` / `gh pr comment` | PR mode only — the thread is the notebook |

If the smoke test comes back denied, hand the user exactly this, say in one
line what each entry buys, and let them add it via `/permissions` or
`.claude/settings.json`:

```json
"permissions": {
  "allow": [
    "Bash(codex login status)",
    "Bash(env -u CODEX_API_KEY -u OPENAI_API_KEY codex exec:*)",
    "Bash(codex exec:*)",
    "Bash(git worktree:*)",
    "Bash(gh pr view:*)",
    "Bash(gh pr comment:*)"
  ]
}
```

Report the denial as what it is — the harness's permission gate, not a refusal
by the model and not a broken Codex. Users read "denied" as "Claude won't do
it" and argue with the wrong party.
