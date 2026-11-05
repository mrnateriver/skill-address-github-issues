---
name: address-github-issues
description: Inventory, triage, order, group, and implement a repository's GitHub issues, with optional uncommitted or local-commit delivery instead of isolated worktrees and rebase-merged pull requests. Use when asked to address all open issues or one specified issue; do not use for issue summaries or metadata-only edits.
---

# Address GitHub Issues

By default, the invoking root agent delegates verification discovery and issue triage, then runs each approved issue or grouped effort through its own fresh coordinator. In `--simple` mode, the root handles inventory and scheduling and spawns one implementation subagent per effort.

## Invocation modes

`$address-github-issues [issue-number-or-url] [--no-pr] [--no-commit] [--simple] [--effort max|xhigh|high|medium|low] [--model family]`

- No issue argument: process a snapshot of every open issue in the current repository, including when only flags are supplied.
- One issue number, `#number`, or issue URL: process only that issue. Inspect referenced dependencies to determine whether the target is blocked, but do not implement another issue or expand scope.
- Flags may precede or follow the issue argument. Reject unknown flags or more than one issue argument before accessing GitHub.
- `--simple` uses the separate [simple workflow](references/simple.md) with exactly one subagent per issue or group and no other subagents. It works with every delivery mode.
- `--effort` overrides the reasoning effort of every subagent in either workflow. Accept one of `max`, `xhigh`, `high`, `medium`, or `low`; reject missing, invalid, or repeated values before accessing GitHub. The requested effort is exact: use a compatible model under the model-selection policy or report a blocker; do not silently lower it.
- `--model` overrides model-family selection for every subagent in either workflow, including preflight and shared triage. Accept exactly one family alias from the table below; reject missing, invalid, or repeated values before accessing GitHub. It takes precedence over issue labels and automatic fallbacks, and combines with every delivery mode and `--effort`.

During triage, skip any issue whose labels indicate that human involvement is required, including `human`, `needs-human`, `requires-human`, and labels with the same meaning. This applies in both invocation modes. Report the issue and label; do not group, implement, or close it. Treat an unresolved skipped prerequisite as a blocker for dependent issues.

| Delivery mode | Workspace | Commits | GitHub writes |
|---|---|---|---|
| Default (no flags) | Fresh branch and worktree per effort | Yes | Push, PR, required checks, rebase-merge, issue closure |
| `--no-pr` | Current worktree and branch for every effort | One per validated issue or approved group | None |
| `--no-commit` | Current worktree for every effort | None; accumulate all changes in one changeset | None |

`--no-commit` implies `--no-pr`; if both appear, use uncommitted mode. Resolve the delivery mode before preflight and pass it to every descendant. Both reduced modes preserve the current branch and worktree, including earlier efforts and pre-existing edits, without creating or switching branches/worktrees or synchronizing with `main`. Do not include unrelated pre-existing changes in commits. Uncommitted mode also forbids temporary commits and stashes. Reduced modes never push, create PRs, merge, or close issues.

## Subagent model selection

At the start of each executable effort, before spawning its coordinator or simple-mode subagent, the root resolves one model family in this precedence order:

1. If `--model` is supplied, use its exact family from the table below, regardless of issue labels. If unavailable or incompatible, report a blocker without falling back to another family.
2. Otherwise, if any member issue in that individual or approved grouped effort has a label whose name is exactly `astra`, use `gpt-6-astra`. If unavailable or incompatible, report a blocker without falling back to another family.
3. Otherwise, use the first available compatible family in this order: `gpt-6.1-sol` → `gpt-6.1-terra` → `gpt-5.6-sol` → `gpt-5.6-terra` → `gpt-5.5`. Report a model-availability blocker only after exhausting this chain.

| `--model` alias | Model family |
|---|---|
| `astra` | `gpt-6-astra` |
| `sol-6.1` | `gpt-6.1-sol` |
| `terra-6.1` | `gpt-6.1-terra` |
| `luna-6.1` | `gpt-6.1-luna` |
| `sol-5.6` | `gpt-5.6-sol` |
| `terra-5.6` | `gpt-5.6-terra` |
| `luna-5.6` | `gpt-5.6-luna` |

Compare label names literally and case-sensitively; do not trim, normalize, or pattern-match them. `Astra`, `astra `, and `gpt-6-astra` do not match. Use this effort's labels in the inventory snapshot; labels on other efforts or out-of-scope dependencies never affect its choice. Astra is eligible only through an exact `astra` label or explicit `--model astra`, never as an automatic fallback.

Repository-wide preflight and shared triage resolve `<shared-model>` from `--model` when supplied, otherwise from the non-Astra fallback chain. Issue labels apply only when resolving each effort's family, not to the entire run.

Resolve availability from the harness catalog or explicit model/effort errors. Before starting an effort, choose a family supporting all its required role configurations and any requested fast mode, then resolve `<chosen-model>` to an available model identifier within it. Every coordinator and descendant must retain that family through research, debugging, planning, implementation, verification, tooling recovery, and CI fixes. Pass the resolved family, actual model, selection source (override, labels, or default), label evidence, and any fallback reasons in every handoff; resolve afresh for the next effort. After selection, subflows must not independently switch families; report a blocker if the resolved family becomes unusable.

In the standard workflow, use `xhigh` reasoning for analytical and verification roles and `medium` for coordinators, implementation, and corrective edits. Supply the actual model identifier and reasoning effort as separate spawn arguments. `--effort` replaces every role's reasoning effort with the exact requested value without changing the selected family or role boundaries.

