---
name: address-github-issues
description: Inventory, triage, order, group, and implement a repository's GitHub issues, with optional uncommitted or local-commit delivery instead of isolated worktrees and rebase-merged pull requests. Use when asked to address all open issues or one specified issue; do not use for issue summaries or metadata-only edits.
---

# Address GitHub Issues

The invoking root agent first obtains repository-specific artifact verification steps during preflight, inventories the requested issue snapshot, spawns an xhigh-reasoning subagent to triage it, then uses that triage report to run each approved issue or grouped effort through its own fresh coordinator subagent. The root agent does not perform technical triage or implementation work.

## Invocation modes

`$address-github-issues [issue-number-or-url] [--no-pr] [--no-commit]`

- No issue argument: process a snapshot of every open issue in the current repository, including when only flags are supplied.
- One issue number, `#number`, or issue URL: process only that issue. Inspect referenced dependencies to determine whether the target is blocked, but do not implement another issue or expand scope.
- Flags may precede or follow the issue argument. Reject unknown flags or more than one issue argument before accessing GitHub.

| Delivery mode | Workspace | Commits | GitHub writes |
|---|---|---|---|
| Default (no flags) | Fresh branch and worktree per effort | Yes | Push, PR, required checks, rebase-merge, issue closure |
| `--no-pr` | Current worktree and branch for every effort | One per validated issue or approved group | None |
| `--no-commit` | Current worktree for every effort | None; accumulate all changes in one changeset | None |

`--no-commit` implies `--no-pr`; if both appear, use uncommitted mode. Resolve the delivery mode before preflight and pass it to every descendant. Both reduced modes preserve the current branch and worktree, including earlier efforts and pre-existing edits, without creating or switching branches/worktrees or synchronizing with `main`. Do not include unrelated pre-existing changes in commits. Uncommitted mode also forbids temporary commits and stashes. Reduced modes never push, create PRs, merge, or close issues.

## Subagent model selection

If the user specifies an LLM model or model family, that choice overrides the default below except for the literal `astra` label rule in this section. Use it for every other subagent and descendant, selecting a compatible model within the requested family when the request names a family rather than an exact model. Do not silently substitute a different family; report a blocker if the requested family cannot support a required reasoning effort or fast mode.

Without a user override, use the first available model in this priority order:

`gpt-5.6-sol` → `gpt-5.6-terra` → `gpt-5.5`.

If the preferred model is unavailable, the agent must select the next available model in this list instead of stopping. Resolve availability from the harness's model catalog or an explicit model-unavailable error. Preserve the role's required reasoning effort (`xhigh` or `low`) and any requested fast mode; select only a model that supports that configuration. Report a model-availability blocker only after exhausting the list.

After fetching the inventory, compare each in-scope issue label's name to the five-character string `astra` using literal, case-sensitive, character-for-character equality. Do not trim, normalize, case-fold, pattern-match, or assess the label's meaning; for example, `Astra`, `astra `, and `gpt-6-astra` do not match. If any in-scope issue in the shared inventory matches, use the `gpt-6-astra` family for the xhigh triage subagent. For every individual or grouped effort containing a matching issue, use the `gpt-6-astra` family for all xhigh debugging, planning, and verification subagents. This label rule overrides an explicit non-Astra user choice and the ordinary fallback policy for those roles; if no `gpt-6-astra` model can satisfy the required reasoning effort and any requested fast mode, block the affected triage or effort instead of using another family. All other roles, including preflight, coordinators, research, implementation, and corrective-edit subagents, retain the ordinary selected model; the label does not switch implementation to Astra.

Replace `<selected-model>` in spawn arguments with the actual model identifier. Pass the user's override or default fallback policy, each in-scope issue's literal `astra` match result, and the resulting role-specific override to every descendant. Record the actual model, reasoning effort, and reason for any override or fallback in delegate reports. Task failures, transient errors, and thread exhaustion do not justify model fallback; follow their existing handling rules.

Except for the literal `astra` label rule, any `gpt-6-astra-*` or Astra role label remaining in this skill or its workflow is shorthand for `<selected-model>` at the stated reasoning effort; it never overrides the user's model or model-family choice.

## Mandatory delegation

