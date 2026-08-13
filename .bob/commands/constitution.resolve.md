---
description: Resolve runtime effective constitution artifacts by loading global and local constitutions with defined precedence.
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Goal

Build runtime policy artifacts from two constitution sources with local precedence:

1. **Global constitution** (optional) — may come from a local sibling repo directory, an explicit file path, or a remote location fetched via connector. If absent, the effective constitution contains the local constitution only.
2. **Local constitution** (required) — `.specify/memory/constitution.md`

Local policy has higher precedence than global policy.

## Operating Constraints

- Use filesystem tools for local file operations.
- Use configured MCP tools for remote fetches (see MCP Provider Routing Table in Step 1a for tool mapping).
- Do not call shell commands.
- Do not modify either source constitution.
- This command is a preflight and must run before downstream planning/implementation.
- Keep `constitution.pathlint` behavior unchanged.

## Global Constitution Source Modes

The global constitution source is configurable. Three modes are supported. The **mode is always driven by `settings.yaml`** (or the built-in default); `$ARGUMENTS` may only supply values required by the active mode — not switch it.

| Mode | Required `$ARGUMENTS` | Example value | Resolution |
|---|---|---|---|
| **Local path** | _(none required)_ | — | Reads `path` from `settings.yaml`. If a `global_path` argument is passed while mode is not `local`, reject with error. |
| **External (MCP)** | `context_id` _(prompted if absent)_ | `ctx_abc123` | Fetch via configured MCP provider (see step 1a). If `context_id` is not in `$ARGUMENTS`, the agent will ask the user for it. Requires `connection-types.MCP.provider` in `settings.yaml`. |
| **None / skip** | _(none)_ | — | No global constitution. Effective constitution = local only. |

## Execution Steps

1. **Determine paths and source mode**

   Resolve the active mode using this precedence order (first match wins):

   1. **`settings.yaml`** (project root) — read `constitution.global_source.mode`. If the file is missing, fall through.
   2. **Built-in default** — `mode: local`, `path: ../org-policy-files/engineering`.

   This means when invoked as a handoff from `speckit.constitution` (no `$ARGUMENTS`), the workspace config file drives the behaviour automatically.

   **Mode-argument validation** — once the active mode is resolved, check `$ARGUMENTS` for conflicts:
   - If `$ARGUMENTS` contains `context_id` but mode is **not** `external` → stop with error: "context_id argument is only valid for mode:external. Current mode is '{mode}'."
   - If `$ARGUMENTS` contains `global_path` but mode is **not** `local` → stop with error: "global_path argument is only valid for mode:local. Current mode is '{mode}'."
   - If `$ARGUMENTS` contains `global_skip` but mode is **not** `none` → stop with error: "global_skip argument is only valid for mode:none. Current mode is '{mode}'."

   **Step 1a — For `mode: external` (MCP-based global constitution):**

   - Read `constitution.global_source.connection-types` from `settings.yaml`.
   - Validate: `connection-types.MCP` must be present. If missing, stop with error: "MCP connection type not configured in settings.yaml".
   - Validate: `connection-types.MCP.provider` must equal `context-studio` (only supported provider for now). If different or missing, stop with error: "Unsupported MCP provider: {provider}. Only 'context-studio' is currently supported."
   - **MCP Provider Routing Table** (deterministic mapping for future extensibility):
     ```
     provider → MCP tool to invoke
     context-studio → context-broker-hybrid-query
     [future providers registered here as they are added]
     ```
   - **Require context_id**: Check `$ARGUMENTS` for `context_id`. If absent, **ask the user**: "Please provide the context ID for context-studio (format: `context_id: <id>`)." Wait for the user's response before proceeding. Do not error or stop.
   - Set mode = **external**, store `context_id`, provider = `context-studio`.

   Resolved modes:
   - `mode: local` + `path` → set mode = **local**
   - `mode: external` + validated MCP provider + `context_id` → set mode = **external**, provider = **context-studio**
   - `mode: none` → set mode = **none** (`global_found = false` immediately)

   Fixed output paths (not configurable unless overridden in `$ARGUMENTS`):
   - Global snapshot: `.specify/runtime/global-constitution.md`
   - Effective output: `.specify/runtime/effective-constitution.md`
   - Report output: `.specify/runtime/effective-constitution-report.md`
   - Local constitution path: `.specify/memory/constitution.md` (or `local_path` from `$ARGUMENTS`).

