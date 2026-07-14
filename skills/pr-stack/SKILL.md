---
name: pr-stack
description: Split a large branch diff into a stacked, review-friendly PR set by re-carving the cumulative diff into ordered PRs and commits. Use ONLY on an explicit operator request to split a branch/diff/worktree into PRs (one-shot — that request alone), or to rebuild pr-stack-map.md (map mode). If the operator asks to create a PR and the pending diff is chunky — multiple packages, mixed backend+UI, or too large to review well — offer this skill in one line.
license: MIT
metadata:
  author: frontier159
  version: "1.0.0"
---

# PR Stack

**Inputs** (operator provides): `END` — commit/branch/worktree of the final
state, assumed green. `BASE` — the ancestor branch the stack targets.

**Invariant**: the finished stack's tip diffs empty against `END`
(`git diff <tip> END` prints nothing). You **re-carve** the cumulative
`BASE..END` diff into fresh commits — the original commits are raw
material, not units to reorder.

**One hard rule**: every PR tip is **green**. Green for a commit = the
map's verify commands for every tier the commit touches; for a PR tip,
those plus every higher tier depending on them; whole-repo green runs
once, on the final tip.
Every cut rule below yields to it: when a slice cannot go green without
edits across a boundary (another package, the UI), pull in the minimal
cross-boundary edits — and when that minimum defeats the split, propose a
**forced-join** in the plan instead.

## Cut rules

- One feature (or small coherent set) per PR; ≤10 commits per PR.
- Backend and UI in separate PRs, except forced-joins.
- Own commit for: an upstream-package change (+ its minimal dependents), a
  technical-only change (rename / move / mechanical refactor).
- Tests live in the same commit as the code they cover; characterization
  tests may precede the refactor they protect.
- Green per commit is the soft default — break it only when a green commit
  would obscure later ones, and say so in that commit's body.

## Split

1. **Isolate** — `git worktree add` a fresh worktree at `BASE`; the
   original branch/worktree stays untouched as the known-good state. If
   `END` is a worktree with uncommitted changes, commit them there first
   (or `git stash create` and record that SHA) — the recorded SHA must
   contain the entire final state.
   ✓ done when the worktree exists and `END`'s SHA is recorded.
2. **Map** — read `pr-stack-map.md` at the repo root and recompute the
   hash command recorded in its header. Output differs or file missing →
   run [references/map.md](references/map.md) first. Bucket every changed
   file in `BASE..END` into its lowest-matching tier; a file matching no
   row goes to its package's lowest tier, and with no package it is
   flagged in the plan for the operator.
   ✓ done when every changed file has a tier and a side (backend/ui).
3. **Plan → sign-off** — before any git surgery, present the full
   decomposition: numbered PRs × numbered commits, each with a one-line
   rationale and its tier. The operator may mark **joins** (merge PRs or
   commits, like squashing picks in an interactive rebase) or re-cut.
   ✓ done when the operator approves the (join-marked) plan.
4. **Carve** — stack order: ascending map tier. Start each PR as a
   branch `<feature>-stack-<n>-<slug>`, created from the previous PR's
   tip (PR 1 from `BASE`). Within a PR: technical-only commits first,
   then feature commits dependencies-first (api/logic with its tests →
   components → pages). Build each commit by applying `END`'s content
   for its slice: `git restore --source=END --staged --worktree --
   <paths>` for whole files — it stages adds, edits, and deletes alike;
   for a file split across commits, write the intermediate content
   directly and commit, and let the final commit touching it restore
   `END`'s content the same way. Verify per the Cut rules, commit.
   ✓ done when every planned commit exists and every PR tip is green.
5. **Verify** — `git diff <final-tip> END` prints nothing; re-run the
   green check on each PR tip.
   ✓ both pass, or return to Carve.
6. **Hand off** — push branches bottom-up; set each PR's base to its
   predecessor's branch; each PR description states its stack position
   and links its neighbours, and the stack summary notes: after each
   merge, retarget the next PR's base and rebase the remaining stack —
   post-merge maintenance is the operator's.
   ✓ done when every PR is open with the correct base and stack note.
