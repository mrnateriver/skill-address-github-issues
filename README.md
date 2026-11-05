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

Issues labeled `human`, `needs-human`, `requires-human`, or otherwise indicating
that human involvement is required are skipped and reported. Issues that depend
on an unresolved skipped issue remain blocked.

### Delivery modes

```text
$address-github-issues [issue-number-or-url] [--no-pr] [--no-commit] [--simple] [--effort max|xhigh|high|medium|low] [--model family]
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

`--simple` lets the root handle inventory and scheduling, then uses one subagent
per issue or approved group to investigate, implement, verify, and deliver it.
It can be combined with either reduced delivery mode. By default that subagent
uses the highest supported effort on the selected model: `max`, then `xhigh`,
`high`, `medium`, or `low`. For example:

```text
$address-github-issues 123 --simple --no-pr --effort high
```

`--effort` sets the exact reasoning effort for every subagent in either workflow.
If no eligible model supports it, the run reports a blocker rather than silently
lowering the effort. Invalid, missing, or repeated effort values are rejected.

`--model` explicitly overrides the family for every subagent, including preflight
and shared triage, and takes precedence over labels and automatic fallbacks.
Accepted values are `astra`, `sol-6.1`, `terra-6.1`, `luna-6.1`, `sol-5.6`,
`terra-5.6`, and `luna-5.6`. An unavailable or incompatible explicit family blocks
the run rather than falling back. Invalid, missing, or repeated values are rejected
before GitHub access. For example:

```text
$address-github-issues 123 --model sol-6.1 --effort high --no-pr
```

Both reduced modes retain the current branch and worktree without creating new
ones or synchronizing or rebasing against `main`. Efforts run sequentially, and
later issues can use earlier validated local changes.
Pre-existing edits are preserved and unrelated changes are excluded from commits;
unsafe overlaps are reported as blockers. If an attempted effort exhausts its
fix attempts, implementation stops with earlier work and unfinished changes
preserved. Reduced modes leave GitHub issues open and report local validation
separately from omitted PR-only checks and delivery steps.

Both workflows exercise required tooling before implementation. Unexpected setup,
dependency, test infrastructure, formatting, or reporting failures must be fixed
in one grouped recovery round per effort, outside the five substantive correction
cycles shared across local checks and CI. Separate heavyweight cycles for routine
tooling repairs are forbidden; unresolved tooling after that round blocks delivery.

## Requirements

- Git and authenticated GitHub read access (for example, GitHub CLI via
  `gh auth login`). Default mode additionally requires permission to push
  branches, create pull requests, rebase-merge them, and close issues.
- A Codex harness supporting the required subagent reasoning efforts. The standard
  workflow uses `xhigh` for analytical and verification roles and `medium` for
  coordinators, implementation, and corrective edits unless `--effort` overrides
  them; simple mode uses the highest available effort unless overridden.
  At the start of each effort, honor `--model` if supplied. Otherwise select
  `gpt-6-astra` if any member issue has an exact, case-sensitive `astra` label,
  with no fallback to another family. Without either, resolve the first available
  family supporting the required roles and execution mode in this order:
  `gpt-6.1-sol` → `gpt-6.1-terra` → `gpt-5.6-sol` → `gpt-5.6-terra` → `gpt-5.5`.
  Every role and subflow in that effort retains its selected family, including
  tooling recovery and CI fixes. Preflight and shared triage honor `--model` or
  use the default fallback chain. Delegates propagate the resolved family and
  report its selection source, evidence, fallback reasons, models, and reasoning
  efforts. Astra is never an automatic fallback.
- A target repository with an `origin` remote. Default mode requires a clean
  local `main` that can be safely fast-forwarded to `origin/main`, with required
  checks and branch protections permitting rebase merging. Reduced modes use
  the current worktree, including existing local commits and edits; `--no-pr`
  requires an existing current branch.
- The tools and environment needed to verify the target repository's artifacts.
  In the standard workflow, before issue inventory and processing, a preflight delegate
  determines verification steps from repository instructions, CI, manifests,
  scripts, documentation, and artifact types. Its report guides baseline checks
  and final verification, with effort-specific refinements during planning.
  Checks may be automated or explicit manual reviews. Missing automated tests
  alone are not a blocker, but unavailable required checks or an inability to
  establish meaningful verification must be reported. In simple mode the root
  determines these verification steps without a separate delegate.

The [skill](plugins/address-github-issues/skills/address-github-issues/SKILL.md)
and [workflow](plugins/address-github-issues/skills/address-github-issues/references/workflow.md)
define sequential issue processing, required delegation, and bounded retries.

The skill's core workflow requires a harness supporting subagent delegation and
the resolved model family. `--effort` changes reasoning effort independently of
the family selected for an effort.
