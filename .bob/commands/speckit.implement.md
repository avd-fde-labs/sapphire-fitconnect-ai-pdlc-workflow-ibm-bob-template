---
description: Execute the implementation plan by processing and executing all tasks defined in tasks.md
mode: advanced
tools: [execute, read, agent, edit, 'git/*', 'context7/*']
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty). Run in "Advanced" mode.

## HARD STOP Rules (enforced throughout this agent)

**Rule 1 — Never exit silently.** You MUST produce at minimum one message explaining what happened before stopping. An empty return is a protocol violation.

**Rule 2 — Never implement directly when invoked by pdlc-workflow.** This agent is the sole implementation executor. You are only allowed to call tools that create or modify code files. You must not ask the caller to implement, and you must not hand work back unexecuted.

**Rule 3 — Prerequisites failure = hard stop with diagnosis, not silent exit.** If Step 1 fails for any reason, output the full diagnosis block below and STOP:

```text
## IMPLEMENT AGENT — HARD STOP: Prerequisites Failed

Reason: <exact error or reason>
Resolution: The caller (pdlc-workflow) must re-invoke this agent and pass STORY_ID=<id>
as the first token in $ARGUMENTS, e.g.:
  STORY_ID=SCRUM-43 Execute all remaining tasks for the active feature in dependency order.

Do NOT implement tasks outside this agent.
```

**Rule 4 — $ARGUMENTS must be parsed for STORY_ID first.** Before running any script, scan `$ARGUMENTS` for a token matching `STORY_ID=<value>`. If found, extract `<value>` as the authoritative story ID and compute `FEATURE_DIR = <repo-root>/specs/<value>`. The script in Step 1 is then used only for supplementary validation and to populate `AVAILABLE_DOCS`. If `FEATURE_DIR` derived from `$ARGUMENTS` is valid (directory exists and contains `tasks.md`), skip the script's branch-detection logic entirely.

**Rule 5 — Detect execution mode from $ARGUMENTS before any other step.** Parse these optional tokens from `$ARGUMENTS` immediately after STORY_ID:
- `REPO=<name>` and `PHASE=<label>` (both required together) → set **MODE = SCOPED**. Execute only the named repo's named phase. See **Scoped Mode** section below.
- Neither → set **MODE = FULL**. Execute all repos and all phases in order. If `impl-queue.md` exists in `FEATURE_DIR`, tick each completed repo/phase entry (`[ ]` → `[x]`) as execution progresses.

## Scoped Mode

Activated when `$ARGUMENTS` contains both `REPO=<name>` and `PHASE=<label>`.

In scoped mode this agent executes **exactly one repo's one phase** and nothing else.

**Entry behaviour:**
1. Resolve `FEATURE_DIR` via Rule 4.
2. Extract `REPO_FILTER = <name>` and `PHASE_FILTER = <label>` from `$ARGUMENTS`.
3. Read `tasks.md` from `FEATURE_DIR`. Do **not** check checklists or run project setup verification (steps 2 and 4 of the Outline are skipped unconditionally in scoped mode).
4. Load `plan.md` for tech stack context. Skip skill auto-detection for specs you do not need — only load skills relevant to `REPO_FILTER`.
5. Filter tasks to: repo == `REPO_FILTER` AND phase header matches `PHASE_FILTER` (exact string match on the phase heading line in `tasks.md`). If no matching `[ ]` tasks are found, output a skip notice and stop:
   ```text
   ## Phase Skipped — <REPO_FILTER> / <PHASE_FILTER>
   No incomplete tasks found for this repo/phase. Already complete or nothing to do.
   ```
