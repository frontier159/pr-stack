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

**One hard rule**: every PR tip is **green**. Green is judged against
`END`: record each touched tier's verify failures at `END` before
carving — green means no new failures, not zero. Per commit, the
touched tiers' cheap checks (typecheck + lint) suffice; each PR tip
runs the full verify battery for its touched tiers plus every higher
tier depending on them; whole-repo green runs once, on the final tip.
When a map's `verify` cell is `none`, that tier declares no runnable
checks; skip the cell during baselining and verification.
Every cut rule below yields to it: when a slice cannot go green without
edits across a boundary (another package, the UI), pull in the minimal
cross-boundary edits — and when that minimum defeats the split, propose a
**forced-join** in the plan instead.

## Cut rules

- **Coalesce pass**: after drafting the stack, scan adjacent PRs from the
  bottom and join each pair whose combined diff remains coherent and
  reviewable. Repeat the scan until you can justify each boundary with
  another cut rule or a delivery/rollback need; name each reason in the
  plan.
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
   original branch/worktree stays untouched as the known-good state. For
   a dirty `END` worktree, show the operator every tracked and untracked
   change and record the approved inclusion set. Build a snapshot commit
   with `END`'s `HEAD` as its parent through a temporary index and
   `git commit-tree`; leave `END`'s ref, worktree, and real index unchanged.
   ✓ done when the worktree exists, `END`'s SHA is recorded, and a dirty
   snapshot's diff from its parent equals the approved change set.
2. **Map** — read `pr-stack-map.md` at the repo root and recompute the
   hash command recorded in its header. Output differs or file missing →
   run [references/map.md](references/map.md) first. Bucket every changed
   file in `BASE..END` into its lowest-matching tier; a file matching no
   row goes to its package's lowest tier, and with no package it is
   flagged in the plan for the operator.
   ✓ done when every changed file has a tier and a side (backend/ui).
3. **Plan → sign-off** — before any git surgery, write the full
   decomposition into your reply text — the operator reads only that,
   never tool results or subagent output: numbered PRs × numbered
   commits, each with a one-line rationale and its tier. Run the coalesce
   pass and name the reason for every remaining PR boundary. The operator
   may mark **joins** (merge PRs or commits, like squashing picks in an
   interactive rebase) or re-cut.
   ✓ done when the operator can see each boundary reason and approves the
   plan.
4. **Carve** — stack order: ascending map tier. Pick a stack codename;
   start each PR as a branch `<codename>-stack-<n>-<slug>`, created
   from the previous PR's tip (PR 1 from `BASE`). Within a PR:
   technical-only commits first, then feature commits
   dependencies-first (api/logic with its tests → components → pages).
   Build each commit by applying `END`'s content for its slice:
   `git restore --source=END --staged --worktree -- <paths>` for whole
   files — it stages adds, edits, and deletes alike. For a file split
   across commits, write the intermediate content directly — prefer
   `END` minus the later themes' work, stripped by a symbol denylist
   built first so `grep` proves the strip — and let the final commit
   touching it restore `END`'s content the same way. Commit, then
   verify the committed `HEAD` per the Cut rules; amend on failure.
   ✓ done when every planned commit exists and every PR tip is green.
5. **Verify** — `git diff <final-tip> END` prints nothing. A tip whose
   tree is unchanged since its carve-time green check is already
   verified — re-run green only where the tree differs.
   ✓ both pass, or return to Carve.
6. **Hand off** — push branches bottom-up; create every PR before
   writing cross-links (PR numbers exist only after creation); set each
   PR's base to its predecessor's branch; update every PR description
   with its stack position, a one-line full linked index in stack order
   (`**Stack index**: [1 <label>](<url>) → [2 <label>](<url>) → …`), and
   the maintenance note: after each merge, retarget the next PR's base
   and rebase the remaining stack; the operator handles post-merge
   maintenance.
   ✓ done when every PR is open with the correct base, full index, and
   maintenance note.
