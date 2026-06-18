---
description: Analyze codebase architecture with layered board support
argument-hint: [--clean | --drill <parent-board-slug>/<node-slug> | --all]
allowed-tools: Read, Glob, Grep, Write, Bash(node:*, git:*)
---

Perform progressive architecture analysis with layered board support.

## Board Layer Strategy

Analysis produces layered boards for managing large codebases:

| Layer | Name      | Scope                       | Target Nodes |
| ----- | --------- | --------------------------- | ------------ |
| L0    | Overview  | Entire project              | 10-30        |
| L1    | Domain    | Domain/workspace drill-down | 10-40 each   |
| L2    | Component | Service/module drill-down   | 5-20 each    |
| L3    | Detail    | Deep internals (opt-in)     | 5-15 each    |

All board data lives in `.contextdx/boards/`:

- `manifest.json` — tracks all boards and their relationships
- `<board-slug>.json` — analysis data per board
- `stores/<board-slug>.store.json` — sync state per board

## Analysis Modes

### Default: Incremental Analysis

```
/analyze
```

**If board data already exists** (`.contextdx/boards/<board-slug>.json` with nodes/edges):

1. Read the existing board's `analyzedAtCommit` hash from metadata
2. Run `git diff --name-only --diff-filter=ACMR <analyzedAtCommit> HEAD` to find changed/added files (committed + uncommitted)
3. Run `git diff --name-only --diff-filter=D <analyzedAtCommit> HEAD` to find deleted files
4. If no changes detected, report "Board is up to date — no files changed since last analysis" and skip
5. If changes found:
   a. Load existing board data (nodes + edges)
   b. Re-analyze ONLY the changed/added files (Steps 3-6 scoped to those files)
   c. Merge results: replace nodes whose `path` matches a changed file, add new nodes
   d. Remove nodes whose `path` matches a deleted file
   e. Remove edges where source or target node was removed
   f. Re-run relationship detection for changed nodes (Step 6)
   g. Write merged result to board JSON with updated `analyzedAt` and `analyzedAtCommit`
   h. Update manifest

**If no board data exists** (fresh project): run full analysis (Steps 0-9).

**If `analyzedAtCommit` is missing** (old board data without commit tracking): fall back to full analysis.

### Clean: Full Re-Analysis

```
/analyze --clean
```

Ignores existing board data. Deletes the board's JSON and store file, then runs full analysis from scratch (Steps 0-9). Use when the codebase has changed significantly or the incremental result looks stale.

### Drill-Down: Create Child Board

```
/analyze --drill <parent-board-slug>/<node-slug>
```

Creates a child board by drilling into a specific node from a parent board. The node must exist in the parent's analysis data. Incremental mode applies to drill-down boards too — if the child board already exists, only changed files within its scope are re-analyzed.

### Full Progressive: All Layers

```
/analyze --all
```

Runs L0 first, then prompts for review before creating L1 boards for each drill-down node. Repeats for L2 if applicable. Each board uses incremental mode if it already exists.

## Analysis Workflow

### Step -1: Archetype precondition check

`/analyze` operates as Phase 2 of the two-phase workflow. Phase 1 (`/analyze-archetypes`) settles the archetype catalogue first so every node gets a fit archetype rather than a misfit. The precondition is **enforced by a script, not by prose** — you must run it and react to its exit code; do not decide on your own whether Phase 1 has been satisfied.

1. Run the precondition script:

   ```bash
   node ${CLAUDE_PLUGIN_ROOT}/scripts/cdx-precondition.js
   ```