6. Read relevant source files for `REPO_FILTER` once.
7. Execute the filtered tasks in order, honouring `[P]` markers within this bucket.
8. Mark each task `[x]` in `tasks.md` immediately on completion.
9. Tick the matching entry in `impl-queue.md` (`[ ]` → `[x]`).
10. **Check if this was the final queue entry** — scan `impl-queue.md` for any remaining `[ ]` entries:
    - If **all entries are now `[x]`**: this is the last item. In `specs/<STORY_ID>/workflow-state.md`:
      - Mark `[x] Phase 8C: Implement` under Completed Phases.
      - Set `CURRENT_STAGE=CHECKPOINT_4_PENDING`.
      - Append to the scoped completion report: `All queue entries complete. Phase 8C marked done.`
    - If **any `[ ]` entries remain**: do nothing to workflow-state. The report should hint at the next invocation (e.g. next unchecked row from `impl-queue.md`).
11. Output the **scoped completion report** and stop:

   ```text
   ## Phase Complete — <REPO_FILTER> / <PHASE_FILTER>
   
   | Task | Status |
   |------|--------|
   | T004 | ✓ done |
   | T005 | ✓ done |
   
   Tasks marked [x] in tasks.md. Queue entry ticked in impl-queue.md.
   [If final queue entry]: All queue entries complete. Phase 8C marked done. Run /speckit.ship STORY_ID=<STORY_ID> to raise PRs.
   [Otherwise]: Next: /speckit.implement STORY_ID=<STORY_ID> REPO=<next-repo> PHASE="<next-phase>"
   
   SCOPED_RESULT: COMPLETE
   ```

   If any task failed:
   ```text
   SCOPED_RESULT: PARTIAL
   Failed tasks: T006 — <reason>
   ```

   If the agent cannot proceed at all:
   ```text
   SCOPED_RESULT: FAILED
   Reason: <exact reason>
   ```

**Do not proceed to the next repo or phase.** The caller (pdlc-workflow) drives iteration.

## Skill Auto-Detection

Before proceeding to Pre-Execution Checks, scan `$ARGUMENTS`, `plan.md`, and `tasks.md` to detect the tech stack and load all relevant personal skills using `read_file`.

After detection and loading, you MUST print an explicit evidence block before any pre-execution checks:

```text
## Skills Loaded

- Detected technologies: <comma-separated list or "none">
- Parent skill routers loaded:
  - <path 1>
  - <path 2>
- Subskills loaded (if any):
  - <path 1>
  - <path 2>
```

If no skills are loaded, still print:

```text
## Skills Loaded

- Detected technologies: none
- Parent skill routers loaded: none
- Subskills loaded: none
```

Skills are located under the .bob/skills folder in the root of the repository.

**Load the parent skill router** for each detected technology:

| Detected technology | Parent skill router to load |
|---|---|
| Java, Spring Boot, Maven, Gradle, JUnit, Mockito | `skills/java/SKILL.md` |
| GraphQL schema, SDL, resolvers, mutations, subscriptions | `skills/graphql/SKILL.md` |
| React, JSX, hooks, components, React Testing Library | `skills/react/SKILL.md` |

**Rules:**
- Load every parent skill router that matches the detected stack — each router contains its own subskill routing instructions and will direct you to load the correct subskill(s).
- If a task spans multiple technologies, load all matching parent routers.
- Apply all loaded skill guidance throughout implementation.
- If no technology match is found, skip this section and proceed normally.

## Pre-Execution Checks

**PDLC Entry Gate — Tasks PR approved and merged (STORY_ID only)**:

> **Activation**: Only when `$ARGUMENTS` contains `STORY_ID=<value>`. Skip if no STORY_ID or if `SKIP_APPROVAL_GATES=true` (log `TEST_BYPASS active: Phase 8A tasks gate skipped`).

1. Read `settings.yaml` and resolve:
   - `tasks_approver_role` = `pdlc.approvals.tasks.approver_role` (default: `fde`)
   - `tasks_require_merge` = `pdlc.approvals.tasks.require_merge` (default: `true`)
   - GitHub team/users = `pdlc.roles.<tasks_approver_role>.github_team` / `.github_users`

2. Check `Key Data > Tasks PR` in `specs/<STORY_ID>/workflow-state.md`.
   - If absent, block: "No tasks PR found. Run `/speckit.analyze STORY_ID=<STORY_ID>` to commit tasks, raise the tasks PR, and obtain approval first."

