# Codex Commit Commands Plugin

Claude Code-style commit commands implemented as Codex skills.

## Overview

The Codex Commit Commands Plugin brings Claude Code's `commit-commands` workflow to Codex, with runtime rules captured in Codex-native `SKILL.md` files.

## Commands

### `$commit`

Creates a git commit with an automatically generated commit message based on staged and unstaged changes.

**What it does:**
1. Analyzes current git status
2. Reviews both staged and unstaged changes
3. Checks changed paths and diffs for likely secrets
4. Examines recent commit messages to match your repository's style
5. Drafts an appropriate commit message
6. Stages relevant files
7. Creates exactly one commit

**Usage:**
```text
$commit
```

**Example workflow:**
```text
# Make some changes to your code
# Then invoke:
$commit

# Codex will:
# - Review your changes
# - Stage the relevant files
# - Create one commit with an appropriate message
# - Show you the commit status
```

**Features:**
- Automatically drafts commit messages that match your repo's style
- Falls back to conventional commit style when no clear repo style exists
- Avoids committing files that look like secrets or credentials
- Adds Codex co-author attribution from `commit_attribution` or the configured model, unless attribution is disabled
- Does not push, create branches, or open pull requests

### `$commit-push-pr`

Complete workflow skill that commits local changes if needed, pushes the branch, and creates a pull request.

**What it does:**
1. Checks that `gh` is installed and authenticated
2. Creates a new feature branch if currently on `main`, `master`, `trunk`, or `develop`
3. Stages and commits changes with an appropriate message when needed
4. Pushes the branch to `origin`
5. Creates a pull request using `gh pr create`
6. Provides the PR URL

**Usage:**
```text
$commit-push-pr
```

**Example workflow:**
```text
# Make your changes
# Then invoke:
$commit-push-pr

# Codex will:
# - Create a feature branch if needed
# - Commit your changes if needed
# - Push to origin
# - Open a PR with summary and test plan
# - Give you the PR URL to review
```

**Features:**
- Analyzes all commits in the branch for the PR description
- Creates PR descriptions with:
  - Summary of changes
  - Test plan checklist
- Handles branch creation automatically when invoked from a protected base branch
- Uses GitHub CLI (`gh`) for PR creation
- Does not force-push, merge, rebase, auto-merge, or delete branches

**Requirements:**
- GitHub CLI (`gh`) must be installed and authenticated
- Repository must have a remote named `origin`

### `$clean-gone`

Cleans up local branches that have been deleted from the remote repository.

**What it does:**
1. Runs `git fetch --prune`
2. Lists local branches to identify `[gone]` upstream status
3. Identifies worktrees associated with `[gone]` branches
4. Removes associated worktrees before deleting those branches
5. Deletes local branches marked as `[gone]`
6. Provides feedback on removed branches

**Usage:**
```text
$clean-gone
```

**Example workflow:**
```text
# After PRs are merged and remote branches are deleted
$clean-gone

# Codex will:
# - Find all local branches marked as [gone]
# - Remove any associated worktrees
# - Delete the stale local branches
# - Report what was cleaned up
```

**Features:**
- Handles both regular branches and worktree branches
- Safely removes worktrees before deleting branches
- Shows clear feedback about what was removed
- Reports if no cleanup was needed
- Only removes branches whose upstream state is exactly `[gone]`

**When to use:**
- After merging and deleting remote branches
- When your local branch list is cluttered with stale branches
- During regular repository maintenance

## Installation

Install this plugin through the Codex plugin marketplace. This repository provides the marketplace file at `.agents/plugins/marketplace.json`.

```bash
codex plugin marketplace add wyuhan/codex-commit-plugins
```

After adding the marketplace, restart Codex, open `/plugins`, and install **Codex Commit Commands**.

For local development:

```bash
git clone https://github.com/wyuhan/codex-commit-plugins.git
cd codex-commit-plugins
codex plugin marketplace add .
```

## Best Practices

### Using `$commit`
- Review the staged changes before committing
- Let Codex analyze your changes and match your repo's commit style
- Verify the generated message is accurate
- Use for routine commits during development

### Using `$commit-push-pr`
- Use when you're ready to create a PR
- Ensure your changes are complete and tested
- Codex will analyze the full branch history for the PR description
- Review the PR description and edit if needed
- Use when you want to minimize context switching

### Using `$clean-gone`
- Run periodically to keep your branch list clean
- Especially useful after merging multiple PRs
- Safe to run because it only removes branches already deleted remotely
- Helps maintain a tidy local repository

## Workflow Integration

### Quick commit workflow:
```text
# Write code
$commit
# Continue development
```

### Feature branch workflow:
```text
# Develop feature across multiple commits
$commit  # First commit
# More changes
$commit  # Second commit
# Ready to create PR
$commit-push-pr
```

### Maintenance workflow:
```text
# After several PRs are merged
$clean-gone
# Clean workspace ready for next feature
```

## Requirements

- Git must be installed and configured
- For `$commit-push-pr`: GitHub CLI (`gh`) must be installed and authenticated
- Repository must be a git repository with a remote

## Troubleshooting

### `$commit` reports no changes

**Issue**: No changes to commit

**Solution**:
- Ensure you have unstaged or staged changes
- Run `git status --short` to verify changes exist

### `$commit-push-pr` fails to create PR

**Issue**: `gh pr create` command fails

**Solution**:
- Install GitHub CLI: `brew install gh` on macOS or see the GitHub CLI installation guide
- Authenticate: `gh auth login`
- Ensure repository has a GitHub remote named `origin`

### `$clean-gone` doesn't find branches

**Issue**: No branches marked as `[gone]`

**Solution**:
- Run `git fetch --prune` to update remote tracking
- Branches must be deleted from the remote to show as `[gone]`

## Tips

- **Combine with other tools**: Use `$commit` during development, then `$commit-push-pr` when ready
- **Let Codex draft messages**: The commit message analysis follows your repo's style
- **Regular cleanup**: Run `$clean-gone` weekly to maintain a clean branch list
- **Review before pushing**: Always review the commit message and changes before pushing

## Author

wyuhan

## Version

0.1.4
