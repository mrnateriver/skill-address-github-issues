# Address GitHub Issues

A Codex plugin that packages the Address GitHub Issues skill. It inventories and
triages issues, orders dependencies, and implements each issue or approved group.
By default, it delivers through isolated worktrees and rebase-merged pull requests;
optional flags keep the work in the current worktree.

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

### Delivery modes

```text
$address-github-issues [issue-number-or-url] [--no-pr] [--no-commit]
```

| Mode | Workspace | Commits | Remote delivery |
|---|---|---|---|
| Default | Fresh branch and worktree per issue or approved group | Yes | Push, PR, checks, rebase-merge, issue closure |
| `--no-pr` | Current worktree and branch for the entire run | One per validated issue or approved group | None |
| `--no-commit` | Current worktree for the entire run | None; all changes accumulate in one changeset | None |

For example, prepare all open issues as uncommitted changes, or commit one issue
locally:

```text
$address-github-issues --no-commit
$address-github-issues 123 --no-pr
```

Flags may precede or follow the issue argument. `--no-commit` implies `--no-pr`
and takes precedence if both are supplied. Flags without an issue select all
open issues; unknown flags and multiple issue targets are rejected.

Both reduced modes retain the current branch and worktree without creating new
ones or synchronizing or rebasing against `main`. Fresh coordinators still handle
efforts sequentially, and later issues can use earlier validated local changes.
Pre-existing edits are preserved and unrelated changes are excluded from commits;
unsafe overlaps are reported as blockers. If an attempted effort exhausts its
fix attempts, implementation stops with earlier work and unfinished changes
preserved. Reduced modes leave GitHub issues open and report local validation
separately from omitted PR-only checks and delivery steps.

## Requirements

- Git and authenticated GitHub read access (for example, GitHub CLI via
  `gh auth login`). Default mode additionally requires permission to push
  branches, create pull requests, rebase-merge them, and close issues.
- A Codex harness supporting subagents with the required `xhigh` and `low`
  reasoning efforts. Model selection follows this order: `gpt-6-astra` →
  `gpt-5.6-sol` → `gpt-5.6-terra` → `gpt-5.5`. If a model is unavailable, every
  agent must use the next available model that supports the role's reasoning
  effort and any requested fast mode. Model availability blocks the run only
  when the list is exhausted. Delegates propagate this policy and report the
  actual models used and any fallbacks.
- A target repository with an `origin` remote. Default mode requires a clean
  local `main` that can be safely fast-forwarded to `origin/main`, with required
  checks and branch protections permitting rebase merging. Reduced modes use
  the current worktree, including existing local commits and edits; `--no-pr`
  requires an existing current branch.
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
