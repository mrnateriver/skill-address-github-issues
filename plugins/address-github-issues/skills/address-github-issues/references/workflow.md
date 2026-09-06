# GitHub issue workflow

The invoking root agent owns scope resolution, preflight, and raw inventory. During preflight it delegates artifact verification discovery to `gpt-6-astra-xhigh`; after inventory it spawns one `gpt-6-astra-xhigh` subagent for all technical triage. Using only that triage report, it spawns one fresh `gpt-6-astra-low` coordinator for each individual issue or Astra-approved grouped effort. An effort coordinator owns only that one effort.

## Non-negotiable role boundaries

An effort coordinator is a control-plane agent. It may read instructions, refresh its effort's raw GitHub and Git data, run prescribed Git/GitHub commands, create a worktree, spawn and wait for subagents, run validation commands, publish reviewed commits, monitor checks, merge, and report that effort's outcome.

The coordinator must not:

- classify or prioritize issues;
- infer blockers, dependencies, shared root causes, or grouping opportunities;
- diagnose an issue, failed baseline, test failure, CI failure, merge conflict, or unexpected behavior;
- choose an algorithm, protocol, library, architecture, implementation approach, or fix;
- plan or edit implementation files.

Delegate every triage or debugging judgment to `gpt-6-astra-xhigh` (`model=<selected-model>`, `reasoning_effort=xhigh`). Delegate every implementation or corrective edit to `gpt-6-astra-low` (`model=<selected-model>`, `reasoning_effort=low`). The coordinator relays evidence and executes the resulting decision-complete reports.

Determining or revising verification steps and performing technical manual reviews are also Astra-xhigh work. The root and coordinators relay the verification report and run prescribed checks; they must not independently choose or waive verification requirements.

