# Address GitHub Issues

A Codex plugin that packages the Address GitHub Issues skill. It inventories and
triages issues, orders dependencies, and implements each issue or approved group.
All modes work in dedicated Git worktrees. By default, delivery uses rebase-merged
pull requests; optional flags deliver local fast-forwards to `main` or uncommitted
changes in a dedicated worktree.

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
| `--no-pr` | Fresh branch and worktree per issue or approved group | One per validated issue or approved group; fast-forward local `main` | None |
| `--no-commit` | Dedicated fresh branch and worktree for the entire run | None; all changes accumulate in one changeset | None |

For example, prepare all open issues as uncommitted changes, or commit one issue
locally:

```text
$address-github-issues --no-commit
$address-github-issues 123 --no-pr
```

Flags may precede or follow the issue argument. `--no-commit` implies `--no-pr`
and takes precedence if both are supplied. Flags without an issue select all
open issues; unknown flags and multiple issue targets are rejected.

`--simple` starts every step directly in the current agent's existing session:
preflight, inventory, triage, investigation, implementation, verification, and
delivery. It spawns no subagents, including a root or coordinator wrapper, and
works with every delivery mode. Work still happens in dedicated Git worktrees.
It keeps the current session's model and reasoning effort; `--model` and
`--effort` are validated but do not reconfigure the session, and this is reported
when those flags are supplied. For example:

```text
$address-github-issues 123 --simple --no-pr
```

In the standard workflow, `--effort` sets the exact reasoning effort for every
subagent. If no eligible model supports it, the run reports a blocker rather than
silently lowering the effort. Invalid, missing, or repeated effort values are rejected.

In the standard workflow, `--model` overrides the family for every subagent,
including preflight and shared triage, and takes precedence over labels and
automatic fallbacks.
Accepted values are `astra`, `sol-6.1`, `terra-6.1`, `luna-6.1`, `sol-5.6`,
`terra-5.6`, and `luna-5.6`. An unavailable or incompatible explicit family blocks
the run rather than falling back. Invalid, missing, or repeated values are rejected
before GitHub access. For example:

```text
$address-github-issues 123 --model sol-6.1 --effort high --no-pr
```

Every mode uses dedicated worktrees for preflight, implementation, fixes, and
validation, preserving pre-existing edits in the invoking checkout. `--no-pr`
supports independent efforts in parallel, each in its own branch and worktree.
Before delivery, agents serialize integration across concurrent agents and runs,
rebase onto the latest local `main`, resolve conflicts, rerun applicable checks,
and fast-forward `main` with `--ff-only`. No merge commits are created on `main`.
Dependent efforts start from `main` after their prerequisites are integrated.

`--no-commit` keeps efforts sequential in one dedicated run worktree, where later
issues can use earlier validated changes. It leaves those changes uncommitted
and does not update `main`. If an attempted effort exhausts its fix attempts in
either reduced mode, implementation stops with earlier work and unfinished
changes preserved. Reduced modes leave GitHub issues open and report local
validation separately from omitted PR-only checks and delivery steps.

Both workflows exercise required tooling before implementation. Unexpected setup,
dependency, test infrastructure, formatting, or reporting failures must be fixed
in one grouped recovery round per effort, outside the five substantive correction
cycles shared across local checks and CI. Separate heavyweight cycles for routine
tooling repairs are forbidden; unresolved tooling after that round blocks delivery.

## Requirements

- Git and authenticated GitHub read access (for example, GitHub CLI via
  `gh auth login`). Default mode additionally requires permission to push
  branches, create pull requests, rebase-merge them, and close issues.
- For the standard workflow, a Codex harness supporting the required subagent
  reasoning efforts. It uses `xhigh` for analytical and verification roles and
  `medium` for coordinators, implementation, and corrective edits unless `--effort`
  overrides them.
  At the start of each effort, honor `--model` if supplied. Otherwise select
  `gpt-6-astra` if any member issue has an exact, case-sensitive `astra` label,
  with no fallback to another family. Without either, resolve the first available
  family supporting the required roles and execution mode in this order:
  `gpt-6.1-sol` → `gpt-6.1-terra` → `gpt-5.6-sol` → `gpt-5.6-terra` → `gpt-5.5`.
  Every role and subflow in that effort retains its selected family, including
  tooling recovery and CI fixes. Preflight and shared triage honor `--model` or
  use the default fallback chain. Delegates propagate the resolved family and
  report its selection source, evidence, fallback reasons, models, and reasoning
  efforts. Astra is never an automatic fallback. Simple mode uses the current
  session without delegation or model selection.
- A target repository with an `origin` remote and local `main` as the worktree
  base in every mode. Default mode requires a clean local `main` that can be
  safely fast-forwarded to `origin/main`, with required checks and branch
  protections permitting rebase merging. `--no-pr` requires a clean checkout
  holding local `main` for fast-forward integration, but does not require it to
  match `origin/main`. `--no-commit` never updates `main`. Pre-existing edits
  remain in their original checkout; they are not copied into run worktrees.
- The tools and environment needed to verify the target repository's artifacts.
  In the standard workflow, before issue inventory and processing, a preflight delegate
  determines verification steps from repository instructions, CI, manifests,
  scripts, documentation, and artifact types. Its report guides baseline checks
  and final verification, with effort-specific refinements during planning.
  Checks may be automated or explicit manual reviews. Missing automated tests
  alone are not a blocker, but unavailable required checks or an inability to
  establish meaningful verification must be reported. In simple mode the current
  agent determines these verification steps directly.

The [skill](plugins/address-github-issues/skills/address-github-issues/SKILL.md)
and [workflow](plugins/address-github-issues/skills/address-github-issues/references/workflow.md)
define issue processing, standard-workflow delegation, and bounded retries.

The standard workflow requires a harness supporting subagent delegation and
the resolved model family. `--effort` changes subagent reasoning effort
independently of the family selected for an effort. Simple mode runs in the
current session without these delegation requirements.