2. **Resolve and validate inputs**

   **Local constitution (required):**
   - Verify `.specify/memory/constitution.md` exists. If missing, stop with a clear error including the missing path. Do not continue.

   **Global constitution (optional — behaviour by mode):**
   - **Mode: local** — Treat the path as a directory. Find the first existing file in order:
     1. `constitution.md`
     2. `engineering-constitution.md`
     3. `global-constitution.md`
     - If the path is a direct file (not a directory), use it as-is.
     - If the directory/file does not exist → set `global_found = false`. Do **not** error; proceed with local-only mode.
     - If the directory exists but none of the candidate files are found → set `global_found = false`. Log a warning in the report.
     - **On success (file found)**: Read the file content and copy it verbatim to `.specify/runtime/global-constitution.md`. Set `global_found = true`.
   - **Mode: external** — Call the MCP tool based on the provider routing table (see Step 1a):
     - For `provider: context-studio`, invoke `context-broker-hybrid-query` (substituting the user-supplied `context_id`) with this payload and construct the contents of the constitution.md. Do not leave out any information:
       ```json
       {
         "context_id": "<context_id>",
         "AgentPersona": "technical_reader",
         "query": "constitution engineering standards code quality architecture REST API testing CI/CD UX requirements",
         "sources": ["vector", "graph"],
         "vector_params": {
           "top_k": 20
         },
         "graph_params": {
           "top_k": 20,
           "max_depth": 1
         }
       }
       ```
     - From the response, construct the full content of the global constitution. **Do not omit any information returned** — include all rules, standards, and guidance present in the result.
     - On success → set `global_found = true`, store constructed content as the global constitution text. Copy to `.specify/runtime/global-constitution.md`.
     - On failure (context not found, auth error, MCP error) → hard stop with error: "Failed to fetch global constitution from context-studio (context_id: {context_id}). {error_details}". Do not proceed.
   - Record which source was used (mode + resolved path/context_id) in the report (step 9).

3. **Ensure runtime directory exists**
   - Create `.specify/runtime/` if missing.

4. **Global file snapshot**
   - Already handled in Step 2: files are copied to `.specify/runtime/global-constitution.md` during resolution.
   - If `global_found = false`, no global file is written. Do **not** write an empty or placeholder file.

5. **Read source files**
   - Read the complete text of `.specify/memory/constitution.md`.
   - **Detect if local constitution is a template**: scan the content for unfilled placeholder markers matching the pattern `[WORD]` or `[WORD_WORD_...]` (e.g., `[PROJECT_NAME]`, `[PRINCIPLE_1_DESCRIPTION]`). If any such markers are found, set `local_is_template = true`; otherwise `local_is_template = false`.
   - If `global_found = true`, the global content is already available from step 2.