Apply [Subagent model selection](../SKILL.md#subagent-model-selection) to every spawn and propagate it recursively. All Astra role names in this workflow use the selected available model with the stated reasoning effort. A model being unavailable requires trying the next model in the ordered list; only exhaustion of that list blocks the run for model availability. Report every fallback.

## Execution mode inheritance and thread exhaustion

When the invoking root explicitly requests **FAST MODE subagents**, fast mode is mandatory for every subagent spawned anywhere in the run. The root must apply it to preflight, triage, and effort coordinators; each coordinator and descendant must state and propagate it in every child task. The model selection policy and exact reasoning-effort requirements still apply. Never spawn a normal-mode fallback. If no model in the fallback list can honor fast mode for a required delegate, stop before that delegate's work and report the run as blocked.

If any agent receives a subagent-thread-limit error or otherwise cannot spawn a required delegate because the harness's thread capacity is exhausted, it must stop immediately. It must not diagnose, plan, implement, validate, publish, merge, reuse a previous delegate, or wait for capacity to recover.

Before stopping, that agent must write a persistent Markdown handover under the repository's ignored `.worktrees/address-github-issues-handovers/` directory. Use a UTC timestamp and the agent task name in the filename. The artifact must record:

- repository path, invocation scope, inventory timestamp, and issue snapshot;
- agent role/task path, current effort, workflow stage, and the exact thread-limit error;
- branch, worktree, commit, push, pull-request, check, merge, and issue states;
- completed delegates and the locations or full contents of their reports;
- commands/checks already run, their results, current Git status, and any uncommitted files;
- remaining work and the single next safe action after restart.

Do not include credentials or other secrets. After writing the artifact, report its absolute path to the parent/root and request that the human restart the harness to clean up sessions. The root must cancel the active issue flow and surface that restart request; no agent may resume the run in the exhausted harness.

## 1. Root agent: resolve scope

Interpret the optional skill argument before accessing GitHub:

- No argument selects all issues that are open when the inventory is taken. This is a finite snapshot; issues opened later are not added to the active run.
- A decimal number, `#number`, or full GitHub issue URL selects exactly one issue.
- For a URL, verify that its owner and repository match the current checkout's `origin`. Stop on a mismatch.
- A single selected issue may read referenced issues for blocker context, but it must not implement, close, group with, or otherwise expand into them.
- Reject ambiguous or multiple explicit issue arguments instead of guessing.

The root agent performs efforts strictly sequentially. It must wait for a coordinator to finish and report its current effort before spawning a fresh coordinator for the next. A pre-approved multi-issue group counts as one effort and is still processed serially relative to every other effort.

## 2. Root agent: preflight and fast-forward `main`

Before fetching issue details:

1. Resolve the repository root, `origin` URL, and the worktree holding local `main`.
2. Read the repository's complete `AGENTS.md` and any instructions it routes to for the affected area.
3. Verify GitHub authentication and read access to the repository.
4. Inspect local `main`. If it is dirty, ahead of `origin/main`, diverged, or unavailable for a safe fast-forward, stop and report the exact state. Never stash, reset, discard, or overwrite user work.
5. Fetch `origin/main`, fast-forward local `main` with `--ff-only`, and verify local `main` equals `origin/main`.
6. Resolve an available model for the required `xhigh` and `low` role configurations using the subagent model selection policy before starting the issue inventory. Record any fallbacks and propagate the policy to every delegate.
7. Spawn a dedicated `gpt-6-astra-xhigh` preflight subagent (`model=<selected-model>`, `reasoning_effort=xhigh`) with the repository path, repository instructions, and this workflow to determine the necessary artifact verification steps. Wait for its report before fetching issue details.

The preflight subagent inspects repository instructions, CI configuration, manifests, existing scripts, documentation, and artifact types. Its verification report must identify applicable automated checks or explicit manual review procedures, working directories, prerequisites, execution stage (local or CI), and success criteria, with supporting repository evidence. Distinguish clean-baseline checks from final artifact verification and identify required CI checks. Derive verification from the target repository; do not assume a particular language, build system, or test framework.

When verification procedures are undocumented, derive suitable checks from the artifacts and existing tooling, including manual review where appropriate. Missing automated tests alone do not block the run. If meaningful verification cannot be established or required tooling or other prerequisites are unavailable, report the blocker before issue processing; do not invent commands or treat unperformed checks as passing. Carry this report through the existing delegate-report flow.

The root agent repeats the safe fetch and fast-forward before it spawns each later effort coordinator because the previous effort's rebase merge changes `main`.

## 3. Root agent: fetch the issue inventory

Fetch only after `main` is current and the preflight verification report is available with no unresolved preflight blockers.

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

Spawn one `gpt-6-astra-xhigh` triage subagent (`model=<selected-model>`, `reasoning_effort=xhigh`) with the raw inventory, preflight verification report, repository instructions, and repository path. The root agent must not pre-classify the issues, suggest an ordering, or make any triage decision.

The triage report must be detailed and decision-complete. It must contain:

- each issue's scope and acceptance criteria;
- applicable baseline and final verification steps for each effort, derived from the preflight verification report;
- explicit blockers and dependencies cited by issue data;
- technical prerequisites inferred from the codebase, protocols, migrations, or rollout order, with evidence;
- likely shared root causes and overlapping implementation surfaces;
- candidate multi-issue groups and an explicit justification or rejection for each plausible group;
- whether external research is beneficial for each effort and why;
- a dependency graph, blocked reason for every blocked node, and the ordered list of efforts;
- the oldest creation timestamp and member issue numbers for every grouped effort.

Triage and debugging are Astra-xhigh work. The root agent may ask the triage subagent to clarify an incomplete report, but it must not fill gaps itself.

### Dependency ordering

Use an edge from prerequisite to dependent. Contract approved multi-issue groups before ordering. Then:

1. Put efforts with no unresolved prerequisites in dependency level 0.
2. Put each remaining acyclic effort in level `1 + max(level of each prerequisite)`.
3. Put efforts blocked by an unresolved external dependency after all executable levels.
4. Put dependency cycles last and report the cycle; never invent an order that pretends the cycle is resolved.
5. Within the same level and blocker class, sort by oldest creation timestamp first, then by lowest issue number.

This produces unblocked-to-most-blocked ordering with oldest-to-newest ordering as the deterministic tie-breaker.

### Grouping rules

Group issues only when the Astra-xhigh triage report establishes all of the following:

- one cohesive implementation or shared root-cause fix satisfies every member;
- one branch and pull request are clearer than separate delivery;
- members have compatible acceptance criteria, rollout, and validation;
- no member requires another member to merge first or expose an intermediate result;
- the combined change can close every member without partial delivery;
- grouping does not hide unrelated refactoring or broaden scope.

Shared labels, the same component, nearby files, or potential merge conflicts are not sufficient reasons to group. When uncertain, keep issues separate. In single-issue mode, grouping is forbidden.

## 5. Root agent: process each effort sequentially

For each effort selected, grouped, and ordered by the Astra-xhigh triage report, the root agent spawns a **new** `gpt-6-astra-low` coordinator (`model=<selected-model>`, `reasoning_effort=low`) with the repository path, raw inventory, current raw member-issue data, triage report, latest verification report, repository instructions, this workflow, and the effort definition. The fresh coordinator completes every subsection below and returns an effort report. The root agent does not reuse that coordinator for another effort and does not make technical decisions between efforts.

### 5.1 Refresh eligibility

1. Safe-fast-forward local `main` again.
2. Refresh every member issue and its known dependencies.
3. Skip and report members already closed by other work.
4. If issue data or dependency state changed materially, send the new raw evidence back to `gpt-6-astra-xhigh` for revised triage. The coordinator must not revise the ordering or group itself.
5. Do not start an effort that the latest Astra-xhigh report marks blocked.
6. If repository instructions, verification configuration, or affected artifact types have changed since verification discovery, obtain an updated report from `gpt-6-astra-xhigh` before baseline validation.

### 5.2 Create the branch and worktree

1. Create a fresh branch from the verified current `main`; never reuse an old issue branch.
2. Use `issue-<number>-<short-slug>` for one issue and `issues-<lowest-number>-<next-number>-<short-slug>` for a group.
3. Create a dedicated Git worktree for that branch. Prefer a repository-native worktree mechanism when available; otherwise use an existing ignored `.worktrees/` directory or a safe external sibling directory.
4. Verify a project-local worktree directory is ignored before using it. Do not add worktree contents to version control.
5. Run the applicable clean-baseline checks from the verification and triage reports in the worktree before implementation, including every repository-required baseline check. Delegate any prescribed technical manual review to `gpt-6-astra-xhigh`.

If baseline validation fails, capture exact commands and complete output and delegate diagnosis to `gpt-6-astra-xhigh`. Do not let the coordinator diagnose or waive a red baseline. Proceed only if Astra-xhigh proves the failure is unrelated and the governing repository instructions permit proceeding; otherwise report the effort blocked.

### 5.3 Research external context

When the triage report marks research beneficial, spawn a dedicated `gpt-6-astra-xhigh` research subagent before planning. Give it the issue data, triage report, repository instructions, and precise research questions.

Research may cover existing solutions and libraries, algorithms, protocols, standards, compatibility constraints, security guidance, performance or quality metrics, and baseline measurements. Require a detailed report with primary sources, conclusions, alternatives, risks, and concrete implications for the plan. The coordinator passes the report onward without replacing its conclusions.

### 5.4 Diagnose the issue

For bugs, regressions, failures, performance problems, or unclear behavior, spawn a `gpt-6-astra-xhigh` debugging subagent before planning. Require it to inspect the real code path, reproduce when feasible, trace callers and data flow, compare working patterns, and provide an evidence-backed root-cause report.

For a pure feature or documentation issue, the Astra-xhigh triage report may explicitly state that separate debugging is unnecessary. The coordinator cannot make that determination.

No fix may be planned from symptoms alone. If root cause remains unknown, report the effort blocked rather than asking Astra-low to guess.

### 5.5 Produce the implementation plan

Spawn a fresh `gpt-6-astra-xhigh` planning subagent. Give it:

- the complete current issue or grouped-issue data;
- the triage report;
- the latest verification report;
- every research and debugging report;
- current repository instructions and architecture documents;
- the clean-baseline result.

Require a decision-complete plan covering scope, code or documentation changes, interfaces, edge cases, migration or compatibility needs, tests, acceptance criteria, and validation. The planner refines the verification report for the effort's artifacts and acceptance criteria, adding effort-specific checks and updating procedures when requirements change. The plan must identify how one grouped change satisfies each member issue separately. The coordinator may request clarification but must not invent missing technical decisions.

### 5.6 Implement in the worktree

Spawn a `gpt-6-astra-low` implementation subagent (`model=<selected-model>`, `reasoning_effort=low`) in the effort's worktree. Give it the approved plan and all supporting reports. Require it to:

- read and obey repository and relevant skill instructions;
- edit only within the approved scope;
- use test-driven development for behavior changes;
- avoid unrelated refactors or speculative abstractions;
- run focused checks while implementing;
- return the changed paths, checks run, and any deviation or blocker.

The coordinator must not edit implementation files. If implementation reveals a missing technical decision, return to Astra-xhigh planning instead of letting Astra-low or the coordinator guess.

### 5.7 Review and validate

1. Collect the diff and implementation report.
2. Give them and the latest verification report to a `gpt-6-astra-xhigh` verification subagent to verify the final implementation against each issue's acceptance criteria, grouped-issue completeness, regressions, and missing tests, and perform any prescribed technical manual reviews.
3. Delegate any required edit to `gpt-6-astra-low`.
4. Execute the applicable local verification steps from the latest report and implementation plan, including all focused tests and repository-required local checks. Record results against the stated success criteria; unavailable or unperformed required local checks remain blockers. Run CI-only checks through the pull-request workflow in section 5.9; their results gate merging.
5. Confirm only intended files changed and run `git diff --check`.

Any unexpected result goes first to a `gpt-6-astra-xhigh` debugging subagent with raw evidence. Only after diagnosis may a `gpt-6-astra-low` subagent implement the prescribed fix. The coordinator never diagnoses the failure itself.

Allow at most five diagnose-fix-verify cycles for one effort across local validation and CI. After the fifth unsuccessful cycle, stop that effort and report the accumulated evidence. Do not stack speculative fixes or bypass checks.

### 5.8 Commit, rebase, and create the pull request

1. Commit all validated changes with an imperative, non-Conventional-Commit message that follows repository conventions.
2. Fetch `origin/main` again and rebase the issue branch onto it.
3. If the rebase changes the resulting tree, rerun applicable verification steps from the latest report. Have Astra-xhigh revise the report first if the rebase changes verification requirements.
4. Treat non-trivial rebase conflicts as debugging: Astra-xhigh diagnoses the correct resolution and Astra-low applies it.
5. Push the branch. After a post-push rebase, use only `--force-with-lease`, never an unconditional force push.
6. Create a pull request only after implementation and local validation are complete.
7. The pull request body must summarize the change, list validation, and include `Closes #<number>` for every issue delivered by the effort.

One grouped effort produces exactly one pull request. Do not place unrelated issues in its closing keywords.

### 5.9 Make checks green and rebase-merge

1. Monitor every required pull-request check until it reaches a terminal state.
2. Never merge while a required check is pending, skipped unexpectedly, cancelled, or failing.
3. For any failure, collect the full check output and delegate diagnosis to `gpt-6-astra-xhigh`; delegate the resulting code change to `gpt-6-astra-low`; then validate, commit, rebase if needed, and push again.
4. Confirm the branch remains based on the latest `main` before merge. Rebase and rerun affected checks when required.
5. Merge only with GitHub's rebase-merge method. Do not squash, create a merge commit, bypass protections, or use administrator override.
6. Verify the pull request state is merged and record the merge commit.
7. Verify every delivered issue is closed. If a correct `Closes` keyword did not close an issue, close it with a comment linking the merged pull request.

### 5.10 Report the effort

The coordinator reports after each merged effort:

- issue numbers and titles;
- whether issues were grouped and the Astra-xhigh justification;
- research, triage, debugging, planning, implementation, and verification delegates used, with actual models, reasoning efforts, and fallback reasons;
- branch and worktree path;
- changed behavior or documentation;
- the latest verification report, including refinements, and results of automated checks, manual reviews, and required CI checks;
- pull request URL and merge commit;
- confirmed final issue states;
- any deferred or newly discovered work.

Only after this report may the root agent fast-forward `main` and spawn the next effort coordinator.

## 6. Root agent: finish the run

After all executable efforts are reported, the root agent returns one final ordered report containing:

- the initial inventory snapshot and triage order;
- every grouping decision and justification;
- merged efforts with pull requests and merge commits;
- skipped issues and why;
- blocked issues, their unresolved dependencies, and evidence;
- issues opened after the snapshot, without processing them;
- confirmation that efforts ran sequentially and all merges used rebase merge.

Do not claim completion for a blocked, unmerged, or unchecked issue. Do not delete worktrees or branches unless the user or repository workflow explicitly requests cleanup.
