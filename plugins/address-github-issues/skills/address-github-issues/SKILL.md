---
name: address-github-issues
description: Inventory, triage, order, group, and fully resolve a repository's GitHub issues through isolated worktrees and rebase-merged pull requests. Use when asked to address all open issues or one specified issue; do not use for issue summaries or metadata-only edits.
---

# Address GitHub Issues

The invoking root agent inventories the requested issue snapshot, spawns a `gpt-6-astra-xhigh` subagent to triage it, then uses that triage report to run each approved issue or grouped effort through its own fresh coordinator subagent. The root agent does not perform technical triage or implementation work.

## Invocation modes

- No argument: process a snapshot of every open issue in the current repository.
- One issue number, `#number`, or issue URL: process only that issue. Inspect referenced dependencies to determine whether the target is blocked, but do not implement another issue or expand scope.
- More than one explicit issue argument is invalid. Use the no-argument mode for repository-wide processing.

## Mandatory delegation

The root agent first performs the preflight and raw issue inventory in [references/workflow.md](references/workflow.md), then spawns exactly one `gpt-6-astra-xhigh` triage subagent (`model=gpt-6-astra`, `reasoning_effort=xhigh`) with the complete raw inventory. It uses only that decision-complete report to select, group, order, and spawn effort coordinators; it must not pre-classify issues, make triage decisions, or fill gaps in the report.

For every dependency-ready individual issue or deliberately approved grouped effort, the root agent spawns exactly one **fresh** `gpt-6-astra-low` coordinator (`model=gpt-6-astra`, `reasoning_effort=low`). It passes the coordinator the repository path, current raw issue data, the applicable triage report, this skill, and [references/workflow.md](references/workflow.md), and requires it to read both files completely before acting. The coordinator's context ends when that one effort is reported; it must never process a later effort.

Each effort coordinator owns that effort from its eligibility refresh through its report, but it is orchestration-only:

- `gpt-6-astra-xhigh` (`model=gpt-6-astra`, `reasoning_effort=xhigh`) performs all issue triage, dependency analysis, grouping, ordering, research, planning, root-cause analysis, and debugging.
- `gpt-6-astra-low` (`model=gpt-6-astra`, `reasoning_effort=low`) performs all implementation and fix edits in the current issue worktree.
- The coordinator must not independently make any technical triage or debugging judgment.
- Never substitute a different model or reasoning effort. If a required model is unavailable, report the run as blocked.

The root agent waits for each coordinator before refreshing scope and starting the next. It must not run effort coordinators in parallel. A grouped effort is handled by one fresh coordinator, not one coordinator per member issue.

## Execution mode and thread limits

If the invoking root agent requests **FAST MODE subagents**, every subagent in the run must use fast mode, recursively: triage, coordinators, research, debugging, planning, implementation, verification, and remediation. Every agent must propagate the requirement to its children and must never silently fall back to normal mode. If fast mode cannot be guaranteed for a required delegate, stop and report the run as blocked.

If any agent hits the harness's subagent thread limit, it must stop immediately, create the persistent handover artifact specified in [references/workflow.md](references/workflow.md), and request that the human restart the harness to clean up sessions. It must not continue locally, reuse an old delegate, substitute a model, or attempt further issue or GitHub work.

## Required procedure

The root agent and every effort coordinator must follow [references/workflow.md](references/workflow.md) exactly. In all-open mode, the root agent inventories all open issues, delegates triage to `gpt-6-astra-xhigh`, and uses its decision-complete report to sequence one fresh coordinator per effort. A grouped effort is one branch, worktree, implementation, pull request, and merge; grouping is not parallel execution.

Do not bypass repository protections, required checks, model requirements, or unresolved blockers. Stop after the bounded remediation limit in the workflow and report evidence instead of guessing.