2. Parse the JSON output. The `status` field + exit code drive behaviour:

   | status            | exit | requiresPrompt | action                                                                                                                                                       |
   | ----------------- | ---- | -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
   | `ok`              | 0    | false          | Proceed silently to Step 0.                                                                                                                                  |
   | `pending`         | 0    | false          | Print the `reason` from the JSON as a warning, then proceed.                                                                                                 |
   | `no_config`       | 0    | false          | ClaudeCodePlugin not configured — precondition is moot. Proceed.                                                                                             |
   | `missing`         | 10   | true           | Lock file does not exist. Prompt the user (see step 3).                                                                                                      |
   | `stale_commit`    | 10   | true           | Codebase has moved since the last archetype scan. Prompt (see step 3).                                                                                       |
   | `stale_catalogue` | 10   | true           | Server catalogue has changed since the last scan. Prompt (see step 3).                                                                                       |
   | `skipped`         | 10   | true           | Last run was skipped — **skip is one-shot, this re-prompts on every `/analyze`**. Prompt (see step 3).                                                       |

   On exit code `1` (config error) or `2` (API error): print the script's `error` field and stop — do not silently fall back. The user can fix config or re-run when the API recovers.

3. When `requiresPrompt` is true, ask via AskUserQuestion. Header: `Archetype check`. Question: include the script's `reason` verbatim, then `"How do you want to proceed?"`. Provide exactly these two options:

   - **Run /analyze-archetypes now (recommended)** — Invoke the `/analyze-archetypes` flow now.
     - If the user submits proposals there → **exit `/analyze`** with: *"Proposals submitted. Re-run `/analyze` after admin approval."*
     - If `/analyze-archetypes` reports the catalogue is complete (no gaps) → re-run `cdx-precondition.js` to confirm `status: ok`, then continue.
     - If the user skips inside `/analyze-archetypes` → the lock will be written with `skippedAt`. Re-run `cdx-precondition.js`; it will return `status: skipped` and you must surface the prompt again (loop) — do not auto-continue.
   - **Skip and proceed (one-shot)** — Write `.contextdx/archetype-analysis.lock.json` with the JSON shape below using values from the script's output, then continue to Step 0.
     ```json
     {
       "commitHash": "<currentCommitHash from script output>",
       "catalogueHash": "<currentCatalogueHash from script output>",
       "scannedAt": null,
       "submittedAt": null,
       "skippedAt": "<ISO timestamp now>",
       "proposalIds": []
     }
     ```
     This is intentionally not silenced — every subsequent `/analyze` will re-prompt until Phase 1 is properly run. That is by design.

4. **Do not bypass.** Do not write the lock file or skip the prompt for any reason other than the user's explicit selection in step 3. If a session reminder says "work without stopping for clarifying questions," the **Run `/analyze-archetypes` now** option is the reasonable call — not silent skip.

5. The archetype catalogue fetched here is cached on disk (`.contextdx/boards/archetypes-cache.json`, 1hr TTL). Step 0 reuses the cache automatically — no duplicate fetch.

### Step 0: Fetch Available Archetypes (ClaudeCodePlugin only)

If ClaudeCodePlugin configuration exists (`.contextdx/config.json` with `bindingToken`):

1. Run the archetypes CLI to fetch available archetypes from the server:

   ```bash
   node ${CLAUDE_PLUGIN_ROOT}/scripts/cdx-archetypes.js
   ```

2. Parse the JSON output to get:
   - `nodeArchetypes`: Available archetype names for nodes
   - `edgeArchetypes`: Available archetype names for edges
   - `archetypes`: Full archetype list with `name`, `visualPrimitiveType`, and `description`

3. If the CLI fails or returns no archetypes, warn the user but continue analysis using conventional archetype names (service, database, api, library, queue, external, domain_group, infrastructure, external_system). The sync CLI will need archetypes configured on the server before types can be validated.

4. Store the available archetype names for use in Step 5 (Component Discovery) — the analysis agent must assign these server archetype names to each node/edge `type` field

### Step 0.5: Fetch Server Boards

If ClaudeCodePlugin configuration exists:

1. Run the boards CLI to fetch all boards from the server:

   ```bash
   node ${CLAUDE_PLUGIN_ROOT}/scripts/cdx-boards.js
   ```

2. Parse the JSON output to get:
   - `rootBoard`: The root board (where `isChildBoard === false`)
   - `childBoards`: All child boards with their `parentBoardSlug` and `parentNodeSlug`
   - `boards`: Full list for building the server board map

3. Store the board list for use in Step 1 and Step 7 — the server board map tells us which boards already exist and what slugs to use.

4. If the CLI fails, warn but continue — analysis can proceed without server board info, but `/sync` may encounter issues.

