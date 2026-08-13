---
description: Perform a non-destructive cross-artifact consistency and quality analysis across spec.md, plan.md, and tasks.md after task generation.
mode: advanced
tools: [execute, read, edit, 'git/*', 'jira/*', 'github_local_mcp/*']
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty). Run in "Advanced" mode.

## Goal

Identify inconsistencies, duplications, ambiguities, and underspecified items across the three core artifacts (`spec.md`, `plan.md`, `tasks.md`) before implementation. This command MUST run only after `/speckit.tasks` has successfully produced a complete `tasks.md`.

## Operating Constraints

**STRICTLY READ-ONLY**: Do **not** modify any files. Output a structured analysis report. Offer an optional remediation plan (user must explicitly approve before any follow-up editing commands would be invoked manually).

**Constitution Authority**: The runtime effective constitution (`.specify/runtime/effective-constitution.md`) is **non-negotiable** within this analysis scope. Constitution conflicts are automatically CRITICAL and require adjustment of the spec, plan, or tasks—not dilution, reinterpretation, or silent ignoring of the principle. If a principle itself needs to change, that must occur in a separate, explicit constitution update outside `/speckit.analyze`.

## Execution Steps

### 0. PDLC Entry Gate (STORY_ID only)

> **Activation**: Only when `$ARGUMENTS` contains `STORY_ID=<value>`. Skip if no STORY_ID.

Before running any analysis, verify the plan PR is approved and merged.

> **Test bypass**: If `SKIP_APPROVAL_GATES=true` is in `$ARGUMENTS`, skip gate checks, log `TEST_BYPASS active: Phase 7A plan gate skipped`, and proceed to Step 1.

1. Read `settings.yaml` and resolve:
   - `plan_approver_role` = `pdlc.approvals.plan.approver_role` (default: `fde`)
   - `plan_require_merge` = `pdlc.approvals.plan.require_merge` (default: `true`)
   - GitHub team/users = `pdlc.roles.<plan_approver_role>.github_team` / `.github_users`

2. Check `Key Data > Plan PR` in `specs/<STORY_ID>/workflow-state.md`.
   - If absent, block: "No plan PR found. Run `/speckit.plan STORY_ID=<STORY_ID>` to generate the plan and raise the plan PR first."

3. Use GitHub MCP tools to find the plan PR from head `<STORY_ID>` to `main` (open or merged).

4. Fetch all reviews:
   - If any reviewer submitted `CHANGES_REQUESTED`: surface reviewer login and comment. Block with: "Plan PR has changes requested. Resolve via `/speckit.plan`, then re-run `/speckit.analyze`."
   - Otherwise verify: at least one approval from `plan_approver_role` exists; if `plan_require_merge: true`, PR must be `MERGED`.

5. If not met: display PR review status and block. Do not proceed to analysis.

**State Update — Phase 7A Entry Gates passed:**
In `specs/<STORY_ID>/workflow-state.md`, mark `[x] Phase 7A: Analysis Entry Gates` under Completed Phases.

### 1. Initialize Analysis Context

Run `.specify/scripts/powershell/check-prerequisites.ps1 -Json -RequireTasks -IncludeTasks` once from repo root and parse JSON for FEATURE_DIR and AVAILABLE_DOCS. Derive absolute paths:

- SPEC = FEATURE_DIR/spec.md
- PLAN = FEATURE_DIR/plan.md
- TASKS = FEATURE_DIR/tasks.md

Abort with an error message if any required file is missing (instruct the user to run missing prerequisite command).
For single quotes in args like "I'm Groot", use escape syntax: e.g 'I'\''m Groot' (or double-quote if possible: "I'm Groot").

### 2. Load Artifacts (Progressive Disclosure)

Load only the minimal necessary context from each artifact:

**From spec.md:**

- Overview/Context
- Functional Requirements
- Non-Functional Requirements
- User Stories
- Edge Cases (if present)

**From plan.md:**

- Architecture/stack choices
- Data Model references
- Phases
- Technical constraints

**From tasks.md:**

- Task IDs
- Descriptions
- Phase grouping
- Parallel markers [P]
- Referenced file paths

**From constitution:**

