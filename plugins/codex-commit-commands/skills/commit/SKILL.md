---
name: commit
description: Create exactly one git commit from current staged and unstaged changes. Use when explicitly invoked as $commit or when the user asks Codex to commit local git changes.
---

# Commit

Notice: This skill adapts Anthropic's Apache-2.0 `commit-commands` workflow for Codex plugin and skill conventions.

Create exactly one git commit from the current repository changes.

## Operating rules

- Do not edit source files.
- Do not push, create branches, or create pull requests.
- Do not use `git commit --amend`, `git reset`, `git clean`, or `git push`.
- Do not bypass hooks with `--no-verify` unless the user explicitly asks.
- Prefer explicit path staging over `git add .` or `git add -A` when there are untracked files or potentially sensitive files.
- Never stage files that appear to contain secrets or credentials.
- Do not print secret values from diffs; summarize the risk instead.

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