3. Use GitHub MCP tools to find the tasks PR from head `<STORY_ID>` to `main` (open or merged).

4. Fetch all reviews:
   - If any reviewer submitted `CHANGES_REQUESTED`: surface reviewer login and comment. Block with: "Tasks PR has changes requested. Resolve via `/speckit.analyze`, then re-run `/speckit.implement`."
   - Otherwise verify: at least one approval from `tasks_approver_role` exists; if `tasks_require_merge: true`, PR must be `MERGED`.

5. If not met: display PR review status and block. Do not proceed to implementation.

**State Update — Phase 8A: Implementation Entry Gates passed:**
In `specs/<STORY_ID>/workflow-state.md`, mark `[x] Phase 8A: Implementation Entry Gates` under Completed Phases.

**Check for extension hooks (before implementation)**:
- Check if `.specify/extensions.yml` exists in the project root.
- If it exists, read it and look for entries under the `hooks.before_implement` key
- If the YAML cannot be parsed or is invalid, skip hook checking silently and continue normally
- Filter out hooks where `enabled` is explicitly `false`. Treat hooks without an `enabled` field as enabled by default.
- For each remaining hook, do **not** attempt to interpret or evaluate hook `condition` expressions:
  - If the hook has no `condition` field, or it is null/empty, treat the hook as executable
  - If the hook defines a non-empty `condition`, skip the hook and leave condition evaluation to the HookExecutor implementation
- For each executable hook, output the following based on its `optional` flag:
  - **Optional hook** (`optional: true`):
    ```
    ## Extension Hooks

    **Optional Pre-Hook**: {extension}
    Command: `/{command}`
    Description: {description}

    Prompt: {prompt}
    To execute: `/{command}`
    ```
  - **Mandatory hook** (`optional: false`):
    ```
    ## Extension Hooks

    **Automatic Pre-Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}
    
    Wait for the result of the hook command before proceeding to the Outline.
    ```
- If no hooks are registered or `.specify/extensions.yml` does not exist, skip silently

## Outline

