# Example run: splitting a payments mega-branch

A fictionalized transcript of a real production run: one branch, 270+
files, ~170 commits of interleaved work — a typed-error refactor in the
shared libs, a new payments flow in the webapp, chore/tooling fixes —
re-carved into 6 stacked PRs. Repo layout as in
[pr-stack-map.example.md](pr-stack-map.example.md).

## Invocation

```
> split payments-mega onto main
```

The agent isolates a worktree at `main`, records `END`'s SHA, finds
`pr-stack-map.md` fresh (hash matches), buckets all changed files, and
runs two scanner subagents — one bucketing files by theme (with per-hunk
notes for files serving several themes), one probing compile coupling
between the old UI and the new api surface. Then it presents the plan.

## The plan (as it appears in the reply)

```markdown
PR 1 — chore: tooling + review noise                (tier 0–1, 2 commits)
  1.1 chore: CI path filters + lint config           [technical-only]
  1.2 chore: drop dead feature flags
PR 2 — core: typed transfer errors                  (tier 2, 2 commits)
  2.1 refactor: extract error module                 [technical-only]
  2.2 feat: TransferError hierarchy + tests
PR 3 — api-client: adopt typed errors               (tier 3, 3 commits)
  3.1 feat: decode revert data into TransferError + tests
  3.2 feat: retry classifier narrows on error type + tests
  3.3 chore: delete string-matching fallback
PR 4 — webapp/api: payments quote pipeline          (tier 4, 3 commits)
  4.1 feat: quote request/response models + tests
  4.2 feat: pipeline orchestration + tests            ← forced-join risk, see note
  4.3 feat: error translation for UI consumption + tests
PR 5 — webapp/components: payment widgets           (tier 5, 2 commits)
  5.1 feat: amount input + validation states
  5.2 feat: progress + error components
PR 6 — webapp/pages: checkout flow                  (tier 6, 2 commits)
  6.1 feat: checkout page wiring
  6.2 feat: confirmation flow

Note on 4.2: the pipeline's shared core, models, and orchestrator may be
one compile unit. If 4.1 cannot go green alone, 4.1–4.2 join into one
commit (pre-authorized join-on-red; no second sign-off round-trip).
```

> **operator:** join PR5+PR6? they're both small
> **agent:** They can join (both ui, combined 4 commits — within
> limits). Flagging as an operator join, not a forced-join: PR 5 goes
> green alone, so they'd stay separate by default. Joining.

## Carve and verify

Each commit is built by restoring `END`'s content for its slice
(`git restore --source=END --staged --worktree -- <paths>`). Six files
belonged to two PRs at once; for those the agent wrote intermediate
content — `END` minus the later PR's symbols, stripped against a
denylist built up front (`grep -c` proves each strip) — and let the
later PR's commit restore the final content byte-for-byte.

Per commit: typecheck + lint of touched tiers. Per PR tip: the map's
full verify battery, judged against the `END` baseline (the 9 env-gated
api-client tests fail identically at `END`, so they don't block). Final
tip:

```
$ git diff pr5-tip END
$ (empty — byte-identical to the branch that was already validated)
```

## Handoff

```
payments-stack-1-chore     → PR #101  (base: main)
payments-stack-2-core      → PR #102  (base: payments-stack-1-chore)
payments-stack-3-apiclient → PR #103  (base: payments-stack-2-core)
payments-stack-4-pipeline  → PR #104  (base: payments-stack-3-apiclient)
payments-stack-5-checkout  → PR #105  (base: payments-stack-4-pipeline)
```

Each PR body carries its stack position and neighbour links (created
first, cross-linked after — PR numbers don't exist until creation).
Merge bottom-up; after each merge, retarget the next PR's base and
rebase the remainder.

## Where the time went

| Phase | Share |
|---|---|
| Survey, map check, coupling scan, plan | ~15–20% |
| Carve + per-commit checks | ~35–40% |
| Split-file intermediate synthesis | ~15% |
| Tip verification | ~25% |
| Push + PR creation | minutes |

The run that produced these numbers also produced most of the rules the
skill now carries: the `END`-baseline definition of green, cheap
per-commit checks, tree-hash-skipped re-verification, and snapshot-dep
refresh recipes all came out of its retrospective.
