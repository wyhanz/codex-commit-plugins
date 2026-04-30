# Codex Commit Plugins

A Codex plugin marketplace that brings Claude Code-style commit commands to Codex.

## Intent

This repository implements the Claude Code `commit-commands` workflow for Codex. The goal is to provide Codex-native skills for structured commits, commit-and-PR handoff, and stale branch cleanup while preserving the safety checks and workflow ergonomics developers expect from Claude Code's commit plugin.

This repository currently publishes one plugin:

- **Codex Commit Commands** (`codex-commit-commands`) provides `$commit`, `$commit-push-pr`, and `$clean-gone`.

## Install From GitHub

Codex installs plugins through a marketplace root. This repository root contains `.agents/plugins/marketplace.json`, so add the repository itself as the marketplace:

```bash
codex plugin marketplace add wyuhan/codex-commit-plugins
```

To pin a branch, tag, or commit:

```bash
codex plugin marketplace add wyuhan/codex-commit-plugins@main
```

You can also install with a Git URL:

```bash
codex plugin marketplace add https://github.com/wyuhan/codex-commit-plugins.git
```

For private forks or SSH-only access:

```bash
codex plugin marketplace add git@github.com:wyuhan/codex-commit-plugins.git
```

After adding the marketplace, restart Codex, open `/plugins`, and install **Codex Commit Commands**.

## Install From A Local Checkout

```bash
git clone https://github.com/wyuhan/codex-commit-plugins.git
cd codex-commit-plugins
codex plugin marketplace add .
```

Restart Codex, open `/plugins`, and install **Codex Commit Commands**.

## Update Or Remove

Upgrade this marketplace:

```bash
codex plugin marketplace upgrade codex-commit-plugins
```

Remove it from Codex:

```bash
codex plugin marketplace remove codex-commit-plugins
```

## Available Plugin

### Codex Commit Commands

Use explicit skill invocation:

```text
$commit
$commit-push-pr
$clean-gone
```

- `$commit` creates exactly one commit from the current staged and unstaged changes.
- `$commit-push-pr` commits local changes if needed, pushes the branch, and creates a GitHub pull request with `gh`.
- `$clean-gone` removes local branches whose upstream tracking branch is gone, including associated worktrees.

See `plugins/codex-commit-commands/README.md` for detailed usage, requirements, and safety notes.

## Repository Layout

```text
.agents/plugins/marketplace.json
plugins/codex-commit-commands/
  .codex-plugin/plugin.json
  LICENSE
  README.md
  THIRD_PARTY_NOTICES.md
  skills/
    commit/SKILL.md
    commit-push-pr/SKILL.md
    clean-gone/SKILL.md
```

## License And Attribution

This repository is licensed under Apache-2.0. The `codex-commit-commands` plugin adapts workflow ideas from Anthropic's Apache-2.0 `commit-commands` plugin in `anthropics/claude-plugins-official`.

See `LICENSE` and `THIRD_PARTY_NOTICES.md` for license terms, upstream attribution, and modification notes.