1. **Resolve FEATURE_DIR** — follow this ordered resolution strategy:

   **Step 1a — STORY_ID from $ARGUMENTS (primary, preferred)**:
   Scan `$ARGUMENTS` for a token in the form `STORY_ID=<value>` (e.g., `STORY_ID=SCRUM-43`).
   - If found, extract `<value>` as the story ID.
   - Determine repo root by reading the current working directory or resolving upward from the workspace root.
   - Set `FEATURE_DIR = <repo-root>/specs/<value>` as an absolute path.
   - Validate: `FEATURE_DIR` must exist as a directory and contain `tasks.md`. If not, apply **Hard Stop Rule 3**.
   - If `FEATURE_DIR` is valid, **skip Step 1b** (do not run the script for branch detection).
   - Run `.specify/scripts/powershell/check-prerequisites.ps1 -Json -RequireTasks -IncludeTasks` solely to populate `AVAILABLE_DOCS`; ignore its `FEATURE_DIR` output.

   **Step 1b — Script-based detection (fallback, used only when STORY_ID absent from $ARGUMENTS)**:
   Run `.specify/scripts/powershell/check-prerequisites.ps1 -Json -RequireTasks -IncludeTasks` from repo root and parse `FEATURE_DIR` and `AVAILABLE_DOCS` list. All paths must be absolute. For single quotes in args like "I'm Groot", use escape syntax: e.g `'I'\''m Groot'` (or double-quote if possible: `"I'm Groot"`).
   - If the script exits with a non-zero code **or** outputs any line beginning with `ERROR:`, apply **Hard Stop Rule 3** immediately. Do not proceed.
   - If `FEATURE_DIR` resolves but does not contain `tasks.md`, apply **Hard Stop Rule 3**.

   **Step 1c — Branch Checkout** ⚠️ MANDATORY — execute before any other work:

   > **STOP.** Before reading any source files, loading any skills, or running any tasks, you MUST complete the branch checkout below and output a `## Branch Checkout Summary` table. Do not skip this step. Do not defer it. It runs immediately after FEATURE_DIR is known.
   >
   > The only exception: if STORY_ID was **not** present in `$ARGUMENTS`, skip this step entirely.

   Use MCP git tools for all branch operations. Do **not** use shell commands.

   **Repos in scope** — depends on execution mode:
   - **FULL mode**: orchestrator repo + all sibling repos listed under `Affected Repos` in `workflow-state.md`.
   - **SCOPED mode**: orchestrator repo + only the single repo matching `REPO_FILTER`. Do **not** touch other sibling repos.

   **Orchestrator repo (current workspace)**:
   - Use MCP git to read the current branch name.
   - If already on `<STORY_ID>`: note `already on branch`.
   - If on a different branch: checkout `<STORY_ID>` (create tracking `origin/<STORY_ID>` if local branch doesn't exist; create from HEAD if remote doesn't exist either).

   **Sibling repo(s)**:
   - Read `specs/<STORY_ID>/workflow-state.md > Child Stories` to map each repo name to its child story key (format: `<repo-name>: <child-key>`).
   - For each repo in scope:
     - Locate it at `../<repo-name>/` relative to the workspace root. If not found, warn and skip — do not block.
     - Use MCP git to read its current branch.
     - If already on `<child-key>`: note `already on branch`.
     - If on a different branch: checkout `<child-key>` (create tracking `origin/<child-key>` if needed; create from HEAD if remote missing).

   **Required output** — you MUST print this table before proceeding to Step 1d:

   ```text
   ## Branch Checkout Summary

   | Repo | Expected Branch | Action |
   |------|----------------|--------|
   | <orchestrator> | <STORY_ID> | already on branch / checked out / created |
   | <sibling> | <child-key> | already on branch / checked out / created |
   ```

   If checkout fails for any repo, stop with:

   ```text
   ## IMPLEMENT AGENT — HARD STOP: Branch Checkout Failed

   Repo: <repo-name>
   Expected branch: <branch>
   Reason: <git error>
   Resolution: Manually checkout the correct branch in <repo-name> and re-run.
   ```

   **Step 1d — Detect RESUMING**:

   Read `tasks.md` and scan for any line matching `- [x]` or `- [X]`.
   - If **any** completed task is found: set **RESUMING = true**. Steps 2 and 4 will be skipped.
   - If **no** completed tasks are found: set **RESUMING = false**. Run all steps in order.

2. **Check checklists status** *(skip this step if RESUMING = true)*

   If FEATURE_DIR/checklists/ exists:
   - Scan all checklist files in the checklists/ directory
   - For each checklist, count:
     - Total items: All lines matching `- [ ]` or `- [X]` or `- [x]`
     - Completed items: Lines matching `- [X]` or `- [x]`
     - Incomplete items: Lines matching `- [ ]`
   - Create a status table:

     ```text
     | Checklist | Total | Completed | Incomplete | Status |
     |-----------|-------|-----------|------------|--------|
     | ux.md     | 12    | 12        | 0          | ✓ PASS |
     | test.md   | 8     | 5         | 3          | ✗ FAIL |
     | security.md | 6   | 6         | 0          | ✓ PASS |
     ```

   - Calculate overall status:
     - **PASS**: All checklists have 0 incomplete items
     - **FAIL**: One or more checklists have incomplete items

   - **If any checklist is incomplete**:
     - Display the table with incomplete item counts
     - **STOP** and ask: "Some checklists are incomplete. Do you want to proceed with implementation anyway? (yes/no)"
     - Wait for user response before continuing
     - If user says "no" or "wait" or "stop", halt execution
     - If user says "yes" or "proceed" or "continue", proceed to step 3

   - **If all checklists are complete**:
     - Display the table showing all checklists passed
     - Automatically proceed to step 3

