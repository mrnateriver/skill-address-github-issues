# GitHub issue workflow

This is the standard workflow. For `--simple`, use [simple.md](simple.md) instead. When `--effort` is supplied, every specified subagent reasoning effort in this workflow is replaced by that exact value; role responsibilities and model-selection rules remain unchanged.

The invoking root agent owns scope resolution, preflight, and raw inventory. During preflight it delegates artifact verification discovery to an xhigh-reasoning subagent using `<shared-model>`; after inventory it spawns one such subagent for all technical triage. Using only that triage report, it resolves each effort's model family under the model-selection policy and spawns one fresh medium-reasoning coordinator using `<chosen-model>` for that individual issue or approved group. An effort coordinator owns only that one effort.

## Non-negotiable role boundaries

An effort coordinator is a control-plane agent. It may read instructions, refresh its effort's raw GitHub and Git data, spawn and wait for subagents, run prescribed validation and Git/GitHub commands, and report that effort's outcome. Workspace creation, commits, pushes, PRs, and merges are permitted only as specified by the delivery mode in [Invocation modes](../SKILL.md#invocation-modes). Pass that mode to every descendant; no delegate may advance beyond its delivery boundary.

The coordinator must not:

- classify or prioritize issues;
- infer blockers, dependencies, shared root causes, or grouping opportunities;
- diagnose an issue, failed baseline, test failure, CI failure, merge conflict, or unexpected behavior;
- choose an algorithm, protocol, library, architecture, implementation approach, or fix;
- plan or edit implementation files.

Delegate every effort-specific analytical or debugging judgment to a subagent with `model=<chosen-model>` and `reasoning_effort=xhigh`. Delegate every implementation or corrective edit to a subagent with `model=<chosen-model>` and `reasoning_effort=medium`. The coordinator relays evidence and executes the resulting decision-complete reports.

Determining or revising verification steps and performing technical manual reviews also require an xhigh-reasoning subagent. The root and coordinators relay the verification report and run prescribed checks; they must not independently choose or waive verification requirements.

Apply [Subagent model selection](../SKILL.md#subagent-model-selection) before each effort and propagate the resolved family recursively to every subflow. `<shared-model>` identifies a model resolved from `--model` or the default fallback chain for repository-wide preflight and shared triage. `<chosen-model>` identifies a model resolved for the current effort from `--model`, its issue labels, or the default fallback chain, in that order. Resolve availability for all required role configurations before starting the effort. Every role in that effort uses the resulting family, including implementation and remediation; subflows must not independently restart family selection.

## Execution mode inheritance and thread exhaustion

When the invoking root explicitly requests **FAST MODE subagents**, fast mode is mandatory for every subagent spawned anywhere in the run. The root must apply it to preflight, triage, and effort coordinators; each coordinator and descendant must state and propagate it in every child task. The model selection policy and exact reasoning-effort requirements still apply. Never spawn a normal-mode fallback. Resolve a family supporting fast mode through the model-selection policy before starting; if none is eligible or the resolved family later becomes unusable, stop before that delegate's work and report the affected stage or effort as blocked.

If any agent receives a subagent-thread-limit error or otherwise cannot spawn a required delegate because the harness's thread capacity is exhausted, it must stop immediately. It must not diagnose, plan, implement, validate, publish, merge, reuse a previous delegate, or wait for capacity to recover.

Before stopping, that agent must write a persistent Markdown handover under the repository's ignored `.worktrees/address-github-issues-handovers/` directory. Use a UTC timestamp and the agent task name in the filename. The artifact must record:

- repository path, invocation scope, delivery mode, inventory timestamp, and issue snapshot;
- agent role/task path, current effort, workflow stage, and the exact thread-limit error;
- branch, worktree, commit, push, pull-request, check, merge, and issue states;
- completed delegates and the locations or full contents of their reports;
- commands/checks already run, their results, current Git status, and any uncommitted files;
- remaining work and the single next safe action after restart.

Do not include credentials or other secrets. After writing the artifact, report its absolute path to the parent/root and request that the human restart the harness to clean up sessions. The root must cancel all active issue flows and surface that restart request; no agent may resume the run in the exhausted harness.

## 1. Root agent: resolve scope and delivery mode

Interpret `$address-github-issues [issue-number-or-url] [--no-pr] [--no-commit] [--simple] [--effort max|xhigh|high|medium|low] [--model family]` before accessing GitHub, using the flag validation in [Invocation modes](../SKILL.md#invocation-modes):

- Parse flags independently of the optional issue target, in either order. `--no-commit` implies `--no-pr` and takes precedence when both are present. Reject unknown flags.
- Validate `--model` against the exact aliases in the skill and reject missing, invalid, or repeated values before GitHub access. Propagate it and any `--effort` override to every delegate.
- No issue argument selects all issues that are open when the inventory is taken, even when flags are supplied. This is a finite snapshot; issues opened later are not added to the active run.
- A decimal number, `#number`, or full GitHub issue URL selects exactly one issue.
- For a URL, verify that its owner and repository match the current checkout's `origin`. Stop on a mismatch.
- A single selected issue may read referenced issues for blocker context, but it must not implement, close, group with, or otherwise expand into them.
- Reject ambiguous or multiple explicit issue arguments instead of guessing.

All modes require dedicated Git worktrees for preflight, implementation, fixes, and validation; never perform this work in the invoking checkout. Default mode delivers through a fresh branch/worktree and a merged PR per effort. `--no-pr` also uses a fresh branch/worktree per effort, then rebases and fast-forwards its validated commits into local `main` as specified in section 5.7. `--no-commit` uses one dedicated fresh branch/worktree for the entire run and leaves its accumulated changes there without updating `main`, commits, temporary commits, or stashes. Both reduced modes perform only read operations against GitHub and leave issues open.

The root agent schedules efforts according to the triage report and the parallel execution rules below. In default and `--no-pr` modes, proven independent efforts must run through separate parallel coordinator flows; otherwise wait for the current coordinator to report before starting the next effort. A pre-approved multi-issue group counts as one effort.

## 2. Root agent: preflight

Before fetching issue details:

1. Resolve the repository root, `origin` URL, invoking worktree, local `main`, and any worktree holding it. Record the starting branch, commit, staged/unstaged diffs, and untracked files so pre-existing changes remain distinguishable from the run's work.
2. Read the repository's complete `AGENTS.md` and any instructions it routes to for the affected area.
3. Verify GitHub authentication and read access to the repository. Only default mode requires permission to push, create PRs, rebase-merge, and close issues.
4. Require local `main` as the worktree base in every mode. Default and `--no-pr` modes require a clean checkout holding `main` for integration; if needed, create a dedicated worktree holding `main`. Default mode also requires a safe fast-forward to `origin/main`; report a blocker if local `main` is ahead or diverged. Reduced modes may use local `main` with existing local commits without matching `origin/main`. Never stash, reset, discard, or overwrite user work, or copy pre-existing edits into new worktrees. If an effort requires those edits or commits from another branch, report that prerequisite instead of silently including them.
5. In default mode only, acquire the repository-wide integration lock described below, fetch `origin/main`, fast-forward local `main` with `--ff-only`, verify equality, and release the lock. In every mode, create a dedicated preflight worktree on a fresh branch from local `main` before running checks or making edits. Use a unique run identifier in its branch and path. `--no-commit` retains this worktree for all efforts; other modes create separate effort worktrees in section 5.1. Verify any project-local worktree directory is ignored, or use a safe external directory.
6. Resolve `<shared-model>` for repository-wide preflight and shared triage from `--model` when supplied, otherwise the default fallback chain. Defer each effort's label-based family selection until that effort starts; never apply one effort's labels to the whole run.
7. Spawn a dedicated preflight subagent (`model=<shared-model>`, `reasoning_effort=xhigh`) with the dedicated preflight worktree path, delivery mode, recorded starting state, repository instructions, and this workflow to determine the necessary artifact verification steps. Wait for its report before fetching issue details.

The preflight subagent inspects repository instructions, CI configuration, manifests, existing scripts, documentation, and artifact types in its dedicated worktree. Its verification report must identify applicable automated checks or explicit manual review procedures, working directories, prerequisites, execution stage (local or CI), and success criteria, with supporting repository evidence. Distinguish baseline checks from final artifact verification and identify required CI checks. In reduced modes, identify PR-only checks as omitted by the delivery mode, not passed or local prerequisites. Verification commands must respect the selected mode's Git and GitHub write boundaries. Derive verification from the target repository; do not assume a particular language, build system, or test framework.

Preflight must exercise the required tooling, not merely list commands: check runtime and package-manager availability, dependency installation and lockfile consistency, sandbox/store/cache write permissions, and applicable test, type-check, formatter, and reporting commands with small representative runs. Include mutation tooling only when required by the verification plan. Identify and resolve setup and tooling failures together before implementation, using the existing diagnosis/edit role boundaries; report exact commands, outcomes, and unresolved prerequisites. Do not defer predictable tooling failures to correction cycles.

When verification procedures are undocumented, derive suitable checks from the artifacts and existing tooling, including manual review where appropriate. Missing automated tests alone do not block the run. If meaningful verification cannot be established or required tooling or other prerequisites are unavailable, report the blocker before issue processing; do not invent commands or treat unperformed checks as passing. Carry this report through the existing delegate-report flow.

In default mode, the root repeats the locked safe fetch and fast-forward before each later effort because the previous rebase merge changes `main`. With `--no-pr`, each effort starts from the latest local `main`, including previously integrated prerequisites. With `--no-commit`, pass the evolving dedicated run worktree and prior effort reports onward without synchronization or cleanup. Preserve pre-existing edits and staging in their original checkout.

## 3. Root agent: fetch the issue inventory

Fetch only after the mode-specific preflight is complete and its verification report is available with no unresolved preflight blockers.

### All-open mode

1. Enumerate every open issue with pagination until no page remains. Do not rely on GitHub CLI's default limit, and exclude pull requests returned by the Issues API.
2. For every issue, collect at least its number, title, body, state, creation and update timestamps, labels, assignees, milestone, URL, comments, linked closing pull requests, and explicit issue relationships available from GitHub.
3. Preserve the raw issue text and relationship data. The root agent must not summarize it before handing it to the triage subagent.
4. Record the inventory timestamp and issue-number snapshot. Newly opened issues are listed in the final report as outside this run.

### Single-issue mode

1. Fetch the selected issue and the same complete fields.
2. If it is not open, report its state and stop without making changes.
3. Fetch raw metadata for issues explicitly referenced as blockers or dependencies, but keep those issues out of implementation scope.

## 4. Root agent: mandatory xhigh-reasoning triage

Spawn one shared triage subagent (`model=<shared-model>`, `reasoning_effort=xhigh`) with the raw inventory, delivery mode, preflight verification report, repository instructions, and dedicated preflight worktree path. Shared triage honors `--model` when supplied; otherwise it uses the default fallback chain. Apply issue-label selection separately at the start of each resulting effort when there is no override. In reduced modes, include the recorded starting state and local progress. The root agent must not pre-classify the issues, suggest an ordering, or make any triage decision.

The triage subagent applies the human-involvement label rule in [Invocation modes](../SKILL.md#invocation-modes) to each in-scope issue using the inventory labels, including semantically equivalent labels. Record skipped issues and the relevant labels; exclude them from groups and executable efforts. A dependent issue remains blocked if a skipped prerequisite is unresolved. If the sole selected issue is skipped, report it without implementation or GitHub writes. Skipped issues do not prevent unrelated eligible efforts from proceeding.

The triage report must be detailed and decision-complete. It must contain:

- each issue's scope and acceptance criteria;
- applicable baseline and final verification steps for each effort, derived from the preflight verification report;
- explicit blockers and dependencies cited by issue data;
- skipped issues and their human-involvement labels, including any dependent issues blocked by them;
- technical prerequisites inferred from the codebase, protocols, migrations, or rollout order, with evidence;
- likely shared root causes and overlapping implementation surfaces;
- candidate multi-issue groups and an explicit justification or rejection for each plausible group;
- whether external research is beneficial for each effort and why;
- a dependency graph, blocked reason for every blocked node, and the ordered list of efforts;
- the oldest creation timestamp and member issue numbers for every grouped effort;
- explicit parallel-safe sets of dependency-ready efforts, with evidence that they are completely unrelated and cannot interfere through implementation, contracts, migrations, generated artifacts, validation resources, or delivery; explain why remaining efforts require sequential execution.

Triage and debugging require xhigh reasoning. The root agent may ask the triage subagent to clarify an incomplete report, but it must not fill gaps itself.

### Dependency ordering

Use an edge from prerequisite to dependent. Contract approved multi-issue groups before ordering. Then:

1. Put efforts with no unresolved prerequisites in dependency level 0.
2. Put each remaining acyclic effort in level `1 + max(level of each prerequisite)`.
3. Put efforts blocked by an unresolved external dependency after all executable levels.
4. Put dependency cycles last and report the cycle; never invent an order that pretends the cycle is resolved.
5. Within the same level and blocker class, sort by oldest creation timestamp first, then by lowest issue number.

This produces unblocked-to-most-blocked ordering with oldest-to-newest ordering as the deterministic tie-breaker.

In reduced modes, the xhigh-reasoning subagent may mark a dependency ready once its prerequisite is validated and fast-forwarded into local `main` with `--no-pr`, or validated in the dedicated run worktree with `--no-commit`, even if the prerequisite issue remains open. Use prior effort reports and current artifacts as evidence; issue state alone does not settle local readiness. External blockers or prerequisites that genuinely require remote integration or deployment remain blockers. Failed or unfinished effort changes never satisfy a dependency.

### Grouping rules

Group issues only when the triage report establishes all of the following:

- one cohesive implementation or shared root-cause fix satisfies every member;
- one cohesive effort is clearer than separate delivery (one branch and PR in default mode, one commit with `--no-pr`);
- members have compatible acceptance criteria, rollout, and validation;
- no member requires another member to merge first or expose an intermediate result;
- the combined change satisfies every member without partial delivery, with issue closure performed only in default mode;
- grouping does not hide unrelated refactoring or broaden scope.

Shared labels, the same component, nearby files, or potential merge conflicts are not sufficient reasons to group. When uncertain, keep issues separate. In single-issue mode, grouping is forbidden.

### Parallel execution rules

In default and `--no-pr` modes, when the triage report proves that dependency-ready efforts are completely unrelated, do not interfere in any way, and are completely safe to implement concurrently, the root **must** spawn their fresh coordinators in parallel. Each coordinator runs the full flow below through its mode's delivery boundary. Different issue numbers, disjoint files, or the absence of dependency edges alone do not prove independence. If evidence is missing, obtain triage clarification; do not assume safety.

Use a separate branch and worktree per effort and isolate any mutable validation resources. The root grants only one coordinator at a time the final integration turn in section 5.7 (`--no-pr`) or 5.8 (default). Serialize all updates to local `main` across concurrent agents and separate runs with a repository-wide exclusive lock at `<git-common-dir>/address-github-issues-integration.lock`, using the repository's locking mechanism or an OS file lock. Resolve the shared Git directory with `git rev-parse --git-common-dir`; a per-worktree lock or an in-memory turn in one root is insufficient. Hold the lock through the final base read, rebase, conflict resolution, validation, and integration; release it on success or an abandoned attempt. Other worktrees may continue independent implementation while waiting. All created PRs must be rebase-merged and local `--no-pr` integration must use `--ff-only`, with no squash merges or merge commits on `main`.

Use the triage ordering to launch parallel-safe efforts deterministically. An effort with prerequisites waits for their successful delivery. If new evidence invalidates independence, pause the affected flows and obtain revised triage before continuing them. `--no-commit` efforts remain sequential within their dedicated run worktree because they share an index and accumulated changes; separate runs still use separate worktrees.

## 5. Root agent: process each effort

At the start of each effort selected, grouped, and ordered by the triage report, the root resolves `<chosen-model>` through [Subagent model selection](../SKILL.md#subagent-model-selection): explicit `--model` first, then that effort's labels, otherwise the default fallback chain. It then spawns a **new** coordinator (`model=<chosen-model>`, `reasoning_effort=medium`) with the resolved family, selection source and evidence, fallback reasons, repository and dedicated preflight worktree paths, delivery mode, raw inventory, triage report, latest verification report, prior effort reports, repository instructions, this workflow, and the effort definition. Every descendant in sections 5.1–5.9 inherits that family without reselecting it. In reduced modes, also pass the recorded starting state and local progress: the latest integrated `main` tip for `--no-pr`, or the dedicated run worktree and accumulated changes for `--no-commit`. The fresh coordinator follows the applicable steps below and returns an effort report at the selected delivery boundary. The root agent does not reuse that coordinator or its family choice for another effort and does not make technical decisions between efforts.

The inventory and combined triage are the sole eligibility decision for the run; coordinators do not refresh issue or dependency state before starting. This deliberately accepts a TOCTOU risk: an issue may be closed, edited, or newly blocked after triage, so work can begin from stale GitHub state. Final validation, current-`main` rebasing, and issue-state reporting still apply, but they do not eliminate that risk.

### 5.1 Select the workspace and run the baseline

With `--no-commit`, keep using the dedicated run worktree created during preflight and skip steps 1–4 below. Record the state before this effort, including prior efforts, so its own changes can be reviewed separately. Never reset or clean that worktree between efforts.

In default and `--no-pr` modes:

1. Create a fresh branch from the verified current `main`; never reuse an old issue branch.
2. Use `issue-<number>-<short-slug>-<run-id>` for one issue and `issues-<lowest-number>-<next-number>-<short-slug>-<run-id>` for a group, with a unique run identifier to avoid concurrent branch/path collisions.
3. Create a dedicated Git worktree for that branch. Prefer a repository-native worktree mechanism when available; otherwise use an existing ignored `.worktrees/` directory or a safe external sibling directory.
4. Verify a project-local worktree directory is ignored before using it. Do not add worktree contents to version control.

In every mode, run applicable baseline checks from the verification and triage reports in the selected dedicated worktree before implementation, including every repository-required baseline check. `--no-commit` checks the evolving run state, including earlier validated changes, without requiring a clean worktree. Delegate any prescribed technical manual review to a subagent with `model=<chosen-model>` and `reasoning_effort=xhigh`.

If baseline validation fails, capture exact commands and complete output and delegate diagnosis to a subagent with `model=<chosen-model>` and `reasoning_effort=xhigh`. Do not let the coordinator diagnose or waive a red baseline. Proceed only if that subagent proves the failure is unrelated and the governing repository instructions permit proceeding; otherwise report the effort blocked.

### 5.2 Research external context

When the triage report marks research beneficial, spawn a dedicated research subagent (`model=<chosen-model>`, `reasoning_effort=xhigh`) before planning. Give it the issue data, triage report, repository instructions, and precise research questions.

Research may cover existing solutions and libraries, algorithms, protocols, standards, compatibility constraints, security guidance, performance or quality metrics, and baseline measurements. Require a detailed report with primary sources, conclusions, alternatives, risks, and concrete implications for the plan. The coordinator passes the report onward without replacing its conclusions.

### 5.3 Diagnose the issue

For bugs, regressions, failures, performance problems, or unclear behavior, spawn a debugging subagent (`model=<chosen-model>`, `reasoning_effort=xhigh`) before planning. Require it to inspect the real code path, reproduce when feasible, trace callers and data flow, compare working patterns, and provide an evidence-backed root-cause report.

For a pure feature or documentation issue, the triage report may explicitly state that separate debugging is unnecessary. The coordinator cannot make that determination.

No fix may be planned from symptoms alone. If root cause remains unknown, report the effort blocked rather than asking the implementation subagent to guess.

### 5.4 Produce the implementation plan

Spawn a fresh planning subagent (`model=<chosen-model>`, `reasoning_effort=xhigh`). Give it:

- the complete current issue or grouped-issue data;
- the triage report;
- the latest verification report;
- every research and debugging report;
- current repository instructions and architecture documents;
- the delivery mode, selected worktree, and baseline result; in reduced modes, also the state before the effort and prior effort reports.

Require a decision-complete plan covering scope, code or documentation changes, interfaces, edge cases, migration or compatibility needs, tests, acceptance criteria, and validation. The planner refines the verification report for the effort's artifacts and acceptance criteria, adding effort-specific checks and updating procedures when requirements change. The plan must identify how one grouped change satisfies each member issue separately. The coordinator may request clarification but must not invent missing technical decisions.

Before implementation, exercise any effort-specific tooling not covered by preflight in the selected worktree. Include predictable dependency/lockfile refreshes caused by manifest or export changes in the plan, and perform them with those edits rather than discovering them in later review rounds.

### 5.5 Implement in the worktree

Spawn an implementation subagent (`model=<chosen-model>`, `reasoning_effort=medium`) in the selected worktree. Give it the delivery mode, approved plan, and all supporting reports. Require it to:

- read and obey repository and relevant skill instructions;
- edit only within the approved scope;
- preserve earlier effort changes in the dedicated `--no-commit` run worktree and leave pre-existing user edits in their original checkout untouched;
- use test-driven development for behavior changes;
- avoid unrelated refactors or speculative abstractions;
- run focused checks while implementing;
- return the changed paths, checks run, and any deviation or blocker.

The coordinator must not edit implementation files. If implementation reveals a missing technical decision, return to the xhigh-reasoning planner instead of letting the implementation subagent or coordinator guess.

### 5.6 Review and validate

1. Collect the diff and implementation report. With `--no-commit`, distinguish this effort's delta from prior efforts and assess regressions against the accumulated run state. With `--no-pr`, review the effort against its `main` base and validate again after the final rebase.
2. Give them and the latest verification report to a verification subagent (`model=<chosen-model>`, `reasoning_effort=xhigh`) to verify the final implementation against each issue's acceptance criteria, grouped-issue completeness, regressions, and missing tests, and perform any prescribed technical manual reviews.
3. Consolidate all findings into a grouped correction request for a corrective-edit subagent (`model=<chosen-model>`, `reasoning_effort=medium`), following the correction rules below.
4. Execute the applicable local verification steps from the latest report and implementation plan, including all focused tests and repository-required local checks. Record results against the stated success criteria; unavailable or unperformed required local checks remain blockers. In default mode, run CI-only checks through section 5.8; their results gate merging. In reduced modes, report PR-only checks as omitted by the mode, never as passed.
5. Confirm only intended changes were introduced by this effort, with earlier work and pre-existing edits preserved, and run `git diff --check`.

Collect unexpected results from applicable checks that can run safely into one evidence bundle for a debugging subagent (`model=<chosen-model>`, `reasoning_effort=xhigh`). Only after consolidated diagnosis may a corrective-edit subagent (`model=<chosen-model>`, `reasoning_effort=medium`) implement the prescribed fixes. The coordinator never diagnoses failures itself.

#### Correction cycles and tooling recovery

Allow at most five substantive diagnose-fix-verify cycles per effort across local validation and CI, using one cumulative count. A cycle diagnoses the collected defects, applies the grouped fixes, and verifies the result; individual failed commands, test files, assertions, or agent handoffs are not separate cycles. After the fifth unsuccessful substantive cycle, stop the effort and report the accumulated evidence. Do not stack speculative fixes or bypass checks.

Unexpected tooling failures must be addressed in **one grouped tooling-recovery round per effort, excluded from the five-cycle limit**. This covers setup and permission failures, dependency/lockfile snapshot refreshes, formatting errors, test-harness or fixture mechanics, and runner/reporting failures. Gather the failures, use one consolidated diagnosis handoff and one corrective-edit handoff, then rerun the affected checks. Preflight setup is also outside the substantive count. Keep each handoff concise under the skill's context handoff rule.

**Do not fragment routine repairs into separate heavyweight debugging, planning, and review pipelines, or restart the full workflow for each tooling failure.** If tooling remains broken after the grouped recovery round, or another tooling failure requires a second round, stop and report the blocker; do not create unlimited exempt rounds or charge tooling repairs to the substantive limit. Existing thread-exhaustion and permission safeguards still apply.

Failures exposing incorrect implementation behavior, consumer incompatibilities, or missing behavioral coverage remain substantive corrections, even when discovered by tests or mutation checks. Never relabel them as tooling to evade the limit. A mixed round counts as a substantive cycle and uses the tooling-recovery allowance if it also repairs tooling.

In either reduced mode, an attempted effort that cannot complete after the permitted remediation stops further implementation in the run. Preserve earlier integrated commits or accumulated uncommitted work and unfinished changes in their worktrees; report previously validated efforts separately from the failed effort. Do not integrate an unfinished `--no-pr` effort, roll back earlier work, or start further issues. Stop active parallel efforts at a safe boundary and preserve their work. Issues skipped or blocked during triage without edits do not prevent other eligible efforts from running.

### 5.7 Deliver according to the selected mode

- `--no-commit`: after successful local validation, leave every effort's changes in the accumulated changeset in the dedicated run worktree. Create no commits, temporary commits, or stashes, and do not update `main`. Proceed directly to section 5.9.
- `--no-pr`: after successful local validation, create one commit for this issue or approved group on its dedicated worktree branch, with an imperative, non-Conventional-Commit message following repository conventions. Include `Closes #<issue number>` for every original GitHub issue delivered by the effort. Commit only this effort's changes. Complete the local integration sequence below before reporting success. Do not push or create a PR.

#### Local integration with `--no-pr`

1. Obtain the root's exclusive integration turn and the repository-wide integration lock. Verify the worktree holding `main` is clean and record its current tip as the integration base. If it is dirty, preserve all work and report the exact blocker; never stash or reset it.
2. In the effort's worktree, rebase its new commits onto that current local `main` tip. Concurrent progress is expected: resolve rebase conflicts rather than treating divergence alone as a blocker. An xhigh-reasoning debugging subagent diagnoses the correct resolution and a medium-reasoning corrective-edit subagent applies it, both using `<chosen-model>`; the simple workflow's sole subagent performs both roles itself. Preserve both the effort's intent and concurrent changes.
3. Rerun applicable verification from the latest report on the rebased result, including any conflict resolution. Follow section 5.6 for failures and bounded correction cycles; never integrate failing or unfinished work. Keep fixes in this effort's commit. Require `git rev-list --merges <integration-base>..HEAD` to return no commits.
4. Recheck that local `main` still equals the integration base and its checkout is clean. If another writer advanced it despite the lock, repeat the rebase and verification against the new tip. In the worktree holding `main`, run `git merge --ff-only <effort-branch>`. Never fall back to a merge commit, reset, or force-update of `main`.
5. Verify local `main` equals the validated effort tip and the newly integrated range contains no merge commits. Record the resulting commit IDs and `main` tip, then release the lock and integration turn. A failed fast-forward requires another base read, rebase, and verification; it is not permission to overwrite concurrent work. Proceed to section 5.9 only after successful integration, or report the preserved worktree and blocker.

#### Pull-request delivery (default)

1. Commit all validated changes with an imperative, non-Conventional-Commit message that follows repository conventions.
2. Fetch `origin/main` again and rebase the issue branch onto it.
3. If the rebase changes the resulting tree, rerun applicable verification steps from the latest report. Have the xhigh-reasoning verification subagent revise the report first if the rebase changes verification requirements.
4. Treat non-trivial rebase conflicts as debugging: an xhigh-reasoning debugging subagent diagnoses the correct resolution and a medium-reasoning corrective-edit subagent applies it, both using `<chosen-model>`.
5. Push the branch. After a post-push rebase, use only `--force-with-lease`, never an unconditional force push.
6. Create a pull request only after implementation and local validation are complete.
7. The pull request body must summarize the change, list validation, and include `Closes #<number>` for every issue delivered by the effort.

One grouped effort produces exactly one pull request. Do not place unrelated issues in its closing keywords.

### 5.8 Make checks green and rebase-merge

Default mode only. Reduced modes skip this entire section and perform no GitHub writes, including PR creation, merges, comments, or issue closure.

1. Monitor every required pull-request check until it reaches a terminal state.
2. Never merge while a required check is pending, skipped unexpectedly, cancelled, or failing.
3. Collect full output from failed checks and follow the same grouped diagnosis, tooling-recovery allowance, and cumulative substantive cycle limit from section 5.6. Delegate diagnosis with `model=<chosen-model>`, `reasoning_effort=xhigh` and the resulting edits with `model=<chosen-model>`, `reasoning_effort=medium`; then validate, commit, rebase if needed, and push again.
4. Obtain the root's exclusive integration turn and repository-wide integration lock before the final fetch, rebase, validation, and merge sequence; retain them until the merge is confirmed or this attempt is abandoned. Confirm the branch remains based on the latest `origin/main`. If it advanced, rebase, rerun applicable verification, push with `--force-with-lease`, and wait for required checks on the updated head. Recheck the base immediately before merging; if external work advances `main`, repeat this sequence rather than merging stale validation.
5. Merge only with GitHub's rebase-merge method. Do not squash, create a merge commit, bypass protections, or use administrator override.
6. Verify the pull request state is merged, record the merge commit, and confirm the newly integrated `main` history is linear (no merge commits). Fetch and safely fast-forward local `main` to the merged remote tip with `--ff-only`; never overwrite concurrent local commits. Release the lock and integration turn so the next coordinator can integrate against the updated base.
7. Verify every delivered issue is closed. If a correct `Closes` keyword did not close an issue, close it with a comment linking the merged pull request.

### 5.9 Report the effort

The coordinator reports after each effort reaches its delivery boundary, or when it is skipped or blocked:

- delivery mode and outcome: uncommitted, fast-forwarded into local `main`, merged through a PR, skipped, or blocked;
- issue numbers and titles;
- whether issues were grouped and the triage justification;
- resolved model family, selection source (override, labels, or default), evidence and initial fallback reasons, plus research, triage, debugging, planning, implementation, and verification delegates used, with actual models, reasoning efforts, and any within-family substitutions;
- branch and worktree path;
- changed behavior or documentation;
- the latest verification report, including refinements, and results of automated checks, manual reviews, and required CI checks;
- substantive correction cycles used out of five, with tooling failures and their single grouped recovery round reported separately and excluded from that count;
- integrated commit IDs and resulting local `main` tip for `--no-pr`, or pull request URL and merge commit in default mode; with `--no-commit`, identify the accumulated changeset in the dedicated run worktree;
- confirmed final issue states;
- any omitted delivery steps or PR-only checks, unfinished changes, deferred or newly discovered work.

The root may start parallel-safe efforts without waiting for peer reports in default and `--no-pr` modes. Before starting dependent or sequential efforts, wait for the necessary delivery reports. Serialize all updates to local `main` across runs. With `--no-commit`, continue in the dedicated run worktree with earlier work intact unless an attempted effort failed and stopped the run.

## 6. Root agent: finish the run

After all executable efforts are reported, the root agent returns one final ordered report containing:

- the initial inventory snapshot and triage order;
- delivery mode, invoking worktree, and starting branch/commit;
- every grouping decision and justification;
- outcomes at the selected delivery boundary: accumulated uncommitted changes in the dedicated run worktree, local commits per effort fast-forwarded into `main`, or merged efforts with pull requests and merge commits;
- skipped issues and why;
- blocked issues, their unresolved dependencies, and evidence;
- issues opened after the snapshot, without processing them;
- validation results, omitted delivery steps, and unfinished changes;
- which efforts ran in parallel, the triage evidence establishing their independence, and which ran sequentially;
- confirmation of dedicated worktree isolation in every mode; serialized integration across concurrent agents and runs, with rebase-merged PRs or local `--ff-only` integration and no new merge commits on `main`; no remote writes in reduced modes and no `main` update with `--no-commit`.

Distinguish completion of the requested local delivery from resolution of a GitHub issue: uncommitted and locally fast-forwarded outcomes leave GitHub issues open. Never claim a blocked, unchecked, or unintegrated `--no-pr` effort succeeded. Do not delete worktrees or branches unless the user or repository workflow explicitly requests cleanup.
