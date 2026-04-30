# Codex Commit Commands

A Codex plugin for common Git workflows:

- `$commit` creates exactly one commit from the current staged and unstaged changes.
- `$commit-push-pr` commits local changes if needed, pushes the branch, and creates a GitHub pull request with `gh`.
- `$clean-gone` removes local branches whose upstream tracking branch is gone, including associated worktrees.

The skills are explicit-invocation only, so Codex will use them when you ask for `$commit`, `$commit-push-pr`, or `$clean-gone`.

## Install From GitHub

This plugin is published through the marketplace root in the repository, not from the plugin directory directly. The repository root must contain `.agents/plugins/marketplace.json`.

Install the marketplace from GitHub:

```bash
codex plugin marketplace add wyuhan/codex-commit-plugins
```

To pin a branch, tag, or commit:

```bash
codex plugin marketplace add wyuhan/codex-commit-plugins@main
```

You can also use a Git URL:

```bash
codex plugin marketplace add https://github.com/wyuhan/codex-commit-plugins.git
```

For a private repository, use an SSH URL that your local Git can access:

```bash
codex plugin marketplace add git@github.com:wyuhan/codex-commit-plugins.git
```

After adding the marketplace, restart Codex, open `/plugins`, and install **Codex Commit Commands**.

## Install From A Local Checkout

Clone the repository, then add the checkout as a local marketplace:

```bash
git clone https://github.com/wyuhan/codex-commit-plugins.git
cd codex-commit-plugins
codex plugin marketplace add .
```

Restart Codex, open `/plugins`, and install **Codex Commit Commands**.

## Update Or Remove

Upgrade the configured marketplace:

```bash
codex plugin marketplace upgrade codex-commit-plugins
```

Remove it from Codex:

```bash
codex plugin marketplace remove codex-commit-plugins
```

## Usage

Invoke the skills explicitly in Codex:

```text
$commit
$commit-push-pr
$clean-gone
```

`$commit` inspects the current repository, checks for likely secrets, stages one coherent change, and creates one commit.

`$commit-push-pr` follows the commit workflow, pushes the current branch to `origin`, and creates a pull request with GitHub CLI.

`$clean-gone` fetches with prune, finds local branches whose upstream state is `[gone]`, removes associated worktrees, and deletes those local branches.

## Requirements

- Git must be installed and configured.
- `$commit-push-pr` requires GitHub CLI (`gh`) installed and authenticated.
- The repository should have a remote named `origin` for push and PR workflows.

## Safety Notes

- The commit workflows do not push unless `$commit-push-pr` is invoked.
- The cleanup workflow only targets local branches whose upstream tracking state is exactly `[gone]`.
- The skills include command policies for Codex, but Codex's shell sandbox and approval settings remain the enforcement layer.

## License And Attribution

This plugin is licensed under Apache-2.0. It adapts workflow ideas from Anthropic's Apache-2.0 `commit-commands` plugin in `anthropics/claude-plugins-official`.

See `LICENSE` and `THIRD_PARTY_NOTICES.md` in this plugin directory for license terms, upstream attribution, and modification notes.
