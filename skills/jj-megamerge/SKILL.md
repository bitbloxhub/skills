---
name: jj-megamerge
description: "Use this skill for Jujutsu megamerge workflows: maintaining one octopus merge over multiple independent work branches, staging new stacks, absorbing WIP into existing commits, restacking onto trunk, and publishing only individual branches."
---

# jj-megamerge

Jujutsu megamerge workflow based on [Jujutsu megamerges for fun and profit](https://isaaccorbrey.com/notes/jujutsu-megamerges-for-fun-and-profit).

## Core model

- Keep one named, empty octopus merge (`megamerge`) with every active work branch as a direct parent.
- Every work branch starts directly from `trunk()`/`main`, unless it intentionally contains a stack of dependent commits.
- Work on an empty child of `megamerge`; this is WIP.
- Megamerge is local integration state. Never push the megamerge itself.
- Push and open PRs for individual parent branches only.
- Do not create replacement megamerges casually. Update the existing megamerge or create one replacement, then abandon obsolete merge revisions immediately.

Example:

```text
main
├─ branch A
├─ branch B
├─ branch C
└─ megamerge
   ├─ branch A
   ├─ branch B
   └─ branch C
      └─ empty WIP (@)
```

A parent may itself be a stack:

```text
main → branch A → branch A follow-up
```

## Create and inspect

Create a megamerge from direct branch tips:

```bash
jj new branch-a branch-b branch-c
jj commit -m megamerge
```

Start WIP above it:

```bash
jj new
```

Inspect only current ancestry when debugging graph shape:

```bash
jj log -r ::@
jj log -r 'parents(@-)'
```

The latter must list each intended megamerge parent. Avoid relying on default `jj log` while cleaning history; it can show unrelated mutable revisions and obsolete divergent rewrites.

## Add work

Aliases are optional convenience. Core megamerge workflow requires no JJ config changes.

Add a complete stack below the closest merge:

```toml
[revset-aliases]
"closest_merge(to)" = "heads(::to & merges())"

[aliases]
stack = ["rebase", "--after", "trunk()", "--before", "closest_merge(@)", "--revision"]
stage = ["stack", "closest_merge(@)+:: ~ empty()"]
```

Then:

```bash
jj stage
```

Stage a specific revset:

```bash
jj stack x::y
```

Stage a specific revset:

```bash
jj stack x::y
```

## Submit WIP

Changes belonging to existing parent commits:

```bash
jj absorb
jj squash --from <wip> --to <target> --interactive
```

Changes belonging in a new commit:

```bash
jj commit
jj rebase --revision <new> --after <parent> --before <megamerge>
```

Move the branch/bookmark after creating a new commit:

```bash
jj bookmark move --from y --to x
```

Keep each PR branch directly based on `trunk()` when it is intended to be independent.

## Restack

Rebase controlled mutable work onto trunk without rewriting commits owned by others.
The alias is optional; run the equivalent command directly:

```bash
jj rebase --onto 'trunk()' --source 'roots(trunk()..) & mutable()' --simplify-parents
```

Optional alias:

```toml
[aliases]
restack = [
  "rebase",
  "--onto", "trunk()",
  "--source", "roots(trunk()..) & mutable()",
  "--simplify-parents",
]
```

Run:

```bash
jj restack
```

`--simplify-parents` removes redundant merge edges after restacking.

## Safety rules

- Preserve one megamerge; do not make a new merge for every amendment.
- Before rewriting a parent, inspect descendants and update the existing megamerge deliberately.
- After rewriting, remove obsolete mutable merge revisions and verify `mutable() & ~::@` is empty or intentional.
- Resolve graph problems with revsets and parent inspection, not visual guesses from branch lines.
- Keep megamerge private. Publish individual branches with explicit bookmarks, for example:

```bash
jj git push --remote origin -c <parent-revision>
```

- Open PRs from those individual branch bookmarks. Do not push a merge containing unrelated work.
