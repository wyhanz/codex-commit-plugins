---
name: commit
description: Create exactly one git commit from current staged and unstaged changes. Use when explicitly invoked as $commit, when the user asks Codex to commit local git changes, or when a workflow requires a routine development commit.
---

# Commit
Create exactly one git commit from the current repository changes.

## When to use

Use this skill for routine local commits during development, including natural language requests such as "commit these changes", "create a commit", or "save this work in git".

Do not use this skill when the user asks to push, create a branch, create a pull request, amend an existing commit, stash changes, clean branches, or only inspect git status.

## Operating rules

- Do not edit source files.
- Do not push, create branches, or create pull requests.
- Do not use `git commit --amend`, `git reset`, `git clean`, or `git push`.
- Do not bypass hooks with `--no-verify` unless the user explicitly asks.
- Prefer explicit path staging over `git add .` or `git add -A` when there are untracked files or potentially sensitive files.
- Never stage files that appear to contain secrets or credentials.
- Do not print secret values from diffs; summarize the risk instead.

## Workflow

Follow this sequence exactly:

1. Analyze the current git state.
2. Review both staged and unstaged changes.
3. Check changed paths and diffs for likely secrets.
4. Examine recent commit messages to match the repository's style.
5. Draft an appropriate commit message.
6. Stage only relevant files for one coherent commit.
7. Create exactly one commit.
8. Show the final commit status.

## Inspect context

Run the smallest necessary command sequence:

```bash
git rev-parse --is-inside-work-tree
git branch --show-current
git status --short
git diff --stat HEAD
git diff HEAD
git log --oneline -10
```

If there are no staged or unstaged changes, stop and report that there is nothing to commit.

If the repository has existing staged changes, preserve them unless they are unsafe. If unrelated unstaged changes exist, leave them unstaged and report them in the final response.

## Secret and safety checks

Before staging, inspect changed paths and diffs for likely secrets.

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

If a likely secret is present, do not commit it. Report the filename and reason without revealing the value.

## Stage changes

- Preserve any files that are already staged unless they are unsafe.
- Stage only relevant files for one coherent commit.
- Prefer one focused development unit over mixing unrelated docs, formatting, generated artifacts, and source edits.
- Use explicit pathspecs when practical:

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

If the staged diff is empty, stop and report why no commit was created.

## Draft the commit message

Use recent commits to match the repository's style. If no clear style exists, prefer Conventional Commits.

Rules:

- Subject line should usually be 72 characters or fewer.
- Use an imperative verb.
- Mention the changed component or behavior.
- Use a body only when it adds useful context.
- Do not mention files mechanically unless that is the repository style.
- Add attribution only if the repository already uses it or the user asks for it.
- Verify the generated message is accurate against the staged diff before committing.

Examples:

```text
fix(auth): handle expired refresh tokens
feat(cli): add stale branch cleanup command
chore: update release workflow
```

## Commit

Create one commit:

```bash
git commit -m "type(scope): concise summary"
```

For a body, use additional `-m` arguments:

```bash
git commit -m "type(scope): concise summary" -m "Optional body paragraph."
```

If hooks fail, report the failure and do not bypass the hook.

## Troubleshooting

- If `git status --short` shows no changes, report that there is nothing to commit.
- If `git diff --cached --check` reports whitespace or conflict-marker errors, stop and report the exact files.
- If a hook fails, report the hook failure and leave the repository state intact.

## Final response

After the commit, run:

```bash
git log --oneline -1
git status --short
```

Respond with:

- commit hash and subject
- any files deliberately left unstaged
- any hook or warning encountered
