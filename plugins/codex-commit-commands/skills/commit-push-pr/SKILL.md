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

Use recent commits to match the repository's style, but follow this structured commit message contract unless the user explicitly asks for a different format:

```text
<type>(optional-scope): <primary change>

<details body>

Co-Authored-By: Codex <current-model> <codex@openai.com>
```

Subject rules:

- Use Conventional Commit type by default: `feat`, `fix`, `docs`, `chore`, `refactor`, `test`, `build`, `ci`, `perf`, or `style`.
- Include a scope when it clarifies the affected component.
- Keep the subject concise, usually 72 characters or fewer.
- Use imperative, present-tense wording.
- Summarize the primary user-facing or engineering change, not the file list.

Body rules:

- Always add a body when the staged diff changes more than one file, removes files, changes generated/binary artifacts, or the subject cannot explain the impact by itself.
- Use detail bullets for one primary change with important implementation details.
- Use additional subject-style summary lines for several related change categories in one commit.
- Do not mix detail bullets and subject-style summary lines in the same body unless the diff genuinely needs both.
- Mention paths only when they clarify location or impact, usually in parentheses.
- For subject-style body lines, the first additional summary line may directly follow the primary subject on the next line, matching the repository's multi-summary sample. Use a blank line before a new logical group and before the co-author trailer.

Footer rules:

- Include a co-author trailer by default:
  `Co-Authored-By: Codex <current-model> <codex@openai.com>`
- Use the current Codex model slug when it is available, for example `gpt-5.5` from the current session or from Codex config.
- Codex documents `model` as the config key and supports model switching with CLI/UI controls, but does not document a stable shell environment variable for the active model. If the active session model is visible in context, use it before falling back to config.
- If Codex config defines `commit_attribution`, use that string exactly. If it is set to an empty string, omit the co-author trailer.
- If no model can be resolved, fall back to Codex's canonical OpenAI attribution: `Co-Authored-By: Codex <codex@openai.com>`.
- Place the trailer after a blank line at the end of the message.
- Omit the trailer only when the user explicitly asks for no attribution.

Resolve the trailer before writing the commit message:

```bash
commit_attribution=""
commit_attribution_set="false"
current_model=""
repo_root="$(git rev-parse --show-toplevel 2>/dev/null || pwd)"

for config in "$repo_root/.codex/config.toml" "${CODEX_HOME:-$HOME/.codex}/config.toml"; do
  [ -f "$config" ] || continue
  if [ "$commit_attribution_set" = "false" ]; then
    attribution_line="$(awk -F= '/^[[:space:]]*commit_attribution[[:space:]]*=/ { value=$2; sub(/^[[:space:]]*/, "", value); sub(/[[:space:]]*$/, "", value); gsub(/^"|"$/, "", value); print "set:" value; exit }' "$config")"
    if [ -n "$attribution_line" ]; then
      commit_attribution_set="true"
      commit_attribution="${attribution_line#set:}"
    fi
  fi
  if [ -z "$current_model" ]; then
    current_model="$(awk -F= '/^[[:space:]]*model[[:space:]]*=/ { value=$2; sub(/^[[:space:]]*/, "", value); sub(/[[:space:]]*$/, "", value); gsub(/^"|"$/, "", value); print value; exit }' "$config")"
  fi
done

if [ "$commit_attribution_set" = "true" ] && [ -n "$commit_attribution" ]; then
  coauthor_trailer="$commit_attribution"
elif [ "$commit_attribution_set" = "true" ]; then
  coauthor_trailer=""
elif [ -n "$current_model" ]; then
  coauthor_trailer="Co-Authored-By: Codex ${current_model} <codex@openai.com>"
else
  coauthor_trailer="Co-Authored-By: Codex <codex@openai.com>"
fi
```

Examples:

```text
chore: remove old Python package and clean up .a files

- Delete fisheye_stitcher/ Python package (migrated to C++ implementation)
- Remove .a files from git tracking (sdk_example/lib/)

Co-Authored-By: Codex <current-model> <codex@openai.com>
```

```text
chore(gitignore): ignore test_files directory
docs: add fish-eye panorama stitching documentation

feat(sdk): add sdk example code

Co-Authored-By: Codex <current-model> <codex@openai.com>
```

Create exactly one commit. Prefer a temporary commit message file with `git commit -F` so multi-line bodies and trailers preserve the exact required layout:

```bash
tmpfile="$(mktemp)"
{
  printf '%s\n' "type(scope): concise summary"
  printf '\n'
  printf '%s\n' "- Detail one"
  printf '%s\n' "- Detail two"
  if [ -n "$coauthor_trailer" ]; then
    printf '\n'
    printf '%s\n' "$coauthor_trailer"
  fi
} > "$tmpfile"
git commit -F "$tmpfile"
rm -f "$tmpfile"
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