In `--simple` mode without `--effort`, use the chosen model's highest supported effort in the order `max`, `xhigh`, `high`, `medium`, `low`. Do not skip an available family in the chain merely to obtain a higher effort. Report the family choice and its source, actual model, reasoning effort, and any fallback or within-family substitution.

## Mandatory delegation

This section's role and delegation requirements apply to the standard workflow. For `--simple`, follow [simple.md](references/simple.md) instead; the context handoff rule immediately below still applies.

Treat every subagent as receiving none of its parent's conversation or working context, regardless of harness inheritance settings, and assume every spawn task has a strict input-size limit. Send only short, self-contained instructions directly. For every other handoff, first write the complete required context to an untracked temporary file, then spawn the subagent with the file's absolute path and an instruction to read it completely before acting. Include all required raw data, reports, decisions, repository and worktree state, and source paths in that file; never rely on inherited context or defer missing context until after the spawn. Keep the file available until the subagent reports, and apply this rule recursively to every descendant.

During the preflight in [references/workflow.md](references/workflow.md), the root agent spawns a dedicated subagent (`model=<shared-model>`, `reasoning_effort=xhigh`) to determine the necessary artifact verification steps from the target repository's instructions, tooling, CI, and artifact types. It waits for that verification report before fetching the raw issue inventory, then spawns exactly one xhigh-reasoning triage subagent using the shared model with the complete raw inventory and verification report. It uses only the triage subagent's decision-complete report to select, group, order, and spawn effort coordinators; it must not pre-classify issues, make triage decisions, or fill gaps in either report.

For every dependency-ready individual issue or deliberately approved grouped effort, the root agent first resolves its model family under the policy above, then spawns exactly one **fresh** coordinator (`model=<chosen-model>`, `reasoning_effort=medium`). It passes the coordinator the resolved family, selection source and evidence, repository path, delivery mode, current raw issue data, applicable triage and verification reports, and prior effort reports. In reduced modes, also pass the invoking worktree, starting branch/commit and pre-existing changes, and accumulated local progress. Provide this skill and [references/workflow.md](references/workflow.md), and require the coordinator to read both files completely before acting. The coordinator's context ends when that one effort is reported; it must never process a later effort.

Each effort coordinator owns that effort from workspace selection through its report, but it is orchestration-only:

- An xhigh-reasoning subagent (`model=<chosen-model>`, `reasoning_effort=xhigh`) performs all effort-specific verification discovery and refinement, dependency analysis, research, planning, root-cause analysis, and debugging.
- A medium-reasoning subagent (`model=<chosen-model>`, `reasoning_effort=medium`) performs all implementation and fix edits in the selected worktree.
- The coordinator must not independently make any technical triage or debugging judgment.
- Apply the effort's resolved family to every role; the initial fallback decision and any within-family substitution preserve role boundaries and reasoning effort.

When the xhigh-reasoning triage subagent establishes that dependency-ready efforts are completely unrelated, cannot interfere with one another, and are completely safe to implement in parallel, the root agent **must** launch separate parallel coordinator flows, one fresh coordinator per effort. Each flow owns implementation, verification, and delivery, including merging in default mode. Follow the concurrency safeguards in [references/workflow.md](references/workflow.md#parallel-execution-rules); otherwise run efforts sequentially. A grouped effort is handled by one fresh coordinator, not one coordinator per member issue.

Parallel implementation must still produce linear `main` history. Serialize integration into `main`, and rebase-merge every created PR; never squash-merge or create merge commits. Reduced modes retain their delivery boundaries.

## Execution mode and thread limits

If the invoking root agent requests **FAST MODE subagents**, every subagent in the run must use fast mode, recursively: preflight, triage, coordinators, research, debugging, planning, implementation, verification, and remediation. Every agent must propagate the requirement to its children and must never silently fall back to normal mode. Resolve a family supporting fast mode through the model-selection policy before starting; if none is eligible or the resolved family later becomes unusable, stop and report the affected stage or effort as blocked.

If any agent hits the harness's subagent thread limit, it must stop immediately, create the persistent handover artifact specified in [references/workflow.md](references/workflow.md), and request that the human restart the harness to clean up sessions. It must not continue locally, reuse an old delegate, substitute a model, or attempt further issue or GitHub work.

## Required procedure

For `--simple`, follow [references/simple.md](references/simple.md). Otherwise, the root agent and every effort coordinator must follow [references/workflow.md](references/workflow.md) for the selected delivery mode. In all-open mode, the root agent inventories all open issues, delegates triage with `model=<shared-model>` and `reasoning_effort=xhigh`, and uses its decision-complete report to sequence one fresh coordinator per effort. A grouped effort is one implementation and, in default mode, one branch, worktree, pull request, and merge; grouping is not parallel execution. In reduced modes, later efforts use validated local prerequisites in the same worktree. In either reduced mode, if an attempted effort cannot complete after the allowed remediation, stop further implementation and preserve earlier work and unfinished changes.

In both workflows, identify tooling failures before implementation through executable preflight checks. Unexpected tooling failures belong in one grouped recovery round per effort, excluded from the five substantive correction cycles. Never fragment setup, test infrastructure, or formatting fixes into separate heavyweight cycles. Apply [Correction cycles and tooling recovery](references/workflow.md#correction-cycles-and-tooling-recovery) across local validation and CI.

Do not bypass repository protections, required checks, model requirements, or unresolved blockers. Stop after the bounded remediation limit in the workflow and report evidence instead of guessing.
