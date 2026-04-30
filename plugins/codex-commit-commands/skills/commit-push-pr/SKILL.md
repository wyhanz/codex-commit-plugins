---
name: commit-push-pr
description: Commit local changes if needed, push the current or newly created branch to origin, and create a GitHub pull request with gh. Use when explicitly invoked as $commit-push-pr or when the user asks for a complete commit, push, and PR workflow.
---

# Commit, Push, and Create Pull Request

Create a coherent commit if needed, push the branch to `origin`, and open a GitHub pull request using the GitHub CLI.

## When to use

Use this skill when the user is ready to publish work and asks to commit and push, open a PR, create a pull request, or run a complete commit-push-PR workflow.

Do not use this skill for a local-only commit, branch cleanup, amend, rebase, merge, force-push, or PR review task.

## Operating rules

- Do not edit source files.
- Do not merge, rebase, reset, force-push, auto-merge, or delete branches.
- Do not bypass hooks with `--no-verify` unless the user explicitly asks.
- Use `gh pr create`; do not use browser automation.
- If a pull request already exists, do not create a duplicate. Report the existing PR URL.
- Add attribution only if the repository already uses it or the user asks for it.

## Workflow

Follow this sequence exactly:

1. Check that GitHub CLI is installed and authenticated.
2. Fetch and inspect repository state.
3. Determine the base branch.
4. Create a new feature branch when currently on `main`, `master`, `trunk`, or `develop`.
5. If local changes exist, stage and create exactly one coherent commit using the commit safety rules below.
6. Analyze all branch commits and the full branch diff for the PR title and body.
7. Push the branch to `origin`.
8. Create one pull request with `gh pr create`, or report the existing PR URL.

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

If there are staged or unstaged changes, follow these commit safety and message rules directly. Do not rely on another skill being loaded.

Inspect recent commits:

```bash
git log --oneline -10
```

Before staging, inspect changed paths and diffs for likely secrets. Never stage files that appear to contain secrets or credentials, and do not print secret values from diffs.

Skip or warn on paths such as:

- `.env`, `.env.*`
- `*.pem`, `*.key`, `*.p12`, `*.pfx`
- `id_rsa*`, `id_ed25519*`
- `credentials.json`, `credentials*.json`
- `secrets.yml`, `secrets.yaml`, `secret*.json`
- `.npmrc`, `.pypirc`, files containing auth tokens

Treat these diff markers as high risk:

- `BEGIN PRIVATE KEY`
- `AWS_SECRET_ACCESS_KEY`
- `GITHUB_TOKEN`, `ghp_`, `github_pat_`
- `OPENAI_API_KEY`, `sk-`
- `SLACK_BOT_TOKEN`, `xoxb-`
- `password=`, `api_key=`, `secret=`, `token=`

Stage only relevant files for one coherent commit, preserving already staged safe files. Prefer explicit pathspecs over `git add .` or `git add -A` when there are untracked files or potentially sensitive files:

```bash
git add -- path/to/file another/path
```

After staging, inspect the staged commit:

```bash
git status --short
git diff --cached --stat
git diff --cached
git diff --cached --check
```

Use recent commits to match the repository's style. If no clear style exists, prefer Conventional Commits. Use an imperative subject line, usually 72 characters or fewer, and add attribution only if the repository already uses it or the user asks for it.

Create exactly one commit:

```bash
git commit -m "type(scope): concise summary"
```

Do not bypass hooks with `--no-verify` unless the user explicitly asks. If hooks fail, report the failure and stop.

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

If the repository has a pull request template, follow it while preserving the summary and test-plan information above.

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

## Troubleshooting

- If `gh` is missing or unauthenticated, stop before committing or pushing and explain the requirement.
- If no base branch can be determined, stop and report the missing `origin/HEAD`, `origin/main`, or `origin/master`.
- If `gh pr create` fails, report the command failure and leave the branch pushed.

## Final response

Respond with:

- branch name
- commit hash and subject, if a commit was created
- PR URL
- test plan status
- any files deliberately left unstaged
