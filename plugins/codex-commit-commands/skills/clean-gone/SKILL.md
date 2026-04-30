---
name: clean-gone
description: Remove local git branches whose upstream tracking branch is gone, including associated worktrees. Use when explicitly invoked as $clean-gone, clean_gone, or when the user asks to clean stale local branches after remote branches were deleted.
---

# Clean Gone Branches

Delete local branches whose upstream tracking branch is marked `[gone]`, including associated worktrees.

## When to use

Use this skill after PRs are merged and remote branches have been deleted, when the user asks to clean stale branches, remove gone branches, tidy local branch lists, or run repository branch maintenance.

Do not use this skill to delete remote branches, delete old branches that are not marked `[gone]`, clean files, reset a repository, or remove the current branch.

## Operating rules

- Only delete branches whose upstream tracking state is exactly `[gone]`.
- Never delete the current branch.
- Never delete branches that merely look old, unmerged, or inactive.
- Remove an associated worktree before deleting its branch.
- Do not delete remote branches.
- Do not use `git clean`, `git reset`, `git push --delete`, or file-system `rm -rf`.

## Workflow

Follow this sequence exactly:

1. Run `git fetch --prune`.
2. List local branches and worktrees.
3. Identify local branches whose upstream tracking state is exactly `[gone]`.
4. Remove associated worktrees for gone branches before deleting those branches.
5. Delete only gone branches that are not the current branch.
6. Verify branch and worktree state after cleanup.
7. Report removed branches and worktrees, or report that no cleanup was needed.

## Inspect and prune

Run:

```bash
git rev-parse --is-inside-work-tree
git fetch --prune
git branch --show-current
git branch -vv
git worktree list
```

Identify gone branches robustly:

```bash
git for-each-ref --format='%(refname:short) %(upstream:track)' refs/heads | awk '$2 == "[gone]" {print $1}'
```

If no branches are returned, report that no cleanup is needed.

## Cleanup command

Use this command to remove associated worktrees and delete gone branches:

```bash
top="$(git rev-parse --show-toplevel)"
current="$(git branch --show-current)"

git for-each-ref --format='%(refname:short) %(upstream:track)' refs/heads |
  awk '$2 == "[gone]" {print $1}' |
  while IFS= read -r branch; do
    if [ -z "$branch" ]; then
      continue
    fi

    if [ "$branch" = "$current" ]; then
      echo "Skipping current branch: $branch"
      continue
    fi

    echo "Processing branch: $branch"

    worktree="$(git worktree list --porcelain | awk -v branch="refs/heads/$branch" '
      /^worktree / { path=$0; sub(/^worktree /, "", path) }
      /^branch / { ref=$0; sub(/^branch /, "", ref); if (ref == branch) { print path; exit } }
    ')"

    if [ -n "$worktree" ] && [ "$worktree" != "$top" ]; then
      echo "Removing worktree: $worktree"
      git worktree remove --force "$worktree"
    fi

    echo "Deleting branch: $branch"
    git branch -D -- "$branch"
  done
```

## Verify

After cleanup, run:

```bash
git branch -vv
git worktree list
```

## Troubleshooting

- If no branches are marked `[gone]`, report that no cleanup is needed and do not delete anything.
- If a gone branch is the current branch, skip it and report it.
- If a worktree removal fails, stop and report the worktree path and branch.

## Final response

Respond with:

- branches deleted
- worktrees removed
- branches skipped, if any
- confirmation when no cleanup was needed
