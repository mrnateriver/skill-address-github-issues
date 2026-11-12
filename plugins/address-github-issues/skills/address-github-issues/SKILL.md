---
name: address-github-issues
description: Inventory, triage, order, group, and implement a repository's GitHub issues in isolated worktrees, delivering uncommitted changes, local fast-forwards to main, or rebase-merged pull requests. Use when asked to address all open issues, open issues with a specified label, or one specified issue; do not use for issue summaries or metadata-only edits.
---

# Address GitHub Issues

By default, the invoking root agent delegates verification discovery and issue triage, then runs each approved issue or grouped effort through its own fresh coordinator. In `--simple` mode, the agent receiving the invocation immediately performs every step in the current session without spawning any subagents, including a root or coordinator wrapper.

## Invocation modes

`$address-github-issues [issue-number-or-url | --label <label>] [--no-pr] [--no-commit] [--simple] [--effort max|xhigh|high|medium|low] [--model top|high|middle|low]`

- No issue or label argument: process a snapshot of every open issue in the current repository, including when only other flags are supplied.
- One issue number, `#number`, or issue URL: process only that issue. Inspect referenced dependencies to determine whether the target is blocked, but do not implement another issue or expand scope.
- Specific GitHub issue label (`--label <label>` or a single exact label specified in the user's invocation arguments): fetch and process only open issues carrying that exact label (literal, case-sensitive match); filter out every issue that does not have that exact label. Referenced non-matching issues may be inspected solely for blocker context and remain out of grouping, implementation, and closure scope. All other processing rules apply unchanged.
- Flags may precede or follow the target argument. Reject unknown flags, missing/empty or repeated `--label` values, multiple labels, more than one issue argument, or combining a label filter with an explicit issue argument before accessing GitHub.
- `--simple` uses the separate [simple workflow](references/simple.md): all steps run directly in the current session, with no subagents. It works with every delivery mode and still requires dedicated worktrees.
- `--effort` overrides the reasoning effort of every subagent in the standard workflow. Accept one of `max`, `xhigh`, `high`, `medium`, or `low`; reject missing, invalid, or repeated values before accessing GitHub. When `--effort` is explicitly supplied by the user, apply any explicit provider-tier policy cap (see [Provider model tiers](#provider-model-tiers)) and require an eligible model supporting the resulting effort; fail with a blocker if unsupported rather than clamping into a narrower model range. In `--simple`, validate its syntax but report that it does not reconfigure the current session.
- `--model` overrides model-family tier selection for every subagent in the standard workflow, including preflight and shared triage. Accept one of `top`, `high`, `middle`, or `low`; reject missing, invalid, or repeated values before accessing GitHub. Because it is explicitly specified by the user, it takes precedence over issue labels and automatic fallbacks and fails with a blocker if unavailable, combining with every delivery mode and `--effort`. In `--simple`, validate its syntax but report that it does not reconfigure the current session.

During triage, skip any issue whose labels indicate that human involvement is required, including `human`, `needs-human`, `requires-human`, and labels with the same meaning. This applies in both invocation modes. Report the issue and label; do not group, implement, or close it. Treat an unresolved skipped prerequisite as a blocker for dependent issues.

| Delivery mode | Workspace | Commits | GitHub writes |
|---|---|---|---|
| Default (no flags) | Fresh branch and worktree per effort | Yes | Push, PR, required checks, rebase-merge, issue closure |
| `--no-pr` | Fresh branch and worktree per effort | One per validated issue or approved group; fast-forward local `main` | None |
| `--no-commit` | Dedicated fresh branch and worktree for the run | None; accumulate all changes in one changeset | None |

All work, including preflight, implementation, fixes, and validation, must happen in a dedicated Git worktree in every mode, including `--simple` and reduced delivery modes. Never use the invoking checkout or another effort's worktree for these operations. Preserve its pre-existing edits; only serialized integration may update the checkout holding `main`.

`--no-commit` implies `--no-pr`; if both appear, use uncommitted mode. Resolve the delivery mode before preflight and pass it to every descendant. `--no-pr` rebases each validated effort onto the latest local `main`, resolves conflicts, reruns applicable checks, then fast-forwards `main` with `--ff-only`; once merged into `main`, any worktrees created as part of the `--no-pr` workflow must be unregistered and fully deleted along with all contents on disk (default PR mode and `--no-commit` retain their worktrees). Serialize integration across concurrent agents and runs; never create merge commits on `main`. `--no-commit` leaves the accumulated changes in its dedicated worktree without updating `main`, commits, temporary commits, or stashes. Both reduced modes perform no GitHub writes: no pushes, PRs, remote merges, or issue closure.

## Subagent model selection

This section applies only to the standard workflow. `--simple` retains the current session's model and reasoning effort without model selection or delegation. Fail on invalid or unsupported model/effort values **only** when they were explicitly specified by the user (`--model` or `--effort`); when automatically resolved values do not match harness reality, log a mild warning and continue with best-effort clamping and fallbacks.

At the start of each executable effort, before spawning its coordinator, the root resolves one model-family tier in this precedence order:

1. If `--model` is supplied by the user, use that exact model-family tier (`top`, `high`, `middle`, or `low`), regardless of issue labels. If unavailable or incompatible with user-specified constraints, report a blocker without falling back to another tier.
2. Otherwise, if any member issue in that individual or approved grouped effort has a label whose name is exactly `top`, select the `top` tier. If `top` is unavailable in the harness, log a mild warning and fall back to step 3 rather than failing.
3. Otherwise, select the first available compatible model-family tier in this order: `high` → `middle` → `low`. If none of the mapped provider families match the harness catalog (and no user-specified constraint is violated), log a mild warning and use a best-effort fallback to the closest available compatible model in the harness; report a blocker only if no usable subagent model exists at all or an explicit user constraint cannot be met.

Compare label names literally and case-sensitively; do not trim, normalize, or pattern-match them. `Top`, `top `, and provider-specific model names do not match. Use this effort's labels in the inventory snapshot; labels on other efforts or out-of-scope dependencies never affect its choice. The `top` tier is eligible only through an exact `top` label or explicit `--model top`, never as an automatic upward or default fallback.

Repository-wide preflight and shared triage resolve `<shared-model>` from `--model` when supplied, otherwise from the `high` → `middle` → `low` fallback chain (with the same mild-warning best-effort fallback if needed), applying any reasoning-effort cap and range clamping for that resolved model. Issue labels apply only when resolving each effort's tier, not to the entire run.

Resolve availability from the harness catalog or explicit model/effort errors by mapping each candidate tier to the active AI provider's corresponding model family and any tier-specific reasoning-effort cap in [Provider model tiers](#provider-model-tiers). Do not reject a candidate tier merely because the workflow's default role efforts fall outside that model's supported effort range—automatically resolved efforts clamp into the model's available range instead. Before starting an effort, choose a tier supporting any user-specified constraints and any requested fast mode, then resolve `<chosen-model>` to the latest available compatible model identifier within that family. Every coordinator and descendant must retain that tier, model family, and any reasoning-effort cap through research, debugging, planning, implementation, verification, tooling recovery, and CI fixes. Pass the resolved tier, model family, reasoning-effort cap (if any), actual model, selection source (override, labels, or default), label evidence, and any warnings or fallback/clamping reasons in every handoff; resolve afresh for the next effort. After selection, subflows must not independently switch tiers or families; if an automatically resolved model later becomes unavailable, log a mild warning and use a best-effort fallback rather than failing, blocking only when an explicit user `--model` or `--effort` constraint cannot be satisfied or no usable subagent model remains.

In the standard workflow, roles default to `xhigh` reasoning for analytical and verification work and `medium` for coordinators, implementation, and corrective edits. Apply any explicit provider-tier reasoning-effort cap first (`min(target_effort, cap)` on `low` < `medium` < `high` < `xhigh` < `max`). Then:

- **When `--effort` was not specified by the user (automatic effort):** if the resolved model family does not support the resulting effort, automatically clamp it into the model's supported effort range—clamping down to the model's highest supported effort when above range (for example, `xhigh` → `high` on a model supporting `high`–`medium`–`low`) or clamping up to its lowest supported effort when below range (for example, `low` → `medium` on a model supporting only `medium`), or using the model's native default if it exposes no effort levels—and log a mild warning while continuing.
- **When `--effort` was explicitly specified by the user:** after applying any explicit provider-tier policy cap, require an eligible model that supports that exact effort; fail with a blocker if the model does not support it.

Role boundaries and mandatory delegation never change when efforts are capped or clamped. Supply the actual model identifier and effective reasoning effort as separate spawn arguments.

## Provider model tiers

Match model-family tiers to the active AI provider's latest available model family rather than pinning specific numeric versions:

| Tier | OpenAI | Google Gemini |
|---|---|---|
| `top` | GPT Astra (latest) | Gemini Carbon (latest) |
| `high` | GPT Sol (latest) | Gemini Carbon (latest; reasoning effort capped at `medium`) |
| `middle` | GPT Terra (latest) | Gemini Flash (latest) |
| `low` | GPT Luna (latest) | Gemini Flash-Lite (latest) |

Always resolve the latest compatible version of the mapped family available in the harness catalog. When running on another AI provider, map `top`, `high`, `middle`, and `low` to that provider's equivalent latest model families before applying [Subagent model selection](#subagent-model-selection).

**Reasoning-effort caps:** A provider-tier entry may define an explicit upper cap on reasoning effort (for example, Google Gemini `high` maps to `Gemini Carbon (latest)` with reasoning effort capped at `medium`). Using `low` < `medium` < `high` < `xhigh` < `max`, clamp both role defaults and any user-supplied `--effort` argument to `min(target_effort, cap)`. Under a `medium` cap, `max`, `xhigh`, and `high` clamp to `medium`, `medium` stays `medium`, and `low` stays `low`. If a provider-tier entry defines no explicit cap, the usual flow applies without any cap.

**Available-range clamping and best-effort fallbacks:** If the resolved model family does not offer an automatically resolved reasoning effort, automatically clamp it to the nearest effort in the model's supported range (for example, `xhigh` clamps down to `high` when a model only supports `high`–`medium`–`low`; `low` clamps up to `medium` when a model only supports `medium`), log a mild warning, and continue. Fail only when invalid or unsupported values were explicitly specified by the user; whenever automatically resolved tiers, model families, or efforts do not match harness reality, log a mild warning and proceed with best-effort fallbacks.

## Mandatory delegation

This section's role, delegation, and context handoff requirements apply only to the standard workflow. For `--simple`, follow [simple.md](references/simple.md) and perform every role yourself without spawning agents or creating handoff files.

Treat every subagent as receiving none of its parent's conversation or working context, regardless of harness inheritance settings, and assume every spawn task has a strict input-size limit. Send only short, self-contained instructions directly. For every other handoff, first write the complete required context to an untracked temporary file, then spawn the subagent with the file's absolute path and an instruction to read it completely before acting. Include all required raw data (including local paths to any fetched issue or comment images), reports, decisions, repository and worktree state, and source paths in that file; never rely on inherited context or defer missing context until after the spawn. Keep the file available until the subagent reports, and apply this rule recursively to every descendant.

During the preflight in [references/workflow.md](references/workflow.md), the root agent spawns a dedicated subagent (`model=<shared-model>`, `reasoning_effort=xhigh`) in the dedicated preflight worktree to determine the necessary artifact verification steps from the target repository's instructions, tooling, CI, and artifact types. It waits for that verification report before fetching the raw issue inventory (including downloading any linked or embedded images in issue bodies and comments to untracked temporary files), then spawns exactly one xhigh-reasoning triage subagent using the shared model with the complete raw inventory and verification report. It uses only the triage subagent's decision-complete report to select, group, order, and spawn effort coordinators; it must not pre-classify issues, make triage decisions, or fill gaps in either report.

For every dependency-ready individual issue or deliberately approved grouped effort, the root agent first resolves its model-family tier and provider model family under the policy above, then spawns exactly one **fresh** coordinator (`model=<chosen-model>`, `reasoning_effort=medium`). It passes the coordinator the resolved tier, model family, and any reasoning-effort cap, selection source and evidence, repository and dedicated preflight worktree paths, delivery mode, current raw issue data, applicable triage and verification reports, and prior effort reports. In reduced modes, also pass the recorded starting state and local progress: the latest integrated `main` tip for `--no-pr`, or the dedicated run worktree and accumulated changes for `--no-commit`. Provide this skill and [references/workflow.md](references/workflow.md), and require the coordinator to read both files completely before acting. The coordinator's context ends when that one effort is reported; it must never process a later effort.

Each effort coordinator owns that effort from workspace selection through its report, but it is orchestration-only:

- An xhigh-reasoning subagent (`model=<chosen-model>`, `reasoning_effort=xhigh`, capped/clamped per [Provider model tiers](#provider-model-tiers)) performs all effort-specific verification discovery and refinement, dependency analysis, research, planning, root-cause analysis, and debugging.
- A medium-reasoning subagent (`model=<chosen-model>`, `reasoning_effort=medium`, capped/clamped per [Provider model tiers](#provider-model-tiers)) performs all implementation and fix edits in the selected worktree.
- The coordinator must not independently make any technical triage or debugging judgment.
- Apply the effort's resolved tier, model family, and any reasoning-effort cap or available-range clamping to every role; fallback decisions, effort clamping, and any within-family substitution preserve role boundaries and mandatory delegation.

In default and `--no-pr` modes, when the xhigh-reasoning triage subagent establishes that dependency-ready efforts are completely unrelated, cannot interfere with one another, and are completely safe to implement in parallel, the root agent **must** launch separate parallel coordinator flows, one fresh coordinator per effort. Each flow owns implementation, verification, and delivery, including merging in default mode. Follow the concurrency safeguards in [references/workflow.md](references/workflow.md#parallel-execution-rules); otherwise run efforts sequentially. A grouped effort is handled by one fresh coordinator, not one coordinator per member issue.

Parallel implementation must still produce linear `main` history. Serialize integration into `main` across concurrent agents and runs: rebase-merge every created PR, and use only local fast-forwards for `--no-pr`; never squash-merge or create merge commits. `--no-commit` remains sequential within its dedicated run worktree and never integrates into `main`.

## Execution mode and thread limits

This section applies to the standard workflow's subagents. `--simple` uses the existing session and spawns none.

If the invoking root agent requests **FAST MODE subagents**, every subagent in the run must use fast mode, recursively: preflight, triage, coordinators, research, debugging, planning, implementation, verification, and remediation. Every agent must propagate the requirement to its children and must never silently fall back to normal mode. Resolve a tier whose mapped model family supports fast mode through the model-selection policy before starting; if none is eligible or the resolved family later becomes unusable, stop and report the affected stage or effort as blocked.

If any agent hits the harness's subagent thread limit, it must stop immediately, create the persistent handover artifact specified in [references/workflow.md](references/workflow.md), and request that the human restart the harness to clean up sessions. It must not continue locally, reuse an old delegate, substitute a model, or attempt further issue or GitHub work.

## Required procedure

For `--simple`, follow [references/simple.md](references/simple.md). Otherwise, the root agent and every effort coordinator must follow [references/workflow.md](references/workflow.md) for the selected delivery mode. In standard all-open or label-filtered mode, the root agent inventories the matching open issues, delegates triage with `model=<shared-model>` and `reasoning_effort=xhigh`, and uses its decision-complete report to sequence one fresh coordinator per effort. A grouped effort is one implementation and, in default mode, one branch, worktree, pull request, and merge; grouping is not parallel execution. With `--no-pr`, dependent efforts start from `main` after their prerequisites have been fast-forwarded into it. With `--no-commit`, later efforts use validated local prerequisites in the dedicated run worktree. In either reduced mode, if an attempted effort cannot complete after the allowed remediation, stop further implementation and preserve earlier work and unfinished changes.

In every mode, any images linked or embedded within GitHub issues or comments must also be fetched (into untracked temporary files outside version control, using authenticated GitHub access when needed) and analyzed alongside the text during triage, diagnosis, planning, and verification.

In both workflows, identify tooling failures before implementation through executable preflight checks. Unexpected tooling failures belong in one grouped recovery round per effort, excluded from the five substantive correction cycles. Never fragment setup, test infrastructure, or formatting fixes into separate heavyweight cycles. Apply [Correction cycles and tooling recovery](references/workflow.md#correction-cycles-and-tooling-recovery) across local validation and CI.

Do not bypass repository protections, required checks, model requirements, or unresolved blockers. Stop after the bounded remediation limit in the workflow and report evidence instead of guessing.
