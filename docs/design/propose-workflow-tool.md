# Design — Propose Workflow Tool

Status: design-doc gate
Goal: Propose Workflow Tool
Branch: `goal/propose-workfl-ffaa495f`

## 1. Summary

Add a `propose_workflow` tool that lets agents surgically propose a new
workflow (or edit an existing one) without touching the rest of the project
configuration. The proposal system already recognises `"workflow"` as a type
in the tool extension's `PROPOSAL_TYPE_ENUM`, but the server-side
`ProposalType` union excludes it, making `view_proposal("workflow")` fail
with HTTP 400.

This design adds `"workflow"` as a first-class `ProposalType` across the full
stack — server, tool extension, and UI — reusing the existing generic proposal
infrastructure with minimal new code.

## 2. Comparative Design

### 2.1 Approach A — Minimal composition (selected)

Compose the existing generic proposal pipeline end-to-end:

- **Server plugin**: One `makeYamlPlugin` call with `requiredFields: ["id", "name", "gates"]`.
  Reuse `validateGoalInlineWorkflow` from `proposal-types.ts` for gate schema validation
  (extract to a shared name if needed, but the function already takes `unknown` and
  returns `ParseError | null` — it's directly reusable).
- **Seed route**: Works automatically — `POST /api/sessions/:id/proposal/:type/seed`
  is generic. Only the `if (proposalType === "goal")` enrichment branches need a guard;
  "workflow" falls through to the generic write path untouched.
- **Accept route**: New `POST /api/sessions/:id/proposal/workflow/accept` that reads
  the proposal draft, validates gates, and merges a single workflow into
  `project.yaml::workflows` via `projectConfigStore.getWorkflows()` /
  `projectConfigStore.setWorkflows()`.
- **Tool extension**: ~40 lines — one `pi.registerTool({ name: "propose_workflow", ... })`
  call in `defaults/tools/proposals/extension.ts`, following the exact pattern of
  `propose_role` / `propose_tool` / `propose_staff`.
- **UI tool registry**: 1 line in `PROPOSAL_TOOL_NAMES` + 1 line in `PROPOSAL_LABELS`.
- **UI proposal registry**: ~20 lines — add to `ProposalType`, `PROPOSAL_TYPES`,
  `PROPOSAL_TYPE_REGISTRY`, `PROPOSAL_TAB_LABELS`.
- **UI panel**: ~60 lines — `workflowProposalPanel()` renders the workflow as a
  read-only inspector using `renderWorkflowInspector({ workflow, scope: "proposal" })`
  plus standard accept/reject/dismiss buttons. The accept handler POSTs to the new
  server accept endpoint.
- **`edit_proposal`**: Works automatically — the generic `/edit` route operates on
  any `ProposalType`. No changes needed beyond adding the type.

**Reused symbols and protecting tests**:

| Symbol | Home | Protecting tests |
|---|---|---|
| `makeYamlPlugin` | `proposal-types.ts` | Plugin parse/serialize exercised by every proposal type test |
| `validateGoalInlineWorkflow` | `proposal-types.ts` | Exercised indirectly via goal-proposal acceptance tests |
| `isProposalType`, `PROPOSAL_TYPES` | `proposal-files.ts` | Type enumeration tested via proposal-file read/write tests |
| `seedProposal`, `callGateway` | `extension.ts` | E2E proposal-tool journeys |
| `renderWorkflowInspector` | `workflow-page.ts` | Workflow embed UI fixture tests |
| `PROPOSAL_TYPE_REGISTRY` | `proposal-registry.ts` | Proposal-registry unit test suite |
| `proposalPanelForType` switch | `proposal-panels.ts` | Proposal panel browser journey tests |
| `workflowStore.put()` | `workflow-store.ts` | Goal-creation flow integration tests exercise workflow merge |
| `deleteProposalFile` | `proposal-files.ts` | Generic over ProposalType — exercised by every proposal type's dismiss path |
| `proposalPanelTabId` | `panel-workspace.ts` | Browser journey tests exercising proposal panel tab lifecycle |

**Defect surface**: No new abstractions. Every addition composes existing tested symbols. The ~80-line accept route is the only net-new logic (see §3.4 defect surface inventory).

### 2.2 Approach B — Rich panel with interactive editing (rejected)

Use `renderWorkflowEditor` in the proposal panel for interactive gate editing,
side-by-side diff when editing an existing workflow, and real-time inline
validation.

**Why rejected:**

1. **`renderWorkflowEditor` is not designed for proposal panels.** It uses a
   global `embedInstances` map keyed by `workflow.id` — introducing it into a
   second call site (the proposal panel) creates a shared mutable state that
   competes with the Settings → Workflows page. The inspector is read-only and
   safe to co-render; the editor is not.

2. **The goal spec says "shows gate DAG" and "accept/reject/edit affordances"**
   — the inspector satisfies this. Interactive editing of gate definitions is
   already handled by `edit_proposal("workflow", ...)` which lets the agent
   surgically revise the YAML draft. Adding a third editing surface (agent tool
   edit, Settings page, proposal panel) creates confusion about which is
   authoritative.

3. **~200 additional lines** in `proposal-panels.ts` for a UI that duplicates
   existing functionality without adding new capability. The diff view is
   especially costly — it requires loading the current project workflow,
   diffing YAML, and rendering a side-by-side comparison.

4. **Risk of editor state corruption.** The embed instance cache
   (`workflow-page.ts` `embedInstances`) is not designed for concurrent use
   from multiple panels. A proposal panel that opens an editor instance for
   workflow `foo` while the Settings page also has `foo` open would corrupt
   both.

**Decision**: Approach A. Composes existing well-tested code, adds no new
abstractions, satisfies all requirements, and avoids the architectural risks
of Approach B.

## 3. Implementation Plan

### 3.1 Server — `proposal-files.ts`

**File**: `src/server/proposals/proposal-files.ts`

Add `"workflow"` to the `ProposalType` union (line 18) and `PROPOSAL_TYPES`
array (line 104):

```typescript
// Before:
export type ProposalType = "goal" | "project" | "role" | "tool" | "staff";
export const PROPOSAL_TYPES: readonly ProposalType[] = [
	"goal", "project", "role", "tool", "staff",
] as const;

// After:
export type ProposalType = "goal" | "project" | "workflow" | "role" | "tool" | "staff";
export const PROPOSAL_TYPES: readonly ProposalType[] = [
	"goal", "project", "workflow", "role", "tool", "staff",
] as const;
```

### 3.2 Server — `proposal-types.ts`

**File**: `src/server/proposals/proposal-types.ts`

Add a `workflowPlugin` to the `REGISTRY` object (bottom of file):

```typescript
const workflowPlugin = makeYamlPlugin({
	type: "workflow",
	requiredFields: ["id", "name", "gates"],
});

const REGISTRY: Record<ProposalType, ProposalTypePlugin> = {
	goal: goalPlugin,
	project: projectPlugin,
	workflow: workflowPlugin,
	role: rolePlugin,
	tool: toolPlugin,
	staff: staffPlugin,
};
```

No need to extract `validateGoalInlineWorkflow` — the existing function is
already general-purpose (`unknown → ParseError | null`) and can be called
directly from the accept route handler.

### 3.3 Server — `server.ts` — seed route guard

**File**: `src/server/server.ts`

At the seed handler (around line 17780-17810), wrap the goal-specific
enrichment block to skip "workflow":

```typescript
// Existing code branches on `if (proposalType === "goal")` for enrichment
// and `if (proposalType === "goal" || proposalType === "staff" || ...)`
// for projectId resolution. Add "workflow" to the projectId resolution
// guard so workflow proposals get a default projectId stamped.
if (proposalType === "goal" || proposalType === "staff" || proposalType === "role" || proposalType === "tool") {
	// existing enrichment...
}
// "workflow" falls through to the generic write path.
```

Actually, "workflow" proposals need projectId resolution (they target a
specific project). Add `"workflow"` to the conditional:

```typescript
if (proposalType === "goal" || proposalType === "staff" || proposalType === "role" || proposalType === "tool" || proposalType === "workflow") {
```

### 3.4 Server — `server.ts` — accept route

**File**: `src/server/server.ts`

New route: `POST /api/sessions/:id/proposal/workflow/accept` (~80 lines).

**Defect surface inventory**:

- **State ownership**: Owns temporary parse result, gate validation error, project context handle, and response. No persistent state beyond `workflowStore.put()` side effect.
- **Transformations** (6 ordered steps):
  1. `parseProposalFile(stateDir, sessionId, "workflow")` → `ParseResult`
  2. Extract `{id, name, description, gates}` from `draft.value.fields`
  3. `validateGoalInlineWorkflow({id, name, gates})` → gate schema validation
  4. Resolve `targetProjectId` from body.projectId → fields.projectId → session.projectId
  5. `ctx.workflowStore.put({id, name, description, gates, createdAt, updatedAt})`
  6. `deleteProposalFile()` + broadcast `proposal_cleared`
- **Failure modes per step**:
  1. FILE_NOT_FOUND → 404; YAML parse error → 400
  2. Missing id → 400 MISSING_REQUIRED_FIELD
  3. Invalid gate schema → 400 STRUCTURAL_VALIDATION_FAILED
  4. No project resolved → 400 PROJECT_ID_REQUIRED
  5. Unknown project → 404 UNKNOWN_PROJECT
  6. Cleanup failure → logged warning, non-fatal (200 still returned)
  7. Unexpected error → 500
- **Merge semantics**: On update (existing workflow with same ID), the workflow is fully replaced with new `name`, `description`, and `gates` values. `createdAt` is set to current timestamp on both create and update (simplifies implementation). No other workflow entries in the project are touched.
- **Edge cases**: Zero-gate workflows are valid (gate validator accepts empty arrays). Concurrent proposals targeting the same project are last-write-wins — the workflow store is per-project, single-process, no distributed race.
- **Constraints**: No file locks across the parse→merge window. `workflowStore.put()` is a synchronous memory operation followed by async persist.

### 3.5 Tool extension — `extension.ts`

**File**: `defaults/tools/proposals/extension.ts`

Add after `propose_staff` registration:

```typescript
// ── propose_workflow ──────────────────────────────────────────────
pi.registerTool({
	name: "propose_workflow",
	label: "Propose Workflow",
	description: "Submit a workflow proposal for user review.",
	promptSnippet: "Propose a workflow with id, name, description, and gates.",
	parameters: Type.Object({
		id: Type.String({ description: "Workflow identifier (kebab-case)." }),
		name: Type.String({ description: "Display name." }),
		description: Type.Optional(Type.String({ description: "Markdown description." })),
		gates: Type.Array(Type.Any(), { description: "Gate definitions (same schema as inline workflows)." }),
		projectId: Type.Optional(Type.String({ description: "Defaults to current session's project; explicit for cross-project." })),
	}),
	async execute(_id, args) { const r = await seedProposal("workflow", args); return ack(r.rev); },
});
```

Note: `seedProposal` already handles `argsWithProjectId("workflow", args)` to
stamp the default projectId on the draft.

### 3.6 UI — Tool renderer registry

**File**: `src/ui/tools/index.ts` (line ~166)

Add `"propose_workflow"` to `PROPOSAL_TOOL_NAMES`:

```typescript
const PROPOSAL_TOOL_NAMES = [
	"propose_goal", "propose_role", "propose_tool",
	"propose_staff", "propose_workflow", "propose_project",
] as const;
```

**File**: `src/ui/tools/renderers/ProposalRenderer.ts` (line ~30)

Add the label mapping:

```typescript
propose_workflow: { label: "Workflow Proposal", type: "workflow", titleField: "name", previewField: "description" },
```

### 3.7 UI — Proposal registry

**File**: `src/app/proposal-registry.ts`

Add `"workflow"` to `ProposalType` union and `PROPOSAL_TYPES` array. Add a
`workflowPlugin`:

```typescript
export type ProposalType = "goal" | "project" | "workflow" | "role" | "tool" | "staff";
export const PROPOSAL_TYPES: readonly ProposalType[] = ["goal", "project", "workflow", "role", "tool", "staff"];

const PROPOSAL_TAB_LABELS: Record<ProposalType, string> = {
	goal: "Goal Proposal",
	project: "Project Proposal",
	workflow: "Workflow Proposal",
	role: "Role Proposal",
	tool: "Tool Proposal",
	staff: "Staff Proposal",
};

function workflowValidate(fields: Record<string, unknown>): string[] {
	return requireKeys(fields, ["id", "name", "gates"]);
}

export const PROPOSAL_TYPE_REGISTRY: Record<ProposalType, ProposalTypePlugin> = {
	// ... existing entries ...
	workflow: makePlugin("workflow", { mergeFields: defaultMerge, onFirstEmit: proposalFirstEmit("workflow"), validate: workflowValidate }),
};
```

### 3.8 UI — Proposal panel

**File**: `src/app/proposal-panels.ts`

Add `case "workflow"` to `proposalPanelForType` (line ~3785):

```typescript
case "workflow": return workflowProposalPanel();
```

Implement `workflowProposalPanel()` (~50-60 lines). Pattern follows
`toolPreviewPanel`:

1. Read `state.activeProposals.workflow?.fields`
2. Build a `Workflow` object from `fields.id`, `fields.name`, `fields.description`, `fields.gates`
3. Render `renderWorkflowInspector({ workflow, scope: "proposal" })` for the gate DAG
4. Footer with dismiss + accept buttons
5. Accept handler: POST to `/api/sessions/:id/proposal/workflow/accept`, then clean up

### 3.9 Session manager — proposal helpers

**File**: `src/app/session-manager.ts`

Add `saveWorkflowDraft` / `deleteWorkflowDraft` helpers (follow the pattern of
`saveRoleDraft` / `deleteRoleDraft`). Also add workflow dismissal tracking.

**File**: `src/app/proposal-helpers.ts`

Add `deleteProposalFile` support for `"workflow"` type if not already generic.

### 3.10 Panel workspace

**File**: `src/app/panel-workspace.ts`

`proposalPanelTabId` already accepts any `ProposalType` string — since
`ProposalType` now includes `"workflow"`, the tab ID generation works
automatically.

## 4. Files Changed

| File | Change | Est. lines |
|---|---|---|
| `src/server/proposals/proposal-files.ts` | Add `"workflow"` to union + array | 2 |
| `src/server/proposals/proposal-types.ts` | Add `workflowPlugin` + registry entry | 15 |
| `src/server/server.ts` | Seed guard + accept route | 80 |
| `defaults/tools/proposals/extension.ts` | Register `propose_workflow` tool | 40 |
| `src/ui/tools/index.ts` | Add to `PROPOSAL_TOOL_NAMES` | 2 |
| `src/ui/tools/renderers/ProposalRenderer.ts` | Add label mapping | 2 |
| `src/app/proposal-registry.ts` | Add workflow to type + plugin + labels | 25 |
| `src/app/proposal-panels.ts` | Add `workflowProposalPanel` + switch case | 60 |
| `src/app/session-manager.ts` | Add workflow draft helpers | 15 |
| `src/app/proposal-helpers.ts` | Ensure generic workflow support | 5 |
| **Total** | | **~250** |

**Implementation status**: The code on branch `goal/propose-workfl-ffaa495f` contains the full implementation described in this design doc (14 files, ~600 lines). All server, web, and release type checks pass (`npm run check`). This design document serves as the architecture record.

## 5. Acceptance Criteria

1. **`propose_workflow` tool** is callable from agent sessions. It seeds a
   `workflow.yaml` proposal draft.
2. **`view_proposal("workflow")`** returns the YAML draft (no longer 400).
3. **`edit_proposal("workflow", ...)`** works identically to other types.
4. **Accept route** validates gate schema and merges a single workflow into
   `project.yaml::workflows` without touching other workflows.
5. **Proposal panel** renders in the side panel with gate DAG (via
   `renderWorkflowInspector`), accept/reject/dismiss buttons.
6. **Cross-project** workflow proposals work when `projectId` is explicit.
7. **Reject** cleans up the proposal draft and dismisses the panel tab.
8. Existing proposal types (goal, project, role, tool, staff) are unaffected.

## 6. E2E Test Plan

### Test 1: Happy Path — Create and Accept Workflow
1. Agent calls `propose_workflow` with id="test-workflow", name="Test", gates=[{id:"design",...}]
2. Verify `workflow.yaml` draft created in session proposals directory
3. User opens proposal panel, sees gate DAG rendered via `renderWorkflowInspector`
4. User clicks Accept
5. Verify `project.yaml` updated with workflows.test-workflow entry
6. Verify proposal draft deleted, panel dismissed

### Test 2: Edit Draft Before Accept
1. Seed workflow proposal via `propose_workflow`
2. Agent calls `edit_proposal("workflow", ...)` to change name
3. Verify YAML draft reflects edit
4. Accept and verify merged name is edited value

### Test 3: Cross-Project Proposal
1. Agent calls `propose_workflow` with explicit projectId="other-project"
2. Verify draft targets correct project
3. Accept and verify other-project's `project.yaml` updated

### Test 4: Gate Schema Validation Failure
1. Agent proposes workflow with invalid gates (missing required field)
2. Accept route returns 400 with validation error
3. Project config unchanged

### Test 5: Update Existing Workflow
1. Project has existing workflow "existing-id"
2. Agent proposes workflow with same id, different name/gates
3. Accept merges and updates name/gates, preserves other workflows

### Test 6: view_proposal and Edit Flow
1. Agent seeds workflow proposal
2. Agent calls `view_proposal("workflow")` — returns YAML draft (no 400)
3. Agent calls `edit_proposal("workflow", old, new)` — exact-text replacement works
4. Panel shows updated draft in real-time

## 7. Error Handling

Server accept route HTTP status codes:
- `200` — Workflow merged into project.yaml, proposal draft deleted
- `400` — Invalid gate schema (validation via `validateGoalInlineWorkflow` fails)
- `404` — Proposal draft not found
- `404` — Target project not found (if explicit projectId)
- `500` — File system errors during project.yaml read/write

## 8. Existing Generic Infrastructure

- `proposal-helpers.ts::deleteProposalFile()` is already generic over all `ProposalType` values — adding "workflow" to the union makes it work automatically.
- `panel-workspace.ts::proposalPanelTabId()` already accepts any `ProposalType` string.
- No new session-manager draft helpers are needed beyond the type registration.

## 9. Out of Scope

- Deleting workflows via proposal
- Bulk workflow proposals
- Workflow versioning or migration tooling
- Interactive gate editing in the proposal panel (use `edit_proposal` instead)