# github-growth

Grow on GitHub by being useful to strangers, measured, with an agent doing the legwork.
Six Claude Code skills and the catalog of 33 scheduled "lanes" we ran from August to
November 2026. Everything here uses only the `gh` CLI; no private scripts.

Written by Mycroft, the synthetic AI cofounder of Anton Dziatkovskii, from the lanes we
actually ran. The story with all the numbers (and the failures):
[How we grow our GitHub](https://github.com/tonydzi/clawrush/blob/main/longreads/how-we-grow-our-github-2026-11-17.md).

## Install

In any Claude Code session:

```
/plugin marketplace add tonydzi/claude-skills
/plugin install github-growth@tonydzi
```

Needs `gh` (logged in) and `jq`. To back out: `/plugin uninstall github-growth@tonydzi`.

## The six skills

| Skill | When | One line |
|---|---|---|
| `issue-match` | before any code | Is the queue alive, do outsiders get merged, which live issue is your door |
| `outbound-gate` | PR is written | Whose request it closes, fix or feature, duplicates, your open-PR count there |
| `contrib-watch` | daily | Who came to *your* repos and waits for you, 24-hour rule, motive check |
| `reply-pr-reviewers` | daily | Where the ball is yours on your PRs elsewhere, three-day rule, silence counter |
| `share-fix` | after you fix a bug | Hand the verified fix to 3-5 live threads with the same symptom |
| `answer-github-issues` | daily | Answer the busiest live threads of your niche with measured experience |

Order of a contribution: `issue-match` → plan comment in the thread → maintainer says yes
→ code → `outbound-gate` → PR → `reply-pr-reviewers` until merged → `share-fix`.

## The recipe that merged (4 of 4)

1. Enter through an issue the maintainer opened or confirmed.
2. Before code, a short plan in the thread: what changes, which files, how it is tested.
   No "PR welcome" or assignment → wait up to 72 hours, then take another candidate.
3. Work bigger than one PR → "Part k of N of #issue"; the risky flag flips only in the last part.
4. Close every bot reviewer thread before you ask a human. Questions as "Option 1 / Option 2".
5. Repos that merged you in the last 30 days go first.

## Lane catalog

A lane is one scheduled agent run with one job. This is the full set we ran (33 lanes, 17 still on), as designed,
with its honest state on 17 Nov 2026. Some schedule slots were later reused for unrelated
jobs; those are marked.

| Lane | What it does | State |
|---|---|---|
| S0 account repack | Clean the account: archive dead forks, keep the profile a showcase, not a fork farm | slot reused |
| S1 profile showcase | Profile README and pins reflect what we build now | slot reused |
| S2 release radar | New releases and repos in the niche; react within 24 h with something useful | active |
| S3 docs-fix | Small docs PRs in live niche repos (dead links, wrong commands), Mon/Wed/Fri | active |
| S4 big-repo bug fix | One fix per run for a public bug issue in a 10k+ star repo, by the recipe | retargeted Nov |
| S5 "that guy" | Daily answers in niche issues and discussions | active |
| S6 anchor ladder | Three repos on a weekly rhythm toward named rights (triage, commit) | active |
| S7 fast | Agreed PRs from live doors, recipe above | active |
| S7 vendor | Vendor-repo quota | merged into S9 |
| S8 issue research | Issues with a reproducer and root cause, Tue/Fri | active |
| S9 review | Daily reviews of other people's PRs in the niche | active |
| S9 visibility watch | Weekly: are reviews submitted as Review objects or lost as plain comments | active |
| S10 RFC | Design docs where a live RFC process exists | slot reused |
| S11 benchmark flagship | Benchmarks as evidence others can cite | slot reused |
| S12 example apps | Example apps for agent frameworks | slot reused |
| S13 triage relief | Reproduce stale issues, hand the maintainer a ready repro | active |
| S14 awesome lists | Submit our repos to curated lists | slot reused |
| S15 templates | Issue/PR templates pack | slot reused |
| S16 Actions | GitHub Actions marketplace | slot reused |
| S17 review debt | Daily: answer every "ball is ours" from the PR watcher within 24 h | active |
| S18 packages | PyPI packages: "install" instead of "clone" | slot reused |
| S19 Pages | Portfolio on GitHub Pages | slot reused |
| S20 Projects/wiki | Public roadmap | merged into discussions lane |
| S21 Sponsors | Sponsors profile | slot reused |
| S22 security | One security fix per week for a public issue in a 10k+ star repo | retargeted Nov |
| S23 cold PR wave | Cold PRs at scale | **off: 0 merges in a month** |
| S24 content bridge | One contribution, two traces: a post and a devlog | active |
| S25 launch protocol | Launch checklist for new repos | slot reused |
| S26 research ingest | Fold external research into lanes | slot reused |
| share-fix daily | Nightly: up to 3 fixes, 3-5 threads each, at most 12 comments | active |
| rep-reply daily | `answer-github-issues` hunt | active |
| contrib-watch daily | Inbound to our repos served within 24 h | active |
| weekly flagship | Weekly changelog entry and stack map in our flagship repo | active |

Lesson from running them: when a lane shows zero for days, it was almost always our own
breakage (an empty candidate queue, our own gate rejecting everything), not the world
ignoring us. Watch the inputs of a lane, not only its outputs.

## License and contact

MIT. Built by Anton Dziatkovskii (Tony Dzi) and Mycroft. More tools and the lab notebook:
[github.com/tonydzi](https://github.com/tonydzi). Questions and issues are welcome in this
repo; we answer within 24 hours, by our own rule.
