# GitHub issue workflow

The invoking root agent owns scope resolution, preflight, and raw inventory. During preflight it delegates artifact verification discovery to `gpt-6-astra-xhigh`; after inventory it spawns one `gpt-6-astra-xhigh` subagent for all technical triage. Using only that triage report, it spawns one fresh `gpt-6-astra-low` coordinator for each individual issue or Astra-approved grouped effort. An effort coordinator owns only that one effort.

## Non-negotiable role boundaries

An effort coordinator is a control-plane agent. It may read instructions, refresh its effort's raw GitHub and Git data, spawn and wait for subagents, run prescribed validation and Git/GitHub commands, and report that effort's outcome. Workspace creation, commits, pushes, PRs, and merges are permitted only as specified by the delivery mode in [Invocation modes](../SKILL.md#invocation-modes). Pass that mode to every descendant; no delegate may advance beyond its delivery boundary.

The coordinator must not:

- classify or prioritize issues;
- infer blockers, dependencies, shared root causes, or grouping opportunities;
- diagnose an issue, failed baseline, test failure, CI failure, merge conflict, or unexpected behavior;
- choose an algorithm, protocol, library, architecture, implementation approach, or fix;
- plan or edit implementation files.

Delegate every triage or debugging judgment to `gpt-6-astra-xhigh` (`model=<selected-model>`, `reasoning_effort=xhigh`). Delegate every implementation or corrective edit to `gpt-6-astra-low` (`model=<selected-model>`, `reasoning_effort=low`). The coordinator relays evidence and executes the resulting decision-complete reports.

Determining or revising verification steps and performing technical manual reviews are also Astra-xhigh work. The root and coordinators relay the verification report and run prescribed checks; they must not independently choose or waive verification requirements.

Apply [Subagent model selection](../SKILL.md#subagent-model-selection) to every spawn and propagate it recursively. All Astra role names in this workflow mean the user-selected model or model family, when provided, with the stated reasoning effort. Without a user override, use the default fallback list. Do not substitute another family for an explicit user choice; report every fallback or blocker.

## Execution mode inheritance and thread exhaustion

When the invoking root explicitly requests **FAST MODE subagents**, fast mode is mandatory for every subagent spawned anywhere in the run. The root must apply it to preflight, triage, and effort coordinators; each coordinator and descendant must state and propagate it in every child task. The model selection policy and exact reasoning-effort requirements still apply. Never spawn a normal-mode fallback. If no model in the fallback list can honor fast mode for a required delegate, stop before that delegate's work and report the run as blocked.

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

Interpret `$address-github-issues [issue-number-or-url] [--no-pr] [--no-commit]` before accessing GitHub:

- Parse flags independently of the optional issue target, in either order. `--no-commit` implies `--no-pr` and takes precedence when both are present. Reject unknown flags.
- No issue argument selects all issues that are open when the inventory is taken, even when flags are supplied. This is a finite snapshot; issues opened later are not added to the active run.
- A decimal number, `#number`, or full GitHub issue URL selects exactly one issue.
- For a URL, verify that its owner and repository match the current checkout's `origin`. Stop on a mismatch.
- A single selected issue may read referenced issues for blocker context, but it must not implement, close, group with, or otherwise expand into them.
- Reject ambiguous or multiple explicit issue arguments instead of guessing.

Default mode delivers through a fresh branch/worktree and a merged PR per effort. Both reduced modes use the invoking worktree throughout the run, perform only read operations against GitHub, and leave issues open. `--no-pr` creates one local commit per validated effort on the current branch, without pushing. `--no-commit` leaves all efforts in one accumulated changeset without commits, temporary commits, or stashes. Neither reduced mode creates or switches branches/worktrees, synchronizes with `main`, or rebases existing work.

The root agent schedules efforts according to the Astra-xhigh triage report and the parallel execution rules below. Proven independent efforts must run through separate parallel coordinator flows; otherwise wait for the current coordinator to report before starting the next effort. A pre-approved multi-issue group counts as one effort.

## 2. Root agent: preflight

Before fetching issue details:

1. Resolve the repository root, `origin` URL, and invoking worktree. Record its starting branch, commit, staged/unstaged diffs, and untracked files so pre-existing changes remain distinguishable from the run's work. In default mode, also resolve the worktree holding local `main`.
2. Read the repository's complete `AGENTS.md` and any instructions it routes to for the affected area.
3. Verify GitHub authentication and read access to the repository. Only default mode requires permission to push, create PRs, rebase-merge, and close issues.
4. In default mode, inspect local `main`. If it is dirty, ahead of `origin/main`, diverged, or unavailable for a safe fast-forward, stop and report the exact state. In reduced modes, use the invoking worktree as it stands, including local commits and edits; `--no-pr` requires an existing current branch. Never stash, reset, discard, or overwrite user work.
5. In default mode only, fetch `origin/main`, fast-forward local `main` with `--ff-only`, and verify local `main` equals `origin/main`. Reduced modes do not require an up-to-date local `main` or alter the invoking branch's relationship to its remote.
6. Resolve an available model for the required `xhigh` and `low` role configurations using the subagent model selection policy before starting the issue inventory. Record any fallbacks and propagate the policy to every delegate.
7. Spawn a dedicated `gpt-6-astra-xhigh` preflight subagent (`model=<selected-model>`, `reasoning_effort=xhigh`) with the repository path, delivery mode, recorded starting state, repository instructions, and this workflow to determine the necessary artifact verification steps. Wait for its report before fetching issue details.

The preflight subagent inspects repository instructions, CI configuration, manifests, existing scripts, documentation, and artifact types. Its verification report must identify applicable automated checks or explicit manual review procedures, working directories, prerequisites, execution stage (local or CI), and success criteria, with supporting repository evidence. Distinguish baseline checks from final artifact verification and identify required CI checks. In reduced modes, use the current worktree for the baseline and identify PR-only checks as omitted by the delivery mode, not passed or local prerequisites. Verification commands must respect the selected mode's Git and GitHub write boundaries. Derive verification from the target repository; do not assume a particular language, build system, or test framework.

When verification procedures are undocumented, derive suitable checks from the artifacts and existing tooling, including manual review where appropriate. Missing automated tests alone do not block the run. If meaningful verification cannot be established or required tooling or other prerequisites are unavailable, report the blocker before issue processing; do not invent commands or treat unperformed checks as passing. Carry this report through the existing delegate-report flow.

In default mode, the root repeats the safe fetch and fast-forward before each later effort because the previous rebase merge changes `main`. In reduced modes, pass the evolving current worktree and prior effort reports onward without synchronization or cleanup. Preserve pre-existing edits and staging; if the effort's changes cannot be safely separated from unrelated existing changes, report a blocker rather than including or overwriting them.

## 3. Root agent: fetch the issue inventory

Fetch only after the mode-specific preflight is complete and its verification report is available with no unresolved preflight blockers.

### All-open mode

1. Enumerate every open issue with pagination until no page remains. Do not rely on GitHub CLI's default limit, and exclude pull requests returned by the Issues API.
2. For every issue, collect at least its number, title, body, state, creation and update timestamps, labels, assignees, milestone, URL, comments, linked closing pull requests, and explicit issue relationships available from GitHub.
3. Preserve the raw issue text and relationship data. The root agent must not summarize it before handing it to the Astra-xhigh triage subagent.
4. Record the inventory timestamp and issue-number snapshot. Newly opened issues are listed in the final report as outside this run.

### Single-issue mode

1. Fetch the selected issue and the same complete fields.
2. If it is not open, report its state and stop without making changes.
3. Fetch raw metadata for issues explicitly referenced as blockers or dependencies, but keep those issues out of implementation scope.

## 4. Root agent: mandatory Astra-xhigh triage

Spawn one `gpt-6-astra-xhigh` triage subagent (`model=<selected-model>`, `reasoning_effort=xhigh`) with the raw inventory, delivery mode, preflight verification report, repository instructions, and repository path. In reduced modes, include the invoking worktree and recorded starting state. The root agent must not pre-classify the issues, suggest an ordering, or make any triage decision.

The triage report must be detailed and decision-complete. It must contain:

- each issue's scope and acceptance criteria;
- applicable baseline and final verification steps for each effort, derived from the preflight verification report;
- explicit blockers and dependencies cited by issue data;
- technical prerequisites inferred from the codebase, protocols, migrations, or rollout order, with evidence;
- likely shared root causes and overlapping implementation surfaces;
- candidate multi-issue groups and an explicit justification or rejection for each plausible group;
- whether external research is beneficial for each effort and why;
- a dependency graph, blocked reason for every blocked node, and the ordered list of efforts;
- the oldest creation timestamp and member issue numbers for every grouped effort;
- explicit parallel-safe sets of dependency-ready efforts, with evidence that they are completely unrelated and cannot interfere through implementation, contracts, migrations, generated artifacts, validation resources, or delivery; explain why remaining efforts require sequential execution.

Triage and debugging are Astra-xhigh work. The root agent may ask the triage subagent to clarify an incomplete report, but it must not fill gaps itself.

### Dependency ordering

Use an edge from prerequisite to dependent. Contract approved multi-issue groups before ordering. Then:

1. Put efforts with no unresolved prerequisites in dependency level 0.
2. Put each remaining acyclic effort in level `1 + max(level of each prerequisite)`.
3. Put efforts blocked by an unresolved external dependency after all executable levels.
4. Put dependency cycles last and report the cycle; never invent an order that pretends the cycle is resolved.
5. Within the same level and blocker class, sort by oldest creation timestamp first, then by lowest issue number.

This produces unblocked-to-most-blocked ordering with oldest-to-newest ordering as the deterministic tie-breaker.

In reduced modes, Astra-xhigh may mark a dependency ready once its prerequisite effort is successfully implemented and locally validated in the shared current worktree, even if the prerequisite issue remains open. Use prior effort reports and current artifacts as evidence; issue state alone does not settle local readiness. External blockers or prerequisites that genuinely require a merge or deployment remain blockers. Failed or unfinished effort changes never satisfy a dependency.

### Grouping rules

Group issues only when the Astra-xhigh triage report establishes all of the following:

- one cohesive implementation or shared root-cause fix satisfies every member;
- one cohesive effort is clearer than separate delivery (one branch and PR in default mode, one commit with `--no-pr`);
- members have compatible acceptance criteria, rollout, and validation;
- no member requires another member to merge first or expose an intermediate result;
- the combined change satisfies every member without partial delivery, with issue closure performed only in default mode;
- grouping does not hide unrelated refactoring or broaden scope.

Shared labels, the same component, nearby files, or potential merge conflicts are not sufficient reasons to group. When uncertain, keep issues separate. In single-issue mode, grouping is forbidden.

### Parallel execution rules

In default mode, when Astra-xhigh triage proves that dependency-ready efforts are completely unrelated, do not interfere in any way, and are completely safe to implement concurrently, the root **must** spawn their fresh coordinators in parallel. Each coordinator runs the full flow below, including implementation, verification, PR creation, and merge. Different issue numbers, disjoint files, or the absence of dependency edges alone do not prove independence. If evidence is missing, obtain triage clarification; do not assume safety.

Use a separate branch and worktree per effort and isolate any mutable validation resources. The root serializes operations on shared local `main` and grants only one coordinator at a time permission to perform the final integration sequence in section 5.8. Other coordinators may continue independent work while awaiting their integration turn. Parallel implementation never permits non-linear `main` history: all created PRs must be rebase-merged, with no squash merges or merge commits.

Use the triage ordering to launch parallel-safe efforts deterministically. An effort with prerequisites waits for their successful delivery. If new evidence invalidates independence, pause the affected flows and obtain revised triage before continuing them. Reduced modes remain sequential because they share the invoking worktree, index, and accumulated changes; do not create extra worktrees or change delivery mode to enable parallelism.

## 5. Root agent: process each effort

For each effort selected, grouped, and ordered by the Astra-xhigh triage report, the root agent spawns a **new** `gpt-6-astra-low` coordinator (`model=<selected-model>`, `reasoning_effort=low`) with the repository path, delivery mode, raw inventory, triage report, latest verification report, prior effort reports, repository instructions, this workflow, and the effort definition. In reduced modes, also pass the invoking worktree, recorded starting state, and accumulated local progress. The fresh coordinator follows the applicable steps below and returns an effort report at the selected delivery boundary. The root agent does not reuse that coordinator for another effort and does not make technical decisions between efforts.

The inventory and combined triage are the sole eligibility decision for the run; coordinators do not refresh issue or dependency state before starting. This deliberately accepts a TOCTOU risk: an issue may be closed, edited, or newly blocked after triage, so work can begin from stale GitHub state. Final validation, current-`main` rebasing, and issue-state reporting still apply, but they do not eliminate that risk.

### 5.1 Select the workspace and run the baseline

In reduced modes, keep using the invoking worktree and current branch. Skip steps 1–4 below. Record the state before this effort, including prior efforts and pre-existing edits, so its own changes can be reviewed and, with `--no-pr`, committed separately. Never reset or clean the worktree between efforts.

In default mode:

1. Create a fresh branch from the verified current `main`; never reuse an old issue branch.
2. Use `issue-<number>-<short-slug>` for one issue and `issues-<lowest-number>-<next-number>-<short-slug>` for a group.
3. Create a dedicated Git worktree for that branch. Prefer a repository-native worktree mechanism when available; otherwise use an existing ignored `.worktrees/` directory or a safe external sibling directory.
4. Verify a project-local worktree directory is ignored before using it. Do not add worktree contents to version control.

In every mode, run applicable baseline checks from the verification and triage reports in the selected worktree before implementation, including every repository-required baseline check. Reduced modes check the evolving current state, including earlier validated changes, without requiring a clean worktree. Delegate any prescribed technical manual review to `gpt-6-astra-xhigh`.

If baseline validation fails, capture exact commands and complete output and delegate diagnosis to `gpt-6-astra-xhigh`. Do not let the coordinator diagnose or waive a red baseline. Proceed only if Astra-xhigh proves the failure is unrelated and the governing repository instructions permit proceeding; otherwise report the effort blocked.

### 5.2 Research external context

When the triage report marks research beneficial, spawn a dedicated `gpt-6-astra-xhigh` research subagent before planning. Give it the issue data, triage report, repository instructions, and precise research questions.

Research may cover existing solutions and libraries, algorithms, protocols, standards, compatibility constraints, security guidance, performance or quality metrics, and baseline measurements. Require a detailed report with primary sources, conclusions, alternatives, risks, and concrete implications for the plan. The coordinator passes the report onward without replacing its conclusions.

### 5.3 Diagnose the issue

For bugs, regressions, failures, performance problems, or unclear behavior, spawn a `gpt-6-astra-xhigh` debugging subagent before planning. Require it to inspect the real code path, reproduce when feasible, trace callers and data flow, compare working patterns, and provide an evidence-backed root-cause report.

For a pure feature or documentation issue, the Astra-xhigh triage report may explicitly state that separate debugging is unnecessary. The coordinator cannot make that determination.

No fix may be planned from symptoms alone. If root cause remains unknown, report the effort blocked rather than asking Astra-low to guess.

### 5.4 Produce the implementation plan

Spawn a fresh `gpt-6-astra-xhigh` planning subagent. Give it:

- the complete current issue or grouped-issue data;
- the triage report;
- the latest verification report;
- every research and debugging report;
- current repository instructions and architecture documents;
- the delivery mode, selected worktree, and baseline result; in reduced modes, also the state before the effort and prior effort reports.

Require a decision-complete plan covering scope, code or documentation changes, interfaces, edge cases, migration or compatibility needs, tests, acceptance criteria, and validation. The planner refines the verification report for the effort's artifacts and acceptance criteria, adding effort-specific checks and updating procedures when requirements change. The plan must identify how one grouped change satisfies each member issue separately. The coordinator may request clarification but must not invent missing technical decisions.

### 5.5 Implement in the worktree

Spawn a `gpt-6-astra-low` implementation subagent (`model=<selected-model>`, `reasoning_effort=low`) in the selected worktree. Give it the delivery mode, approved plan, and all supporting reports. Require it to:

- read and obey repository and relevant skill instructions;
- edit only within the approved scope;
- in reduced modes, preserve earlier effort changes and pre-existing user edits in the current worktree;
- use test-driven development for behavior changes;
- avoid unrelated refactors or speculative abstractions;
- run focused checks while implementing;
- return the changed paths, checks run, and any deviation or blocker.

The coordinator must not edit implementation files. If implementation reveals a missing technical decision, return to Astra-xhigh planning instead of letting Astra-low or the coordinator guess.

### 5.6 Review and validate

1. Collect the diff and implementation report. In reduced modes, distinguish this effort's delta from prior efforts and pre-existing changes, and assess regressions against the accumulated current state.
2. Give them and the latest verification report to a `gpt-6-astra-xhigh` verification subagent to verify the final implementation against each issue's acceptance criteria, grouped-issue completeness, regressions, and missing tests, and perform any prescribed technical manual reviews.
3. Delegate any required edit to `gpt-6-astra-low`.
4. Execute the applicable local verification steps from the latest report and implementation plan, including all focused tests and repository-required local checks. Record results against the stated success criteria; unavailable or unperformed required local checks remain blockers. In default mode, run CI-only checks through section 5.8; their results gate merging. In reduced modes, report PR-only checks as omitted by the mode, never as passed.
5. Confirm only intended changes were introduced by this effort, with earlier work and pre-existing edits preserved, and run `git diff --check`.

Any unexpected result goes first to a `gpt-6-astra-xhigh` debugging subagent with raw evidence. Only after diagnosis may a `gpt-6-astra-low` subagent implement the prescribed fix. The coordinator never diagnoses the failure itself.

Allow at most five diagnose-fix-verify cycles for one effort across local validation and CI. After the fifth unsuccessful cycle, stop that effort and report the accumulated evidence. Do not stack speculative fixes or bypass checks.

In either reduced mode, an attempted effort that cannot complete after the permitted remediation stops further implementation in the shared worktree. Preserve earlier commits or uncommitted work and the unfinished changes; report previously validated efforts separately from the current, possibly failing state. Do not roll back the effort or continue implementing other issues. Issues skipped or blocked during triage without edits do not prevent other eligible efforts from running.

### 5.7 Deliver according to the selected mode

- `--no-commit`: after successful local validation, leave every effort's changes in the same accumulated current changeset. Create no commits, temporary commits, or stashes. Proceed directly to section 5.9.
- `--no-pr`: after successful local validation, create one commit for this issue or approved group on the existing current branch, with an imperative, non-Conventional-Commit message following repository conventions. Commit only this effort's changes, preserving unrelated staged and unstaged edits. Do not blindly commit the existing index; if overlapping changes cannot be safely separated, report a blocker. Do not push, rebase, or create a PR. Proceed directly to section 5.9.

The remaining steps in this section apply only to default mode:

1. Commit all validated changes with an imperative, non-Conventional-Commit message that follows repository conventions.
2. Fetch `origin/main` again and rebase the issue branch onto it.
3. If the rebase changes the resulting tree, rerun applicable verification steps from the latest report. Have Astra-xhigh revise the report first if the rebase changes verification requirements.
4. Treat non-trivial rebase conflicts as debugging: Astra-xhigh diagnoses the correct resolution and Astra-low applies it.
5. Push the branch. After a post-push rebase, use only `--force-with-lease`, never an unconditional force push.
6. Create a pull request only after implementation and local validation are complete.
7. The pull request body must summarize the change, list validation, and include `Closes #<number>` for every issue delivered by the effort.

One grouped effort produces exactly one pull request. Do not place unrelated issues in its closing keywords.

### 5.8 Make checks green and rebase-merge

Default mode only. Reduced modes skip this entire section and perform no GitHub writes, including PR creation, merges, comments, or issue closure.

1. Monitor every required pull-request check until it reaches a terminal state.
2. Never merge while a required check is pending, skipped unexpectedly, cancelled, or failing.
3. For any failure, collect the full check output and delegate diagnosis to `gpt-6-astra-xhigh`; delegate the resulting code change to `gpt-6-astra-low`; then validate, commit, rebase if needed, and push again.
4. Obtain the root's exclusive integration turn before the final fetch, rebase, validation, and merge sequence; retain it until the merge is confirmed or this attempt is abandoned. Confirm the branch remains based on the latest `origin/main`. If it advanced, rebase, rerun applicable verification, push with `--force-with-lease`, and wait for required checks on the updated head. Recheck the base immediately before merging; if external work advances `main`, repeat this sequence rather than merging stale validation.
5. Merge only with GitHub's rebase-merge method. Do not squash, create a merge commit, bypass protections, or use administrator override.
6. Verify the pull request state is merged, record the merge commit, and confirm the newly integrated `main` history is linear (no merge commits). Release the integration turn so the next coordinator can integrate against the updated base.
7. Verify every delivered issue is closed. If a correct `Closes` keyword did not close an issue, close it with a comment linking the merged pull request.

### 5.9 Report the effort

The coordinator reports after each effort reaches its delivery boundary, or when it is skipped or blocked:

- delivery mode and outcome: uncommitted, committed locally, merged, skipped, or blocked;
- issue numbers and titles;
- whether issues were grouped and the Astra-xhigh justification;
- research, triage, debugging, planning, implementation, and verification delegates used, with actual models, reasoning efforts, and fallback reasons;
- branch and worktree path;
- changed behavior or documentation;
- the latest verification report, including refinements, and results of automated checks, manual reviews, and required CI checks;
- local commit ID for `--no-pr`, or pull request URL and merge commit in default mode; with `--no-commit`, identify the accumulated changeset in the current worktree;
- confirmed final issue states;
- any omitted delivery steps or PR-only checks, unfinished changes, deferred or newly discovered work.

The root may start parallel-safe efforts without waiting for peer reports. Before starting dependent or sequential efforts, wait for the necessary reports; serialize safe fast-forwards of local `main` in default mode. In reduced modes, continue in the same worktree with earlier work intact unless an attempted effort failed and stopped the shared run.

## 6. Root agent: finish the run

After all executable efforts are reported, the root agent returns one final ordered report containing:

- the initial inventory snapshot and triage order;
- delivery mode, invoking worktree, and starting branch/commit;
- every grouping decision and justification;
- outcomes at the selected delivery boundary: accumulated uncommitted changes, local commits per effort, or merged efforts with pull requests and merge commits;
- skipped issues and why;
- blocked issues, their unresolved dependencies, and evidence;
- issues opened after the snapshot, without processing them;
- validation results, omitted delivery steps, and unfinished changes;
- which efforts ran in parallel, the triage evidence establishing their independence, and which ran sequentially;
- confirmation that integration was serialized, all created PRs were rebase-merged, and resulting `main` history remained linear, or that the reduced mode performed no remote writes.

Distinguish completion of the requested local delivery from resolution of a GitHub issue: uncommitted and locally committed outcomes leave issues open and are not merged resolutions. Never claim a blocked or unchecked effort succeeded. Do not delete worktrees or branches unless the user or repository workflow explicitly requests cleanup.