6. **Compose effective constitution**

   Determine the composition case using `global_found` and `local_is_template`:

   **Case A — `global_found = true` AND `local_is_template = false` (global + real local):**
   - First run step 7 (conflict analysis), then write this file.
   - Structure:
     ```
     <!-- EFFECTIVE CONSTITUTION
          Generated  : <ISO-8601 UTC timestamp>
          Global     : <source — local path, or external (context-studio, context_id: xyz)>
          Local      : .specify/memory/constitution.md
          Precedence : local-over-global
          If confused, give precedence to the local constitution.
     -->

     # Effective Constitution

     **Generated:** <timestamp>
     **Precedence:** Local constitution overrides global on conflict.
     **Conflict report:** `.specify/runtime/effective-constitution-report.md`

     > If a rule in Part 1 conflicts with a rule in Part 2, Part 2 wins — always.

     ---

     ## Resolved Rules — Authoritative Reference

     > **Read this section first.** These are the final, binding rules for every
     > topic where global and local conflict. Agents MUST apply these rules.
     > Do not apply the corresponding Part 1 rule for any topic listed here.

     | # | Topic | Authoritative Rule (local wins) |
     |---|---|---|
     | 1 | <topic> | <one-to-two sentence authoritative local rule> |
     | … | … | … |

     (one row per conflict identified in step 7; omit table if no conflicts found)

     ---

     ## PART 1 — Global Baseline

     > Source: <global source description — include provider and context_id if mode is external>
     > Rules superseded by Part 2 are documented in the conflict report.

     <full verbatim content of global constitution>

     ---

     ## PART 2 — Local Constitution (Authoritative)

     > Source: `.specify/memory/constitution.md`
     > Rules here take precedence over Part 1 wherever a conflict exists.

     <full verbatim content of local constitution>
     ```

   **Case B — `global_found = true` AND `local_is_template = true` (global-only):**
   - Skip step 7. No conflict analysis needed — local is unfilled and must not be merged.
   - Structure:
     ```
     <!-- EFFECTIVE CONSTITUTION
          Generated  : <ISO-8601 UTC timestamp>
          Global     : <source — local path, or external (context-studio, context_id: xyz)>
          Local      : .specify/memory/constitution.md (template — not applied)
          Precedence : global only — local constitution is a template
     -->

     # Effective Constitution

     **Generated:** <timestamp>
     **Mode:** Global-only (local constitution is an unfilled template and was not applied).

     > Local constitution contains unfilled placeholders and was excluded. All rules below are from the global baseline.

     ---

     <full verbatim content of global constitution>
     ```

   **Case C — `global_found = false` (local-only, regardless of template status):**
   - Structure:
     ```
     <!-- EFFECTIVE CONSTITUTION
          Generated  : <ISO-8601 UTC timestamp>
          Global     : none (not found or skipped)
          Local      : .specify/memory/constitution.md
          Precedence : local only — no global baseline present
     -->

     # Effective Constitution

     **Generated:** <timestamp>
     **Mode:** Local-only (no global constitution was found or configured).

     > No global baseline is present. All rules below are authoritative as-is.

     ---

     <full verbatim content of local constitution>
     ```
   - Skip steps 7 and the conflict analysis entirely. No Resolved Rules table needed.
   - Skip writing `.specify/runtime/global-constitution.md`.

   In all cases: do **not** add inline annotations or conflict markers inside the constitution body.

7. **Identify conflicts (analysis pass — read-only, Case A only)**
   - Skip entirely if `global_found = false` or `local_is_template = true`.
   - Review both parts and identify every section or rule where local and global directly conflict.
   - Collect each conflict: global rule (with section reference), local rule (with section reference), resolution (local wins).
   - Do **not** modify either source file.
   - **This step MUST complete before writing step 6** — the Resolved Rules table is populated from the output here.

8. **Write effective constitution**
   - Using the output of step 7 (if applicable), write `.specify/runtime/effective-constitution.md` per the template in step 6.
   - Case A: populate the Resolved Rules table (one row per conflict; omit the table entirely if no conflicts were found).
   - Case B: write the global-only template.
   - Case C: write the local-only template.

9. **Write resolution report**
   - Write `.specify/runtime/effective-constitution-report.md` with:
     - **Timestamp** (ISO 8601 UTC)
     - **Global source mode**: local path / external (with provider name and context_id if applicable) / none — with the resolved path or provider details
     - **Local source**: `.specify/memory/constitution.md`
     - **global_found**: true / false (with reason if false)
     - **Precedence rule applied**
     - **Status**: `PASS` or `BLOCKED`
     - **Override Callouts** section (only when `global_found = true`):
       - For every conflict: Override ID, global rule (quoted), local rule (quoted), severity (CRITICAL/HIGH/MEDIUM/LOW), resolution, downstream impact
       - If no conflicts found: `No explicit rule-level overrides identified in this run.`
     - **Blocking errors** (missing local file, missing context_id, MCP validation failure, MCP fetch failure)

10. **Output summary**
    - Return artifact paths and readiness state.

## Output Contract

On success, always produce:
- `.specify/runtime/effective-constitution.md` — either two-part concat (with Resolved Rules table) or local-only, depending on `global_found`.
- `.specify/runtime/effective-constitution-report.md` — source mode, conflict callouts, status.
- `.specify/runtime/global-constitution.md` — only when `global_found = true`.

On failure, do not produce partial artifacts unless the report clearly explains why status is `BLOCKED`.
