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

Use recent commits to match the repository's style, but follow the structured message contract below unless the user explicitly asks for a different format.

### Commit message contract

Default shape:

```text
<type>(optional-scope): <primary change>

<details body>

Co-Authored-By: Codex <current-model> <noreply@openai.com>
```

Subject rules:

- Use Conventional Commit type by default: `feat`, `fix`, `docs`, `chore`, `refactor`, `test`, `build`, `ci`, `perf`, or `style`.
- Include a scope when it clarifies the affected component: `docs(api): ...`, `chore(gitignore): ...`, `feat(sdk): ...`.
- Keep the subject concise, usually 72 characters or fewer.
- Use imperative, present-tense wording.
- Summarize the primary user-facing or engineering change, not the file list.

Body rules:

- Always add a body when the staged diff changes more than one file, removes files, changes generated/binary artifacts, or the subject cannot explain the impact by itself.
- Use one of these two body styles:
  - Detail bullets for one primary change with important implementation details.
  - Additional subject-style summary lines for several related change categories in one commit.
- Do not mix detail bullets and subject-style summary lines in the same body unless the diff genuinely needs both.
- Mention paths only when they clarify location or impact, usually in parentheses.
- Use body lines to explain what changed and why it matters; do not restate the subject mechanically.
- For subject-style body lines, the first additional summary line may directly follow the primary subject on the next line, matching the repository's multi-summary sample. Use a blank line before a new logical group and before the co-author trailer.

Detail bullet style:

```text
chore: remove old Python package and clean up .a files

- Delete fisheye_stitcher/ Python package (migrated to C++ implementation)
- Remove .a files from git tracking (sdk_example/lib/)

Co-Authored-By: Codex <current-model> <noreply@openai.com>
```

Subject-style body style:

```text
chore(gitignore): ignore test_files directory
docs: add fish-eye panorama stitching documentation

feat(sdk): add sdk example code

Co-Authored-By: Codex <current-model> <noreply@openai.com>
```

Footer rules:

- Include a co-author trailer by default:
  `Co-Authored-By: Codex <current-model> <noreply@openai.com>`
- Use the current Codex model slug when it is available, for example `gpt-5.5` from the current session or from Codex config.
- Codex documents `model` as the config key and supports model switching with CLI/UI controls, but does not document a stable shell environment variable for the active model. If the active session model is visible in context, use it before falling back to config.
- If Codex config defines `commit_attribution`, use that string exactly. If it is set to an empty string, omit the co-author trailer.
- If no model can be resolved, fall back to Codex's canonical OpenAI attribution: `Co-Authored-By: Codex <noreply@openai.com>`.
- Place the trailer after a blank line at the end of the message.
- Omit the trailer only when the user explicitly asks for no attribution.
- Verify the generated message is accurate against the staged diff before committing.

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
  coauthor_trailer="Co-Authored-By: Codex ${current_model} <noreply@openai.com>"
else
  coauthor_trailer="Co-Authored-By: Codex <noreply@openai.com>"
fi
```

Examples:

```text
docs: expand commit command workflow constraints

- Add runtime message contract for structured commit bodies
- Document co-author trailer handling

Co-Authored-By: Codex <current-model> <noreply@openai.com>
```

## Commit

Create one commit. Prefer a temporary commit message file with `git commit -F` so multi-line bodies and trailers preserve the exact required layout:

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

For subject-style body lines, preserve the exact line breaks in the message file:

```bash
tmpfile="$(mktemp)"
{
  printf '%s\n' "chore(gitignore): ignore test_files directory"
  printf '%s\n' "docs: add fish-eye panorama stitching documentation"
  printf '\n'
  printf '%s\n' "feat(sdk): add sdk example code"
  if [ -n "$coauthor_trailer" ]; then
    printf '\n'
    printf '%s\n' "$coauthor_trailer"
  fi
} > "$tmpfile"
git commit -F "$tmpfile"
rm -f "$tmpfile"
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
