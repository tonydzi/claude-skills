# Multi-part projects

For a parent orchestrator splitting one project into parts, each part run as
its own duet. The motivating failure was ~sequential parts, each spun up with
its own fresh budget and an inherited expensive model, draining the account
with no shared accounting — concurrency wasn't the culprit; unbounded
aggregate spend was. This reference is how a parent avoids that.

## Invocation shape

Nested runs are invoked `project <id> part <i>/<n>`, must name the governing
spec file/section, and resolve to **Implement mode** — a parent orchestrator
passes mode explicitly rather than relying on the usual ambiguous-invocation
inference (SKILL.md's **Parse the invocation**).

## Classify each part before launching it

| Classification | When | Treatment |
|---|---|---|
| **Architectural** | Changes module boundaries, public interfaces, persisted data, dependencies, concurrency, security boundaries, or user-facing flows | Blind openings + merge, constrained by the top-level design |
| **Routine** | Implements already-settled decisions | No blind openings, no new Fable planning: Codex plans, a Sonnet round checks conformance to the governing spec |
| **Uncertain** | — | Treat as architectural |

Directive `architecture: yes|no|auto`, default `auto`.

## Project ledger

One shared file, its path passed to every part — suggested location
`~/.claude/duet/<repo>/projects/<id>/ledger.md`:

- **Round pool**: total/remaining, default **6 × parts + 4**.
- **Fable-checkpoint pool**: total/remaining, default **5**, with **one
  checkpoint reserved** for a final cross-part coherence review that routine
  parts may not consume.
- **Per-part status + usage**, degradation decisions.

Parts draw from the pools and return unused allocation; a nested duet must
never silently create its own independent budget.

## Pacing

Sequential by default. If parallel, max 2–3 parts at once, and pause between
batches to read the ledger before launching more.

## Circuit breakers

Recorded in the ledger and the final report as they trip:

| Pool consumed | Effect |
|---|---|
| **60%** | Per-round escalation to Opus disabled except for genuine disagreement |
| **80%** | Delta-only review everywhere; no new architectural part starts |
| **95%** | Finish the part in flight, start nothing new, preserve the reserved cross-part checkpoint, stop cleanly |

Never report a partially-completed part as done. The final report says which
parts were reviewed at full depth and which degraded, and why.

## Instrumentation

Each part's `usage.jsonl` rollup (SKILL.md's **Instrumentation**) is copied
into the project ledger, so the parent's final report can total spend across
every part without re-deriving it.
