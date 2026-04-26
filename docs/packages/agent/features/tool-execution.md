# Tool Execution

This document covers everything about how `pi-agent-core` defines, validates, and executes tools. Read [agent-loop.md](./agent-loop.md) first for the broader context.

---

## Defining a tool

A tool is an object that implements `AgentTool<TParameters, TDetails>`:

```typescript
import { Type } from "@mariozechner/pi-ai";   // re-exports TypeBox Type
import type { AgentTool } from "@mariozechner/pi-agent-core";

const readFileTool: AgentTool<typeof schema, { path: string; bytes: number }> = {
  // Required fields from pi-ai's Tool interface
  name: "read_file",
  description: "Read a file from disk and return its contents.",
  parameters: Type.Object({
    path: Type.String({ description: "Absolute or relative file path." }),
    encoding: Type.Optional(Type.String({ default: "utf-8" })),
  }),

  // Additional fields from AgentTool
  label: "Read File",          // human-readable label shown in UI
  execute: async (toolCallId, params, signal, onUpdate) => {
    const content = await fs.readFile(params.path, params.encoding ?? "utf-8");
    return {
      content: [{ type: "text", text: content }],
      details: { path: params.path, bytes: content.length },
    };
  },
};
```

### TypeBox schema

The `parameters` field is a [TypeBox](https://github.com/sinclairzx81/typebox) schema. TypeBox generates JSON Schema-compatible schemas and static TypeScript types simultaneously:

```typescript
import { Type, Static } from "@mariozechner/pi-ai";

// Define the schema
const schema = Type.Object({
  city: Type.String({ description: "City name." }),
  units: Type.Union([Type.Literal("celsius"), Type.Literal("fahrenheit")], { default: "celsius" }),
});

// The inferred type: { city: string; units: "celsius" | "fahrenheit" }
type Params = Static<typeof schema>;
```

At runtime, the loop calls `validateToolArguments(tool, toolCall)` (from `pi-ai`) before executing the tool. This validates the LLM-supplied JSON arguments against the schema. Validation errors produce an error tool result without calling `execute`.

### `prepareArguments` shim

If the LLM sends arguments in a format that does not quite match the schema — for example, a legacy provider that omits optional fields — you can normalise them before validation:

```typescript
const myTool: AgentTool = {
  // ...
  prepareArguments: (rawArgs: unknown) => {
    const args = rawArgs as Record<string, unknown>;
    return {
      city: String(args.city ?? ""),
      units: args.units ?? "celsius",
    };
  },
};
```

`prepareArguments` runs before schema validation. If it returns the same object reference, the shim step is skipped.

### `execute` contract

```typescript
execute: (
  toolCallId: string,
  params: Static<TParameters>,   // validated, type-safe parameters
  signal?: AbortSignal,           // agent's abort signal
  onUpdate?: AgentToolUpdateCallback<TDetails>,
) => Promise<AgentToolResult<TDetails>>
```

Rules:
- **Throw on failure.** The loop catches throws and converts them to an error `ToolResultMessage` sent back to the LLM. Do not encode errors in `content` manually.
- **Honour the signal.** Check `signal?.aborted` or pass the signal to child processes / fetch calls. When aborted, throw or let the signal cancel the operation.
- **Call `onUpdate` for progress.** Long-running tools should call `onUpdate` with a partial result to emit `tool_execution_update` events for real-time UI updates.

```typescript
execute: async (toolCallId, params, signal, onUpdate) => {
  let downloaded = 0;
  for await (const chunk of downloadFile(params.url, signal)) {
    downloaded += chunk.length;
    onUpdate?.({
      content: [{ type: "text", text: `Downloaded ${downloaded} bytes…` }],
      details: { downloaded },
    });
  }
  return {
    content: [{ type: "text", text: `Download complete: ${downloaded} bytes` }],
    details: { downloaded },
  };
},
```

### `AgentToolResult`

```typescript
interface AgentToolResult<T> {
  content: (TextContent | ImageContent)[];  // returned to the LLM
  details: T;                                // for UI / logs, never sent to LLM
  terminate?: boolean;                       // hint to stop after this batch
}
```

- `content` must be non-empty. The LLM reads it to decide what to do next.
- `details` is arbitrary structured data. The UI can render it as a card, diff, file preview, etc.
- `terminate` signals that the loop should stop after this tool batch. See [early termination](#early-termination).

---

## Parallel vs sequential execution

When the LLM requests multiple tool calls in a single assistant message, the loop chooses between two execution modes.

### Parallel (default)

```typescript
const agent = new Agent({ toolExecution: "parallel" }); // default
```

In parallel mode:

1. All tool calls are **prepared** sequentially: `prepareArguments` → validate → `beforeToolCall`.
2. Prepared tool calls are launched concurrently via `Promise.all`.
3. `tool_execution_end` events are emitted as each tool completes (completion order).
4. `ToolResultMessage` events are emitted in **assistant source order** (the order the tool calls appeared in the assistant message). This ensures deterministic message IDs.

```mermaid
sequenceDiagram
  participant Loop
  participant ToolA
  participant ToolB
  Loop->>ToolA: tool_execution_start
  Loop->>ToolB: tool_execution_start
  Loop->>ToolA: execute() [concurrent]
  Loop->>ToolB: execute() [concurrent]
  ToolB-->>Loop: result (finishes first)
  Loop->>Loop: tool_execution_end (B)
  ToolA-->>Loop: result
  Loop->>Loop: tool_execution_end (A)
  Note over Loop: Emit ToolResultMessages in source order
  Loop->>Loop: message_start/end (A result)
  Loop->>Loop: message_start/end (B result)
```

### Sequential

```typescript
const agent = new Agent({ toolExecution: "sequential" });
```

Or per tool:

```typescript
const safeTool: AgentTool = {
  executionMode: "sequential",
  // ...
};
```

If any tool in a batch has `executionMode: "sequential"`, the entire batch runs sequentially regardless of the global setting.

In sequential mode, each tool is fully prepared, executed, and finalized before the next one starts. `tool_execution_start` / `tool_execution_end` / `message_start` / `message_end` for each tool all complete before the next tool begins.

### When to use sequential

Use `"sequential"` for tools that:
- Modify shared state (e.g., writing to the same file).
- Have ordering constraints (tool B needs the output of tool A).
- Must not be interrupted mid-way (e.g., database transactions).

---

## `beforeToolCall` hook

Called after argument validation, before `execute`. Return `{ block: true }` to prevent execution:

```typescript
const agent = new Agent({
  beforeToolCall: async ({ toolCall, args, assistantMessage, context }, signal) => {
    // Block dangerous commands
    if (toolCall.name === "bash" && (args as any).command?.includes("rm -rf")) {
      return { block: true, reason: "Destructive commands are not allowed." };
    }

    // Log every tool invocation
    await auditLog.write({ tool: toolCall.name, args, timestamp: Date.now() });

    // Return undefined (or nothing) to allow execution
  },
});
```

When blocked, the loop emits `tool_execution_end` with `isError: true` and a tool result containing the `reason` text. The LLM sees the blocked result and can try a different approach.

Context available in `BeforeToolCallContext`:
- `assistantMessage` — the full assistant message that requested this tool call.
- `toolCall` — the raw `AgentToolCall` block (id, name, arguments).
- `args` — validated, type-safe arguments (after `prepareArguments`).
- `context` — current `AgentContext` snapshot (system prompt, messages, tools).

---

## `afterToolCall` hook

Called after `execute` succeeds or fails, before `tool_execution_end` is emitted. Return an `AfterToolCallResult` to override parts of the result:

```typescript
const agent = new Agent({
  afterToolCall: async ({ toolCall, result, isError, context }, signal) => {
    if (isError) {
      // Log errors but don't change the result
      await errorLog.write({ tool: toolCall.name, error: result.content });
      return; // undefined → keep original
    }

    // Annotate details with metadata for the UI
    return {
      details: { ...result.details, processedAt: Date.now() },
    };
  },
});
```

`AfterToolCallResult` uses merge semantics:

| Field | Behaviour |
|---|---|
| `content` | Replaces the entire content array. No deep merge. |
| `details` | Replaces the entire details value. No deep merge. |
| `isError` | Replaces the error flag. |
| `terminate` | Replaces the early-termination hint. |

Omitting a field keeps the original value. If `afterToolCall` throws, the exception is caught, the result is replaced with an error result, and the loop continues.

Context available in `AfterToolCallContext`:
- Everything in `BeforeToolCallContext`, plus:
- `result` — the executed `AgentToolResult` before overrides.
- `isError` — whether the execution produced an error.

---

## Early termination

A tool can signal that the loop should stop after the current batch by setting `terminate: true` on its result:

```typescript
execute: async (toolCallId, params, signal) => {
  await finalizeReport(params);
  return {
    content: [{ type: "text", text: "Report finalized. Session complete." }],
    details: {},
    terminate: true,   // ← stop the loop after this batch
  };
},
```

Early termination only happens when **every** tool result in the batch has `terminate: true`. If even one tool returns `terminate: false` (or omits it), the loop continues normally.

The `afterToolCall` hook can also set or override `terminate` on any result, enabling external orchestration to stop the loop based on policy rather than tool logic.

---

## Image tool results

Tools can return images alongside text by including `ImageContent` in `content`:

```typescript
return {
  content: [
    { type: "text", text: "Here is the rendered chart:" },
    { type: "image", mediaType: "image/png", data: base64PngData },
  ],
  details: { chartType: "bar" },
};
```

Only multimodal LLMs that accept images in tool results will use this correctly. Check `model.supportsImages` before relying on image content.