Treat every subagent as receiving none of its parent's conversation or working context, regardless of harness inheritance settings, and assume every spawn task has a strict input-size limit. Send only short, self-contained instructions directly. For every other handoff, first write the complete required context to an untracked temporary file, then spawn the subagent with the file's absolute path and an instruction to read it completely before acting. Include all required raw data, reports, decisions, repository and worktree state, and source paths in that file; never rely on inherited context or defer missing context until after the spawn. Keep the file available until the subagent reports, and apply this rule recursively to every descendant.

During the preflight in [references/workflow.md](references/workflow.md), the root agent spawns a dedicated subagent (`model=<selected-model>`, `reasoning_effort=xhigh`) to determine the necessary artifact verification steps from the target repository's instructions, tooling, CI, and artifact types. It waits for that verification report before fetching the raw issue inventory, then spawns exactly one xhigh-reasoning triage subagent with the complete raw inventory and verification report. It uses only the triage subagent's decision-complete report to select, group, order, and spawn effort coordinators; it must not pre-classify issues, make triage decisions, or fill gaps in either report.

For every dependency-ready individual issue or deliberately approved grouped effort, the root agent spawns exactly one **fresh** coordinator (`model=<selected-model>`, `reasoning_effort=low`). It passes the coordinator the repository path, delivery mode, current raw issue data, applicable triage and verification reports, and prior effort reports. In reduced modes, also pass the invoking worktree, starting branch/commit and pre-existing changes, and accumulated local progress. Provide this skill and [references/workflow.md](references/workflow.md), and require the coordinator to read both files completely before acting. The coordinator's context ends when that one effort is reported; it must never process a later effort.

Each effort coordinator owns that effort from workspace selection through its report, but it is orchestration-only:

- An xhigh-reasoning subagent (`model=<selected-model>`, `reasoning_effort=xhigh`) performs all verification discovery and refinement, issue triage, dependency analysis, grouping, ordering, research, planning, root-cause analysis, and debugging.
- A low-reasoning subagent (`model=<selected-model>`, `reasoning_effort=low`) performs all implementation and fix edits in the selected worktree.
- The coordinator must not independently make any technical triage or debugging judgment.
- Apply the subagent model selection policy to every role; model fallback does not change role boundaries or reasoning effort.

When Astra-xhigh triage establishes that dependency-ready efforts are completely unrelated, cannot interfere with one another, and are completely safe to implement in parallel, the root agent **must** launch separate parallel coordinator flows, one fresh coordinator per effort. Each flow owns implementation, verification, and delivery, including merging in default mode. Follow the concurrency safeguards in [references/workflow.md](references/workflow.md#parallel-execution-rules); otherwise run efforts sequentially. A grouped effort is handled by one fresh coordinator, not one coordinator per member issue.

Parallel implementation must still produce linear `main` history. Serialize integration into `main`, and rebase-merge every created PR; never squash-merge or create merge commits. Reduced modes retain their delivery boundaries.

## Execution mode and thread limits

If the invoking root agent requests **FAST MODE subagents**, every subagent in the run must use fast mode, recursively: preflight, triage, coordinators, research, debugging, planning, implementation, verification, and remediation. Every agent must propagate the requirement to its children and must never silently fall back to normal mode. If no model in the fallback list can guarantee fast mode for a required delegate, stop and report the run as blocked.

If any agent hits the harness's subagent thread limit, it must stop immediately, create the persistent handover artifact specified in [references/workflow.md](references/workflow.md), and request that the human restart the harness to clean up sessions. It must not continue locally, reuse an old delegate, substitute a model, or attempt further issue or GitHub work.

## Required procedure

The root agent and every effort coordinator must follow [references/workflow.md](references/workflow.md) for the selected delivery mode. In all-open mode, the root agent inventories all open issues, delegates triage to `gpt-6-astra-xhigh`, and uses its decision-complete report to sequence one fresh coordinator per effort. A grouped effort is one implementation and, in default mode, one branch, worktree, pull request, and merge; grouping is not parallel execution. In reduced modes, later efforts use validated local prerequisites in the same worktree. In either reduced mode, if an attempted effort cannot complete after the allowed remediation, stop further implementation and preserve earlier work and unfinished changes.

Do not bypass repository protections, required checks, model requirements, or unresolved blockers. Stop after the bounded remediation limit in the workflow and report evidence instead of guessing.