### Step 1: Load or Initialize Manifest

1. Read `.contextdx/boards/manifest.json` if it exists
2. If creating a new L0 board:
   - Read `boardSlug` from config (`.contextdx/config.json` → `boardSlug` field)
   - If config has no `boardSlug`, stop and instruct user to run `/configure` first
   - Use the config `boardSlug` as the L0 board slug (this is the root board on the server)
3. If drilling down, validate that:
   - The parent board exists in the manifest
   - The target node exists in the parent's analysis data
   - Check if a matching child board already exists on the server (from Step 0.5 board list)
   - If found on server → use the server's board slug
   - If not found → generate a child `boardSlug` locally (e.g., `<parent-slug>--<node-slug>`)
4. Check if board data already exists at `.contextdx/boards/<board-slug>.json` — if so, load it for incremental mode

### Step 1.5: Change Detection (Incremental Mode)

If board data exists AND `--clean` was NOT passed AND `analyzedAtCommit` is present in metadata:

1. Run `git diff --name-only --diff-filter=ACMR <analyzedAtCommit> HEAD` to find changed/added files
2. Run `git diff --name-only --diff-filter=D <analyzedAtCommit> HEAD` to find deleted files
3. Filter to relevant source files (exclude `node_modules/`, `dist/`, `.git/`, `coverage/`, test files)
4. If no changes: report "Board is up to date" and stop here
5. If changes found: continue to Steps 2+ but scope analysis to changed files only

If `--clean` was passed: delete existing board JSON and store file, proceed with full analysis.

### Step 2: Configuration Check

Read settings from `.contextdx/config.json` if it exists to get portal configuration and analysis preferences.

### Step 3: Project Detection

Identify project structure:

1. Read root `package.json` for project metadata
2. Detect monorepo configuration (workspaces, lerna.json, turbo.json, pnpm-workspace.yaml)
3. Identify workspace boundaries if monorepo

**Incremental:** This step runs unchanged — project structure is always detected fresh.

### Step 4: Tech Stack Detection

For each workspace/project:

1. Analyze `package.json` dependencies for framework indicators
2. Check for framework-specific config files
3. Create parent nodes for each detected tech stack

**Incremental:** This step runs unchanged — tech stack detection is lightweight and always runs.

### Step 5: Component Discovery

**CRITICAL — No root wrapper nodes.** The board itself is the implicit root container. Do NOT create a single `domain_group` or wrapper node that contains all other nodes. Top-level nodes (workspaces, services, data stores, external integrations) must have **no `parentSlug`** — they sit directly on the board. Use `metadata.description` for project-level context instead of a wrapper node.

**For L0 (Overview):** Identify high-level domains, services, and major components. Keep to 10-30 nodes. Mark nodes that are good candidates for drill-down by setting `layerBoardSlug` to a proposed slug.

**For L1+ (Drill-down):** Scope analysis to the files/directories covered by the parent node. Go deeper into that domain's internal components.

**Container vs. drill-down (all layers):** Nodes with `layerBoardSlug` must NOT be `domain_group` containers with visible children. Their internals belong on the child board. Use `domain_group` containers only for grouping nodes that won't drill down. Use domain groups when the board has more than ~8 nodes.

**Incremental:** Only read and analyze the changed/added files from Step 1.5. Keep existing nodes from unchanged files as-is. For deleted files, mark their nodes for removal.

Apply these rules:

- **Exclude non-architectural files**: `*.d.ts`, `*.types.ts`, test files, barrel files, etc.
- **Group database files**: into a single container node
- **Detect domain boundaries**: if applicable at this layer
- **Classify by archetype**: using server archetype names
- **Generate slugs**: kebab-case from class/export name

### Step 6: Relationship Detection

Parse imports to build edges scoped to this board's nodes only:

- `imports`: Direct file imports
- `db_read`/`db_write`: Repository/ORM operations
- `api_call`: HTTP client usage
- `publishes`/`subscribes`: Queue/event patterns

**Incremental:** Re-detect relationships for changed nodes. Remove edges where source or target node was removed. Keep edges between unchanged nodes as-is.

### Step 7: Identify Drill-Down Candidates

