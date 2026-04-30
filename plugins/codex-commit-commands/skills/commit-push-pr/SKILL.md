---
name: commit-push-pr
description: Commit local changes if needed, push the current or newly created branch to origin, and create a GitHub pull request with gh. Use when explicitly invoked as $commit-push-pr or when the user asks for a complete commit, push, and PR workflow.
---

# Commit, Push, and Create Pull Request

Notice: This skill adapts Anthropic's Apache-2.0 `commit-commands` workflow for Codex plugin and skill conventions.

Create a coherent commit if needed, push the branch to `origin`, and open a GitHub pull request using the GitHub CLI.

## Operating rules

- Do not edit source files.
- Do not merge, rebase, reset, force-push, auto-merge, or delete branches.
- Do not bypass hooks with `--no-verify` unless the user explicitly asks.
- Use `gh pr create`; do not use browser automation.
- If a pull request already exists, do not create a duplicate. Report the existing PR URL.
- Add attribution only if the repository already uses it or the user asks for it.

## Required tools

Check these first:

```bash
git rev-parse --is-inside-work-tree
git remote get-url origin
command -v gh
gh auth status
```

If `gh` is missing or unauthenticated, stop and explain the requirement.

## Inspect context

```bash
git fetch origin --prune
git branch --show-current
git status --short
git diff --stat HEAD
git diff HEAD
```

Determine the base branch:

```bash
git symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null | sed 's#^origin/##'
```

If that command returns nothing, use `main` when it exists on origin, otherwise use `master` when it exists.

## Branch handling

If the current branch is `main`, `master`, `trunk`, or `develop`, create a new branch before committing.

Use a short slug derived from the change summary:

```text
codex/<type>-<summary-slug>
```

Examples:

```text
codex/fix-auth-token-refresh
codex/feat-commit-plugin
codex/chore-clean-branches
```

Create the branch:

```bash
git checkout -b codex/<summary-slug>
```

If already on a feature branch, use the current branch.

## Commit changes if needed

If there are staged or unstaged changes, follow the `$commit` skill's safety and message rules:

- inspect recent commits
- skip likely secrets
- stage one coherent change
- create exactly one commit
- do not bypass hooks

After committing, verify:

```bash
git log --oneline -1
git status --short
```

If there are no local changes and the branch has no commits ahead of the base branch, stop; there is nothing to push or open as a PR.

## Analyze the branch for PR content

Use the full branch history, not only the latest commit:

```bash
git log --oneline origin/<base>..HEAD
git diff --stat origin/<base>...HEAD
git diff origin/<base>...HEAD
```

Draft a PR title matching the repository style. Draft the body with this structure:

```markdown
## Summary
- 1-3 bullets describing the user-facing or engineering impact

## Test plan
- [ ] Command or manual check performed
- [ ] Not run, with reason
```

Only claim tests that were actually run. If no tests were run, include `- [ ] Not run (reason: <reason>)`.

## Push and create PR

Push the branch:

```bash
git push -u origin HEAD
```

Create the PR with a temporary body file:

```bash
tmpfile="$(mktemp)"
cat > "$tmpfile" <<'PR_BODY'
## Summary
- ...

## Test plan
- [ ] ...
PR_BODY

gh pr create --base "<base>" --head "$(git branch --show-current)" --title "<title>" --body-file "$tmpfile"
rm -f "$tmpfile"
```

If `gh pr create` reports that a PR already exists, run:

```bash
gh pr view --json url -q .url
```

and report the existing URL.

## Final response

Respond with:

- branch name
- commit hash and subject, if a commit was created
- PR URL
- test plan status
- any files deliberately left unstaged
