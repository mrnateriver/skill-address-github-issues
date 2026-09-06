# Address GitHub Issues

A Codex plugin that packages the Address GitHub Issues skill. It inventories and
triages issues, orders dependencies, and resolves each issue or approved group
through an isolated worktree and a rebase-merged pull request.

## Install from GitHub

Once the plugin files are published to
[mrnateriver/skill-address-github-issues](https://github.com/mrnateriver/skill-address-github-issues),
install with:

```bash
codex plugin marketplace add https://github.com/mrnateriver/skill-address-github-issues
codex plugin add address-github-issues@address-github-issues
```

Use a Codex CLI version that supports `codex plugin` (local installation tested
with version 0.153.4). Restart Codex after installation to load the skill.

## Install from a local checkout

Run these commands from this repository's root:

```bash
codex plugin marketplace add .
codex plugin add address-github-issues@address-github-issues
```

The marketplace loads the bundled plugin using a path relative to the
repository root.

## Use

Open Codex in the target GitHub repository. To process a snapshot of all open
issues, invoke:

```text
$address-github-issues
```

To process one issue, supply its number, `#number`, or full GitHub issue URL:

```text
$address-github-issues 123
```

Only one explicit issue argument is supported. In single-issue mode, referenced
dependencies are inspected but remain outside implementation scope.

## Requirements

- Git, authenticated GitHub access (for example, GitHub CLI via `gh auth login`),
  and permission to push branches, create pull requests, rebase-merge them, and
  close issues in the target repository.
- A Codex harness that supports `gpt-6-astra` subagents with `xhigh` and `low`
  reasoning effort. The skill requires these exact configurations and stops
  when they are unavailable.
- A target repository with an `origin` remote and a clean local `main` that can
  be safely fast-forwarded to `origin/main`. Required checks and branch
  protections must permit rebase merging.
- The existing workflow explicitly requires `cargo fmt`,
  `cargo check --workspace`, and `cargo clippy --workspace --all-targets -- -D warnings`.
  It therefore assumes a Rust workspace with Cargo, rustfmt, and Clippy, in
  addition to the target repository's own validation requirements.

The plugin preserves the original [skill](plugins/address-github-issues/skills/address-github-issues/SKILL.md)
and [workflow](plugins/address-github-issues/skills/address-github-issues/references/workflow.md),
including sequential issue processing, required delegation, and bounded retries.

The skill's core workflow is broadly reusable across AI agents that support
subagent delegation. Its current instructions explicitly reference OpenAI models
and reasoning-effort settings for subagents. To use another provider, update
those references in both the skill and workflow files to equivalent models and
settings supported by that provider and agent harness.

## Publish

Publish this entire repository, including `.agents/plugins/marketplace.json`
and the `plugins/` directory, to
[mrnateriver/skill-address-github-issues](https://github.com/mrnateriver/skill-address-github-issues).
No build step or package registry is needed. The installation commands above
use the repository's default branch;
the GitHub installation can be verified once the repository is published.
