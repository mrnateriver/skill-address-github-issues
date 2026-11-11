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
$address-github-issues [issue-number-or-url] [--no-pr] [--no-commit] [--simple] [--effort max|xhigh|high|medium|low] [--model top|high|middle|low]
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

In the standard workflow, `--effort` overrides the reasoning effort for every
subagent (`low` < `medium` < `high` < `xhigh` < `max`). When the resolved
provider-tier entry defines an explicit reasoning-effort cap, any `--effort`
value (as well as any role default) above that cap is clamped down to the capped
value. After applying any tier policy cap, an explicitly user-supplied `--effort`
must be supported by an eligible model; if unsupported, or if `--effort` is
missing, invalid, or repeated, the run fails rather than silently changing the
user's request.

In the standard workflow, `--model` overrides the model-family tier for every
subagent, including preflight and shared triage, and takes precedence over
labels and automatic fallbacks.
Accepted values are `top`, `high`, `middle`, and `low`. Because `--model` is
explicitly specified by the user, an unavailable or incompatible explicit tier
blocks the run rather than falling back, and invalid, missing, or repeated
values are rejected before GitHub access. By contrast, when tiers, model
families, or reasoning efforts are resolved automatically and do not match
harness reality, the workflow logs a mild warning and continues with
available-range clamping and best-effort fallbacks. For example:

```text
$address-github-issues 123 --model high --effort high --no-pr
```

### Provider model tiers

Model-family tiers map to the active AI provider's latest available model family
rather than specific numeric model versions:

| Tier | OpenAI | Google Gemini |
|---|---|---|
| `top` | GPT Astra (latest) | Gemini Carbon (latest) |
| `high` | GPT Sol (latest) | Gemini Carbon (latest; reasoning effort capped at `medium`) |
| `middle` | GPT Terra (latest) | Gemini Flash (latest) |
| `low` | GPT Luna (latest) | Gemini Flash-Lite (latest) |

When a provider-tier entry specifies a reasoning-effort cap (such as Google
Gemini `high` capped at `medium`), every subagent's target reasoning
effort—whether from its role default or `--effort`—is clamped to
`min(target_effort, cap)` on `low` < `medium` < `high` < `xhigh` < `max`. When
no explicit cap is defined for the selected entry, no policy cap applies.

If the resolved model family does not support an automatically resolved
reasoning effort, the effort is automatically clamped into that model's
available range (for example, `xhigh` clamps down to `high` when a model only
supports `high`–`medium`–`low`, and `low` clamps up to `medium` when a model
only supports `medium`), logging a mild warning and continuing. The workflow
fails on invalid or unsupported model/effort settings only when explicitly
specified by the user.

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
- For the standard workflow, an agent harness supporting subagent delegation.
  Roles default to `xhigh` for analytical and verification work and `medium` for
  coordinators, implementation, and corrective edits unless `--effort` overrides
  them, applying any explicit provider-tier cap first and automatically clamping
  non-user-specified efforts into the resolved model's available effort range
  with a mild warning when needed.
  At the start of each effort, honor `--model` if supplied (blocking if
  unavailable). Otherwise prefer the `top` tier if any member issue has an
  exact, case-sensitive `top` label, or resolve through `high` → `middle` →
  `low`, mapped to the active provider's latest corresponding model family—using
  best-effort fallbacks with a mild warning if automatically resolved tiers or
  families do not match harness reality, and failing only on invalid or
  unsupported user-specified values.
  Every role and subflow in that effort retains its selected tier, model
  family, and effort rules, including tooling recovery and CI fixes. Preflight
  and shared triage honor `--model` or use the `high` → `middle` → `low`
  fallback chain. Delegates propagate the resolved tier, family, caps, and
  warnings and report selection source, evidence, fallback/clamping reasons,
  models, and effective reasoning efforts. The `top` tier is never an automatic
  upward or default fallback. Simple mode uses the current session without
  delegation or model selection.
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
the resolved model-family tier. `--effort` changes subagent reasoning effort
independently of the tier and model family selected for an effort. Simple mode
runs in the current session without these delegation requirements.
