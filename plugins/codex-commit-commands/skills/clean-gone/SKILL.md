---
name: clean-gone
description: Remove local git branches whose upstream tracking branch is gone, including associated worktrees. Use when explicitly invoked as $clean-gone, clean_gone, or when the user asks to clean stale local branches after remote branches were deleted.
---

# Clean Gone Branches

Notice: This skill adapts Anthropic's Apache-2.0 `commit-commands` workflow for Codex plugin and skill conventions.

Delete local branches whose upstream tracking branch is marked `[gone]`, including associated worktrees.

## Operating rules

- Only delete branches whose upstream tracking state is exactly `[gone]`.
- Never delete the current branch.
- Never delete branches that merely look old, unmerged, or inactive.
- Remove an associated worktree before deleting its branch.
- Do not delete remote branches.
- Do not use `git clean`, `git reset`, `git push --delete`, or file-system `rm -rf`.

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

## Final response

Respond with:

- branches deleted
- worktrees removed
- branches skipped, if any
- confirmation when no cleanup was needed