- Load `.specify/runtime/effective-constitution.md` for principle validation
- If the runtime effective constitution file is missing, stop and instruct the user to run `/constitution.resolve` first

### 3. Build Semantic Models

Create internal representations (do not include raw artifacts in output):

- **Requirements inventory**: Each functional + non-functional requirement with a stable key (derive slug based on imperative phrase; e.g., "User can upload file" → `user-can-upload-file`)
- **User story/action inventory**: Discrete user actions with acceptance criteria
- **Task coverage mapping**: Map each task to one or more requirements or stories (inference by keyword / explicit reference patterns like IDs or key phrases)
- **Constitution rule set**: Extract principle names and MUST/SHOULD normative statements

### 4. Detection Passes (Token-Efficient Analysis)

Focus on high-signal findings. Limit to 50 findings total; aggregate remainder in overflow summary.

#### A. Duplication Detection

- Identify near-duplicate requirements
- Mark lower-quality phrasing for consolidation

#### B. Ambiguity Detection

- Flag vague adjectives (fast, scalable, secure, intuitive, robust) lacking measurable criteria
- Flag unresolved placeholders (TODO, TKTK, ???, `<placeholder>`, etc.)

#### C. Underspecification

- Requirements with verbs but missing object or measurable outcome
- User stories missing acceptance criteria alignment
- Tasks referencing files or components not defined in spec/plan

#### D. Constitution Alignment

- Any requirement or plan element conflicting with a MUST principle
- Missing mandated sections or quality gates from constitution

#### E. Coverage Gaps

- Requirements with zero associated tasks
- Tasks with no mapped requirement/story
- Non-functional requirements not reflected in tasks (e.g., performance, security)

#### F. Inconsistency

- Terminology drift (same concept named differently across files)
- Data entities referenced in plan but absent in spec (or vice versa)
- Task ordering contradictions (e.g., integration tasks before foundational setup tasks without dependency note)
- Conflicting requirements (e.g., one requires Next.js while other specifies Vue)

### 5. Severity Assignment

Use this heuristic to prioritize findings:

- **CRITICAL**: Violates constitution MUST, missing core spec artifact, or requirement with zero coverage that blocks baseline functionality
- **HIGH**: Duplicate or conflicting requirement, ambiguous security/performance attribute, untestable acceptance criterion
- **MEDIUM**: Terminology drift, missing non-functional task coverage, underspecified edge case
- **LOW**: Style/wording improvements, minor redundancy not affecting execution order

### 6. Produce Compact Analysis Report

Output a Markdown report (no file writes) with the following structure:

## Specification Analysis Report

| ID | Category | Severity | Location(s) | Summary | Recommendation |
|----|----------|----------|-------------|---------|----------------|
| A1 | Duplication | HIGH | spec.md:L120-134 | Two similar requirements ... | Merge phrasing; keep clearer version |

(Add one row per finding; generate stable IDs prefixed by category initial.)

**Coverage Summary Table:**

| Requirement Key | Has Task? | Task IDs | Notes |
|-----------------|-----------|----------|-------|

**Constitution Alignment Issues:** (if any)

**Unmapped Tasks:** (if any)

**Metrics:**

- Total Requirements
- Total Tasks
- Coverage % (requirements with >=1 task)
- Ambiguity Count
- Duplication Count
- Critical Issues Count

### 7. Provide Next Actions

At end of report, output a concise Next Actions block:

- If CRITICAL issues exist: Recommend resolving before `/speckit.implement`
- If only LOW/MEDIUM: User may proceed, but provide improvement suggestions
- Provide explicit command suggestions: e.g., "Run /speckit.specify with refinement", "Run /speckit.plan to adjust architecture", "Manually edit tasks.md to add coverage for 'performance-metrics'"

### 8. Offer Remediation

Ask the user: "Would you like me to suggest concrete remediation edits for the top N issues?" (Do NOT apply them automatically.)

### 9. Check Extension Hooks (`after_analyze`)

