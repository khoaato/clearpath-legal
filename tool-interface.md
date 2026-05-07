# ClearPath Tool Interface Guide

Tools are integrations that gather real-world information for users during the goal-setting and task-planning stages. This document explains how to create, register, and wire up a tool.

---

## What Is a Tool?

A tool is a named capability stored in the `tool_definitions` table. It can be:

- **Goal-gathering tool** — runs during an AI session on a profile's Goals page to surface relevant data before the user sets goals (e.g. pulling a credit score before setting finance goals).
- **Task tool** — matched to a specific task and run to complete or assist with that task.

Tools are surfaced to the user via AI suggestions during a session, or matched to tasks automatically by keyword.

---

## `tool_definitions` Schema

| Field | Type | Description |
|---|---|---|
| `name` | `string` | Human-readable name shown in the UI (e.g. `"Net Worth Snapshot"`) |
| `description` | `string` | One-sentence description shown to the user and the AI |
| `domain` | `string` | Comma-separated keywords for matching (e.g. `"finance,savings,net worth"`) |
| `inputSchema` | `any` | JSON Schema object describing what input the tool needs (use `{}` if none) |
| `outputSchema` | `any` | JSON Schema object describing the output shape — also used as mock data for placeholder tools |
| `convexAction` | `string?` | Name of a Convex action to run (e.g. `"toolActions:fetchCreditScore"`). Not yet wired; reserved for real implementations. |
| `externalEndpoint` | `string?` | HTTP endpoint to call. Not yet wired; reserved for real implementations. |
| `applicableStages` | `string[]?` | Which stages this tool applies to. Values: `"goal_gathering"`, `"task"`. Omit to apply to no stages automatically. |

---

## `applicableStages` Values

| Value | Where it surfaces |
|---|---|
| `"goal_gathering"` | AI session on the profile Goals page — AI can suggest this tool mid-conversation |
| `"task"` | Task detail page — matched by keyword to a task's title/notes |
| Both | Tool appears in both contexts |

---

## Execution Lifecycle

Tool runs are tracked in the `profile_tool_runs` table (for goal_gathering tools):

```
pending → running → completed | failed
```

When `runForProfile` is called:
1. A `profile_tool_runs` row is inserted with `status: "running"`
2. The tool's output is computed (currently: mock data from `outputSchema`)
3. The row is patched to `status: "completed"` with `outputData`
4. The output is returned to the client

When a real `convexAction` is wired, the action will be called between steps 2 and 3.

---

## How Tool Output Surfaces

### Goal-gathering flow

1. AI suggests tools via a `<tool_suggestions>` block in its response
2. Tool suggestion cards appear below the chat in the AI session UI
3. User clicks **Run** — `runForProfile` is called
4. Output is injected as a user message: `"Tool result — [Tool Name]: { ... }"`
5. AI receives this in context and incorporates the data when asking follow-up questions or proposing goals

### Task flow (task tool assignments)

Tool output is stored in `task_tool_assignments.outputData` and shown on the task detail page.

---

## AI Tool Suggestion Format

During a `goal_gathering` session, the AI may include this block in any message:

```
<tool_suggestions>
["Net Worth Snapshot", "Budget Analyzer"]
</tool_suggestions>
```

The block is stripped from the displayed message. Suggested tool names must exactly match entries in `tool_definitions.name`.

The AI receives the list of available goal_gathering tools in its system prompt, so it knows which names are valid.

---

## How to Register a New Tool

### Option 1: Add to `seedGoalTools` (recommended for built-in tools)

In `convex/tools.ts`, add your tool to the `goalTools` array inside `seedGoalTools`:

```typescript
{
  name: "Credit Score Lookup",
  description: "Fetch an estimated credit score and summary.",
  domain: "finance,credit,debt,loan,credit score",
  inputSchema: {},
  outputSchema: { score: 720, rating: "Good", summary: "No missed payments in 24 months" },
  applicableStages: ["goal_gathering"],
},
```

`seedGoalTools` checks by name and only inserts tools that don't already exist.

### Option 2: Insert directly

Call `ctx.db.insert("tool_definitions", { ... })` in any mutation with the fields above.

---

## Wiring a Real Implementation

To replace the mock output with a real data source:

1. Set `convexAction: "toolActions:myToolAction"` on the tool definition
2. Create `convex/toolActions.ts` (with `"use node"`) containing your action:

```typescript
"use node";
import { internalAction } from "./_generated/server";
import { v } from "convex/values";

export const myToolAction = internalAction({
  args: { profileId: v.id("profiles"), userId: v.id("users") },
  handler: async (ctx, args) => {
    // Call external API, fetch data, etc.
    return { score: 720, summary: "..." };
  },
});
```

3. Update `runForProfile` in `convex/tools.ts` to call `ctx.runAction(internal.toolActions.myToolAction, ...)` instead of using `outputSchema` as mock data.

---

## Example Tool Definition

```typescript
{
  name: "Bank Balance Summary",
  description: "Pull your current checking and savings balances.",
  domain: "finance,bank,balance,checking,savings,money",
  inputSchema: {},
  outputSchema: {
    checking: "$2,400",
    savings: "$8,100",
    lastUpdated: "2026-03-01",
  },
  convexAction: "toolActions:fetchBankBalance",
  applicableStages: ["goal_gathering"],
}
```