3. Load implementation context:

   **Load now (every run)**:
   - `tasks.md` — already read in step 1; do **not** re-read. Use the in-memory content for all subsequent steps.
   - `plan.md` — read now for tech stack, architecture, and file structure.

   **Lazy-load (per phase, only when needed)**:
   - `data-model.md` — load before the first phase that involves data-layer, entity, or schema tasks.
   - `contracts/` — load before the first phase that involves API resolvers, GraphQL schema, or contract tests.
   - `research.md` — load before the first phase that references technical decisions or library constraints.
   - `quickstart.md` — load before the first phase that involves integration or E2E scenarios.

   Do **not** read lazy docs upfront. Load them immediately before the phase that first requires them, then keep them in context for all subsequent phases.

4. **Project Setup Verification** *(skip this step if RESUMING = true)*

   Create/verify ignore files based on actual project setup:

   **Detection & Creation Logic**:
   - Check if the following command succeeds to determine if the repository is a git repo (create/verify .gitignore if so):

     ```sh
     git rev-parse --git-dir 2>/dev/null
     ```

   - Check if Dockerfile* exists or Docker in plan.md → create/verify .dockerignore
   - Check if .eslintrc* exists → create/verify .eslintignore
   - Check if eslint.config.* exists → ensure the config's `ignores` entries cover required patterns
   - Check if .prettierrc* exists → create/verify .prettierignore
   - Check if .npmrc or package.json exists → create/verify .npmignore (if publishing)
   - Check if terraform files (*.tf) exist → create/verify .terraformignore
   - Check if .helmignore needed (helm charts present) → create/verify .helmignore

   **If ignore file already exists**: Verify it contains essential patterns, append missing critical patterns only
   **If ignore file missing**: Create with full pattern set for detected technology

   **Common Patterns by Technology** (from plan.md tech stack):
   - **Node.js/JavaScript/TypeScript**: `node_modules/`, `dist/`, `build/`, `*.log`, `.env*`
   - **Python**: `__pycache__/`, `*.pyc`, `.venv/`, `venv/`, `dist/`, `*.egg-info/`
   - **Java**: `target/`, `*.class`, `*.jar`, `.gradle/`, `build/`
   - **C#/.NET**: `bin/`, `obj/`, `*.user`, `*.suo`, `packages/`
   - **Go**: `*.exe`, `*.test`, `vendor/`, `*.out`
   - **Ruby**: `.bundle/`, `log/`, `tmp/`, `*.gem`, `vendor/bundle/`
   - **PHP**: `vendor/`, `*.log`, `*.cache`, `*.env`
   - **Rust**: `target/`, `debug/`, `release/`, `*.rs.bk`, `*.rlib`, `*.prof*`, `.idea/`, `*.log`, `.env*`
   - **Kotlin**: `build/`, `out/`, `.gradle/`, `.idea/`, `*.class`, `*.jar`, `*.iml`, `*.log`, `.env*`
   - **C++**: `build/`, `bin/`, `obj/`, `out/`, `*.o`, `*.so`, `*.a`, `*.exe`, `*.dll`, `.idea/`, `*.log`, `.env*`
   - **C**: `build/`, `bin/`, `obj/`, `out/`, `*.o`, `*.a`, `*.so`, `*.exe`, `*.dll`, `autom4te.cache/`, `config.status`, `config.log`, `.idea/`, `*.log`, `.env*`
   - **Swift**: `.build/`, `DerivedData/`, `*.swiftpm/`, `Packages/`
   - **R**: `.Rproj.user/`, `.Rhistory`, `.RData`, `.Ruserdata`, `*.Rproj`, `packrat/`, `renv/`
   - **Universal**: `.DS_Store`, `Thumbs.db`, `*.tmp`, `*.swp`, `.vscode/`, `.idea/`

   **Tool-Specific Patterns**:
   - **Docker**: `node_modules/`, `.git/`, `Dockerfile*`, `.dockerignore`, `*.log*`, `.env*`, `coverage/`
   - **ESLint**: `node_modules/`, `dist/`, `build/`, `coverage/`, `*.min.js`
   - **Prettier**: `node_modules/`, `dist/`, `build/`, `coverage/`, `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`
   - **Terraform**: `.terraform/`, `*.tfstate*`, `*.tfvars`, `.terraform.lock.hcl`
   - **Kubernetes/k8s**: `*.secret.yaml`, `secrets/`, `.kube/`, `kubeconfig*`, `*.key`, `*.crt`

