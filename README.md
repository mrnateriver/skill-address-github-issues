# Address GitHub Issues

A Codex plugin that packages the Address GitHub Issues skill. It inventories and
triages issues, orders dependencies, and resolves each issue or approved group
through an isolated worktree and a rebase-merged pull request.

## Install from GitHub

Install with:

```bash
codex plugin marketplace add https://github.com/mrnateriver/skill-address-github-issues
codex plugin add address-github-issues@address-github-issues
```

Use a Codex CLI version that supports `codex plugin` (local installation tested
with version 0.153.4). Restart Codex after installation to load the skill.

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
- A Codex harness supporting subagents with the required `xhigh` and `low`
  reasoning efforts. Model selection follows this order: `gpt-6-astra` →
  `gpt-5.6-sol` → `gpt-5.6-terra` → `gpt-5.5`. If a model is unavailable, every
  agent must use the next available model that supports the role's reasoning
  effort and any requested fast mode. Model availability blocks the run only
  when the list is exhausted. Delegates propagate this policy and report the
  actual models used and any fallbacks.
- A target repository with an `origin` remote and a clean local `main` that can
  be safely fast-forwarded to `origin/main`. Required checks and branch
  protections must permit rebase merging.
- The tools and environment needed to verify the target repository's artifacts.
  Before issue inventory and processing, an Astra-xhigh preflight delegate
  determines verification steps from repository instructions, CI, manifests,
  scripts, documentation, and artifact types. Its report guides baseline checks
  and final verification, with effort-specific refinements during planning.
  Checks may be automated or explicit manual reviews. Missing automated tests
  alone are not a blocker, but unavailable required checks or an inability to
  establish meaningful verification must be reported.

The [skill](plugins/address-github-issues/skills/address-github-issues/SKILL.md)
and [workflow](plugins/address-github-issues/skills/address-github-issues/references/workflow.md)
define sequential issue processing, required delegation, and bounded retries.

The skill's core workflow is broadly reusable across AI agents that support
subagent delegation. Its current instructions explicitly reference OpenAI models
and reasoning-effort settings for subagents. To use another provider, update
those references in both the skill and workflow files to equivalent models and
settings supported by that provider and agent harness.