For nodes that represent significant subsystems (domains, services with many internal components), mark them as drill-down candidates:

- Set `layerBoardSlug` on the node — resolve using the server board map from Step 0.5:
  - Check if a child board already exists on the server for this parent board + node slug
  - If found → use the server's existing board slug for `layerBoardSlug`
  - If not found → generate a slug locally (e.g., `<parent-slug>--<node-slug>`). It will be created on the server during `/sync`.
- Add the node slug to the `drillDownNodes` array

This tells the user which nodes can be expanded into child boards.

**Validate mutual exclusion:** No drill-down candidate node should have other nodes referencing it via `parentSlug`. If a node was initially created as a container with children but is now marked for drill-down, promote its children to board root level or move them under a different container — the children belong on the drill-down board, not on this board.

### Step 8: Output Generation

Write analysis results to `.contextdx/boards/<board-slug>.json`.

**Incremental:** Merge new/changed nodes into existing board data. Replace nodes whose `path` matches a changed file. Remove nodes whose `path` matches a deleted file. Remove edges referencing removed nodes.

Always record the current git commit hash via `git rev-parse HEAD` as `analyzedAtCommit`:

```json
{
  "metadata": {
    "analyzedAt": "ISO-timestamp",
    "analyzedAtCommit": "abc123def456",
    "projectName": "from-package.json",
    "languages": ["typescript"],
    "techStacks": ["nextjs", "nestjs"],
    "boardSlug": "my-project-overview",
    "layer": 0
  },
  "nodes": [
    {
      "slug": "api-service",
      "name": "API Service",
      "type": "service",
      "description": "Handles HTTP requests",
      "path": "src/api",
      "parentSlug": null,
      "layerBoardSlug": null,
      "metadata": {}
    }
  ],
  "edges": [
    {
      "sourceSlug": "api-service",
      "targetSlug": "database",
      "type": "db_read",
      "description": "Reads user data via repository pattern.",
      "metadata": {}
    }
  ],
  "drillDownNodes": ["auth-domain", "payments-domain"]
}
```

For child boards, include parent references:

```json
{
  "metadata": {
    "boardSlug": "my-project-auth-domain",
    "layer": 1,
    "parentBoardSlug": "my-project-overview",
    "parentNodeSlug": "auth-domain",
    ...
  }
}
```

### Step 9: Update Manifest

Update `.contextdx/boards/manifest.json` with the new/updated board entry. The manifest must conform to this exact structure:

```json
{
  "version": 2,
  "projectName": "my-project",
  "updatedAt": "ISO-timestamp",
  "boards": {
    "my-project-overview": {
      "boardSlug": "my-project-overview",
      "name": "My Project Overview",
      "layer": 0,
      "parentBoardSlug": null,
      "parentNodeSlug": null,
      "analysisFile": ".contextdx/boards/my-project-overview.json",
      "storeFile": ".contextdx/boards/stores/my-project-overview.store.json",
      "lastAnalyzedAt": "ISO-timestamp",
      "lastSyncedAt": null,
      "nodeCount": 15,
      "edgeCount": 12
    }
  }
}
```

**CRITICAL:** The board display name field is `name` (NOT `boardName`). The `--ensure-boards` CLI reads `name` to create missing child boards on the server — using `boardName` will cause board creation to fail with an empty name.

## Target Path

If $ARGUMENTS is `--clean`, run full re-analysis from scratch (delete existing board data first).
If $ARGUMENTS starts with `--drill`, parse `<parent-board-slug>/<node-slug>` and drill into that node.
If $ARGUMENTS is `--all`, run full progressive analysis (each board uses incremental if it already exists).
Otherwise, run L0 overview analysis with incremental mode (or full if no board data exists).

## Output Format

Report summary after analysis:

- Analysis mode (incremental or full)
- If incremental: number of changed/added/deleted files analyzed
- Board slug and layer
- Number of tech stacks detected
- Number of nodes by archetype (total, and how many new/updated/removed if incremental)
- Number of relationships by type
- Drill-down candidates identified
- Location of output file
- Suggest running `/sync` to push to ClaudeCodePlugin