5. Using the `tasks.md` content already loaded in step 3 (no re-read), extract and organise:
   - **Repo execution order**: Collect the unique set of `target repo` values across all tasks. Order them by first appearance in `tasks.md` and by cross-repo task dependencies (a repo whose tasks are depended upon by another repo must come first). This ordered list is the **repo queue**.
   - **Task phases per repo**: For each repo, collect its tasks and group them by phase (Setup → Tests → Core → Integration → Polish) in declared order. A task belongs to the phase it is listed under in `tasks.md`.
   - **Task dependencies**: Sequential vs parallel execution rules, sourced from the `Dependencies & Execution Order` section.
   - **Parallel groups**: Sourced from the `Parallel Opportunities` section. Apply only within the scope of a single repo's phase bucket.
   - **Task details**: ID, description, file paths, parallel markers [P], and `target repo` label.

6. Execute implementation **one repository at a time** — complete all phases for a repo before starting the next:

   **For each repo in the repo queue (in order):**

   a. **Skip if already complete**: If every task for this repo is already `[x]`, skip it and continue to the next repo.

   b. **Announce the repo**: Output a repo start banner before any work:

      ```text
      ## Repo: <repo-name>
      Phases to execute: <list of phases that have [ ] tasks for this repo>
      ```

   c. **Read source files once**: Before the first phase for this repo, read all relevant source files for this repo in a single batch. Do not re-read files between phases for the same repo.

   d. **Execute phases in order for this repo** (Setup → Tests → Core → Integration → Polish):

      For each phase that has at least one `[ ]` task for this repo:

      i. **Identify tasks**: Collect all `[ ]` tasks for this repo in this phase.

      ii. **Execute tasks**:
         - Honour `[P]` markers: issue tool calls for parallel tasks as a concurrent batch (different files, no dependency on each other within this repo's phase). Run sequential tasks one after the other.
         - Follow TDD order: test tasks before their corresponding implementation tasks within the same phase.
         - Tasks that touch the same file must remain sequential regardless of `[P]`.

      iii. **Mark each task complete immediately**: As soon as each individual task finishes, update `tasks.md` to mark it `[x]` before starting the next task. Do not batch updates.

      iv. **Phase-within-repo completion — verify then summarise**: Scan `tasks.md` for every task belonging to this phase **for this repo**. If any is still `[ ]`, mark it `[x]` now. Then output the summary table:

         ```text
         ## [<repo-name>] Phase <N> Complete — <Phase Name>

         | Task | Status |
         |------|--------|
         | T001 | ✓ done |
         | T002 | ✓ done |

         All tasks for this phase in <repo-name> marked [x] in tasks.md.
         ```

      v. **Gate before next phase (within same repo)**: After the summary, pause and ask:
         > "[`<repo-name>`] Phase \<N\> (\<Phase Name\>) is complete. Type **yes** to proceed to Phase \<N+1\> for this repo, or describe any issues to address first."

         Wait for user confirmation. If the user raises issues, address them (re-mark any re-done tasks `[x]`) before re-asking.

   e. **Repo completion — summarise**: Once all phases for this repo are done, output a repo completion banner:

      ```text
      ## Repo Complete — <repo-name>

      | Task | Phase | Status |
      |------|-------|--------|
      | T001 | Setup | ✓ done |
      | T003 | Core  | ✓ done |
      ...

      All tasks for <repo-name> marked [x] in tasks.md.
      ```

   f. **Gate before next repo**: After the repo completion banner, pause and ask:
      > "`<repo-name>` is fully implemented. Type **yes** to proceed to the next repo (`<next-repo-name>`), or describe any issues to address first."

      Wait for user confirmation before starting the next repo. If the user raises issues, address them before re-asking. If this is the last repo, skip the gate and proceed directly to step 9.

7. Implementation execution rules:
   - **Repo-first ordering**: Complete all phases for a repo before starting the next repo. Never interleave work across repos.
   - **Setup first within each repo**: Run the Setup phase tasks before Tests, Core, Integration, or Polish for that repo.
   - **Tests before code within each phase**: Write test stubs before implementing the corresponding feature, within the same repo's phase bucket.
   - **Core development**: Implement models, services, CLI commands, endpoints.
   - **Integration work**: Database connections, middleware, logging, external services.
   - **Polish and validation**: Unit tests, performance optimization, documentation.

8. Progress tracking and error handling:
   - Report progress after each completed task
   - Halt execution if any non-parallel task fails
   - For parallel tasks [P], continue with successful tasks, report failed ones
   - Provide clear error messages with context for debugging
   - Suggest next steps if implementation cannot proceed
   - **IMPORTANT**: Mark each completed task `[x]` in tasks.md immediately upon completion — do not batch updates. All tasks for a phase MUST be crossed off before the phase gate is presented.

9. Completion validation:
   - Verify all required tasks are completed
   - Check that implemented features match the original specification
   - Validate that tests pass and coverage meets requirements
   - Confirm the implementation follows the technical plan
   - **Always output a structured completion report as the final message** so the invoking agent can parse the outcome. Use this exact format:

     ```text
     ## Implementation Complete — <feature dir name>

     ### Tasks Summary
     | Task ID | Description | Repo | Status |
     |---------|-------------|------|--------|
     | T001    | <desc>      | <repo> | ✓ done |
     | T002    | <desc>      | <repo> | ✓ done |
     | T003    | <desc>      | <repo> | ✗ failed |

     ### Totals
     - Total tasks: <N>
     - Completed: <N>
     - Failed / skipped: <N>

     ### Files Changed
     | Repo | Files Added | Files Modified |
     |------|-------------|----------------|
     | <repo> | <count> | <count> |

     ### Validation
     - Unit tests: PASS / FAIL / SKIPPED
     - Compilation: PASS / FAIL / SKIPPED
     - Linting: PASS / FAIL / SKIPPED

     ### Overall Result
     COMPLETE | PARTIAL | FAILED
     ```

   - If implementation did not complete fully, set **Overall Result** to `PARTIAL` (some tasks done) or `FAILED` (no tasks completed), and list the blocking reason under a `### Blockers` section.

Note: This command assumes a complete task breakdown exists in tasks.md. If tasks are incomplete or missing, suggest running `/speckit.tasks` first to regenerate the task list.

10. **Check for extension hooks**: After completion validation, check if `.specify/extensions.yml` exists in the project root.
    - If it exists, read it and look for entries under the `hooks.after_implement` key
    - If the YAML cannot be parsed or is invalid, skip hook checking silently and continue normally
    - Filter out hooks where `enabled` is explicitly `false`. Treat hooks without an `enabled` field as enabled by default.
    - For each remaining hook, do **not** attempt to interpret or evaluate hook `condition` expressions:
      - If the hook has no `condition` field, or it is null/empty, treat the hook as executable
      - If the hook defines a non-empty `condition`, skip the hook and leave condition evaluation to the HookExecutor implementation
    - For each executable hook, output the following based on its `optional` flag:
      - **Optional hook** (`optional: true`):
        ```
        ## Extension Hooks

        **Optional Hook**: {extension}
        Command: `/{command}`
        Description: {description}

        Prompt: {prompt}
        To execute: `/{command}`
        ```
      - **Mandatory hook** (`optional: false`):
        ```
        ## Extension Hooks

        **Automatic Hook**: {extension}
        Executing: `/{command}`
        EXECUTE_COMMAND: {command}
        ```
    - If no hooks are registered or `.specify/extensions.yml` does not exist, skip silently

---

## Next Step — Raise Pull Requests

Once the `## Implementation Complete` report above has been produced and step 10 hooks have run, the implementation phase is finished.

To commit changes, raise pull requests, and update Jira child stories, run:

```
/speckit.ship STORY_ID=<STORY_ID>
```