Check if `.specify/extensions.yml` exists and contains a `hooks.after_analyze` list.
- If the file does not exist or the list is missing: skip silently.
- For each entry in `hooks.after_analyze`:
  - If `enabled: false`, skip it.
  - If `optional: true`:
    ```
    ℹ️ Optional after_analyze hook available: <command>
    Description: <description>
    To run it manually: /<command> ARTIFACT=tasks STORY_ID=<STORY_ID if present>
    ```
  - If `optional: false` (mandatory):
    - Output `EXECUTE_COMMAND: <command> ARTIFACT=tasks STORY_ID=<STORY_ID if present>`
    - Wait for the hook result. If `HOOK_RESULT: FAIL`, surface the `REMEDIATION` field and halt until resolved.
- If no hooks are registered or `.specify/extensions.yml` does not exist, skip silently.

## Operating Principles

### Context Efficiency

- **Minimal high-signal tokens**: Focus on actionable findings, not exhaustive documentation
- **Progressive disclosure**: Load artifacts incrementally; don't dump all content into analysis
- **Token-efficient output**: Limit findings table to 50 rows; summarize overflow
- **Deterministic results**: Rerunning without changes should produce consistent IDs and counts

### Analysis Guidelines

- **NEVER modify files** (this is read-only analysis)
- **NEVER hallucinate missing sections** (if absent, report them accurately)
- **Prioritize constitution violations** (these are always CRITICAL)
- **Use examples over exhaustive rules** (cite specific instances, not generic patterns)
- **Report zero issues gracefully** (emit success report with coverage statistics)

## Context

$ARGUMENTS

---

## PDLC Orchestration — Post-Analyze

> **Activation**: This section executes only when `$ARGUMENTS` contains a `STORY_ID=<value>` token. If no `STORY_ID` is present, skip this section.

After the analysis report has been produced and any CRITICAL/MAJOR findings addressed (Execution Steps 1–8 complete), execute these PDLC governance steps.

**State Update — Phase 7B complete:**
In `specs/<STORY_ID>/workflow-state.md`, set `CURRENT_STAGE` to `PHASE_7C_PENDING` and mark `[x] Phase 7B: Analyze`.

### Phase 7C/7D — Tasks PR Raised and Approval Gate

Commit `tasks.md` and analysis-resolved artifacts to the branch, then require the configured tasks approver to review before proceeding to CHECKPOINT 3.

**Test bypass**: If `SKIP_APPROVAL_GATES=true` is in `$ARGUMENTS`, still commit and push `tasks.md` and create/reuse the design PR, then skip reviewer wait. Present warning: `TEST_BYPASS active: Phase 7C/7D approval gate skipped`. Mark `[x] Phase 7C: Tasks PR Raised` and `[x] Phase 7D: Tasks PR Approved` in `workflow-state.md`, set `Key Data > Tasks Approval` to `TEST_BYPASS`, set `CURRENT_STAGE` to `PHASE_7E_PENDING`, and continue to Phase 7E.

1. Read `settings.yaml` and resolve `tasks_approver_role`, `tasks_require_merge`, and GitHub team/users for that role.

2. Commit tasks artifact to the `<STORY_ID>` branch:
   - Stage `specs/<STORY_ID>/tasks.md` and any other task-related or analysis-resolved files under `specs/<STORY_ID>/`.
   - Hard rule: only stage files within `specs/<STORY_ID>/`. Do not stage files outside `specs/`.
   - Commit message: `<STORY_ID>: add tasks for review`
   - Push to `origin/<STORY_ID>`.

3. Identify the open design review PR:
   - Use GitHub MCP tools to find the open PR from head `<STORY_ID>` to `main` (raised in Phase 4B).
   - If it was merged and closed, raise a new PR with title: `<STORY_ID>: <story title> — tasks`
   - Record the PR URL under `Key Data > Tasks PR` in `workflow-state.md` (may be the same URL as Plan PR).

4. Request review from the resolved `tasks_approver_role` team/users on the PR.

**State Update — Phase 7C: Tasks PR Raised:**
In `specs/<STORY_ID>/workflow-state.md`, mark `[x] Phase 7C: Tasks PR Raised` and set `CURRENT_STAGE` to `PHASE_7D_PENDING`.

5. Present to the user:
   - Tasks PR URL and current review status.
   - Required approver: `<tasks_approver_role>` — `<github_team>` / `<github_users>`.

   > "`tasks.md` has been committed and pushed. Awaiting `<tasks_approver_role>` approval on the design PR: `<PR_URL>`
   > Type **yes** once the tasks have been approved, or describe any issues."

6. On user confirmation, verify via GitHub MCP:
   - Fetch all reviews on the PR. If any reviewer submitted `CHANGES_REQUESTED`, surface each comment and block.
   - Otherwise verify: at least one approval from `tasks_approver_role` team/users exists on the PR.
   - If `tasks_require_merge: true`, verify the PR is also merged.
   - If not met: display current PR review status and prompt again. Do not accept chat-only confirmation as a bypass.

**Hard stop**: Do not proceed to Phase 7E until `tasks.md` has the required approval from `<tasks_approver_role>`, unless `SKIP_APPROVAL_GATES=true`.

**State Update — Phase 7D complete:**
In `specs/<STORY_ID>/workflow-state.md`, set `CURRENT_STAGE` to `PHASE_7E_PENDING`, mark `[x] Phase 7D: Tasks PR Approved`, and record the tasks approver identity and approval timestamp (or `TEST_BYPASS`) under `Key Data > Tasks Approval`.

### Phase 7E — Post-Tasks-Approval Jira Story Updates

Run immediately after Phase 7D passes. For every affected repo (all repos except the pdlc repo), update the child Jira story with the approved tasks planned for that repo.

> **Formatting rule**: When calling any Jira MCP tool, always pass body text with actual line breaks — never use `\n` escape sequences.

1. Read `specs/<STORY_ID>/tasks.md` and group all tasks by their target repo.

2. For each repo that has at least one task:
   a. Collect the full list of tasks for that repo, preserving dependency order.
   b. Build a structured Jira comment (substitute all placeholders, use real newlines):

      ```
      Tasks approved for repo `<REPO>` — implementation starting:

      Planned tasks:
      - [ ] <task description 1>
      - [ ] <task description 2>
      ...

      Parent story: <STORY_ID>
      Tasks approved by: <tasks_approver_role> (<approver identity from Key Data>)
      Approval timestamp: <timestamp from Key Data>
      ```

   c. Add the comment to the child Jira story (from `Child Stories` in `workflow-state.md`) using the Jira MCP tool.
   d. Transition the child Jira story to `In Progress` using the Jira MCP tool.

3. If a Jira comment or transition call fails, log `FAILED: <reason>` and continue. Do not block the workflow.

4. Display a summary table:

   | Repo | Child Story | Tasks Listed | Jira Comment Added | Status Transition |
   |------|-------------|-------------|-------------------|------------------|
   | `<repo>` | `<key>` | `<n>` | `YES` / `NO (error)` | `UPDATED` / `FAILED` |

**State Update — Phase 7E complete:**
In `specs/<STORY_ID>/workflow-state.md`, set `CURRENT_STAGE` to `CHECKPOINT_3_PENDING` and mark `[x] Phase 7E: Jira Stories Updated with Tasks`.

### CHECKPOINT 3 — Confirm Analysis and Implementation Readiness

Present:
- Path to `specs/<STORY_ID>/tasks.md` — total task count and breakdown per user story.
- Constitution Check gate status from `plan.md` (all passing?).
- Any CRITICAL or MAJOR analysis findings and how they were addressed.
- Confirmed plan approver and tasks approver from `Key Data` (Phases 4B and 7D).
- Phase 7E Jira update summary (child stories notified with planned tasks?).

Ask the user:
> "Plan has been approved (Phase 4B). Tasks have been analyzed (Phase 7B), approved (Phase 7D), and child Jira stories updated with planned tasks (Phase 7E).
> Type **yes** to proceed to implementation, or describe any changes to make first."

Do not write any implementation code until the user confirms.

**State Update — CHECKPOINT 3 confirmed:**
In `specs/<STORY_ID>/workflow-state.md`, set `CURRENT_STAGE` to `PHASE_8A_PENDING` and mark `[x] CHECKPOINT 3: Ready for Implementation`.

> **Next step**: Run `/speckit.impl-queue STORY_ID=<STORY_ID>` to generate the implementation queue, then `/speckit.implement STORY_ID=<STORY_ID>` to execute it.
