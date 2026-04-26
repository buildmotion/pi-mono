# Terminology

Package-specific terms for `@mariozechner/pi-agent-core`. Terms that are already defined in the monorepo [glossary](../../glossary.md) are cross-linked rather than duplicated.

---

## Agent

An instance of the `Agent` class. Manages a single conversation session: holds `AgentState`, a list of event listeners, two internal `PendingMessageQueue` objects (steering and follow-up), and exposes the public API (`prompt`, `continue`, `abort`, `steer`, `followUp`, `subscribe`).

One `Agent` = one conversation = one `AgentState.messages` array.

---

## AgentEvent

The discriminated union of all events emitted during a run. Every field `type` value is unique across the union, so a `switch (event.type)` is a safe exhaustive check.

Full taxonomy: see [observability.md — Event taxonomy](./observability.md#event-taxonomy).

---

## AgentLoopConfig

The configuration object passed to the low-level loop functions (`runAgentLoop`, `agentLoop`, etc.). Contains model selection, tool list, hook callbacks, stream function, and execution strategy. Corresponds to the `AgentLoopConfig` interface in `types.ts`.

At the `Agent` level, most fields of `AgentLoopConfig` are derived from `AgentState` and `AgentOptions`.

---

## AgentMessage

The element type of the conversation history (`AgentState.messages`). Defined as:

```typescript
type AgentMessage = Message | CustomAgentMessages[keyof CustomAgentMessages];
```

When no custom messages are declared, `AgentMessage === Message` (the `pi-ai` message type). See [extension-points.md — CustomAgentMessages](./extension-points.md#customagentmessages--custom-message-types).

---

## AgentState

The mutable snapshot of an agent's conversation at any point in time. Contains:

- `messages` — the full conversation history
- `model` — the active LLM model
- `systemPrompt` — the current system prompt
- `thinkingLevel` — thinking/reasoning budget level
- `tools` — the active tool list
- `isStreaming` — true from first `agent_start` event until after the last `agent_end` listener settles
- `currentTurn` — 0-based turn counter within the current run

`AgentState` is the single source of truth. Do not mutate it directly; use the `Agent` API.

---

## AgentTool

The Command object that describes one tool. Contains `name`, `description`, `input_schema` (TypeBox), optional `prepareArguments`, and `execute`. See [tool-execution.md](./features/tool-execution.md) for the full contract.

---

## beforeToolCall / afterToolCall

Hook callbacks called before and after each tool execution respectively. `beforeToolCall` can block execution. `afterToolCall` can override result fields. Both receive a context object with the tool call, resolved arguments, and current conversation state. See [extension-points.md](./extension-points.md#beforetoolcall) and [tool-execution.md](./features/tool-execution.md).

---

## convertToLlm

A callback that converts `AgentMessage[]` (the full conversation history, potentially including custom message types) to the `Message[]` type accepted by `pi-ai`'s `streamSimple`. Called before every LLM call. See [extension-points.md](./extension-points.md#converttollm).

---

## early termination

A stop condition where ALL tools in a parallel or sequential batch return `{ terminate: true }` from their `execute` method. The agent emits `agent_end` with `stopReason: "terminated"` and does not call the LLM again, regardless of pending follow-up messages. See [tool-execution.md — early termination](./features/tool-execution.md#early-termination).

---

## emit (awaited vs fire-and-forget)

There are two variants of the loop:

- `runAgentLoop` / `runAgentLoopContinue` — used by the `Agent` class. `emit` callbacks are awaited before the loop advances. Listeners can safely do async I/O.
- `agentLoop` / `agentLoopContinue` — low-level. `emit` pushes events into an `EventStream` without awaiting listeners. Listeners receive events concurrently.

---

## follow-up

A message (or batch of messages) enqueued via `agent.followUp()`. Drained when the loop would otherwise stop but before emitting `agent_end`. Allows chaining of additional user messages without re-calling `prompt()`. See [steering-and-followup.md](./features/steering-and-followup.md).

---

## run

A single invocation lifecycle: from `agent.prompt()` or `agent.continue()` through to `agent_end`. One run may consist of many turns. A run ends when the LLM stops with no tool calls and no follow-up is queued, or when early termination fires, or when the run is aborted.

---

## steering

A message (or batch of messages) enqueued via `agent.steer()`. Injected into the conversation after each tool-execution batch during an active run. Steered messages appear in `AgentState.messages` and are sent to the LLM on the next turn. See [steering-and-followup.md](./features/steering-and-followup.md).

---

## streamFn

A strategy-pattern callback that replaces `streamSimple` as the LLM caller. If omitted, `streamSimple` from `pi-ai` is used. Use `streamProxy` for browser deployments or a mock for tests. See [extension-points.md](./extension-points.md#streamfn).

---

## tool execution mode

`"sequential"` (default) or `"parallel"`. Controls whether tools in a single LLM response are executed one at a time or concurrently. Set via `AgentLoopConfig.toolExecution`. See [tool-execution.md — parallel vs sequential](./features/tool-execution.md#parallel-vs-sequential-execution).

---

## transformContext

A callback that receives and returns `AgentMessage[]`. Called before `convertToLlm` on every turn. The correct place for context window management (pruning old messages, injecting external context). See [extension-points.md](./extension-points.md#transformcontext).

---

## turn

One outer-loop iteration: a single `streamAssistantResponse` call followed by (optionally) a tool execution batch. Bounded by `turn_start` and `turn_end` events. A run consists of one or more turns.

---

## See also

- [Monorepo glossary](../../glossary.md) — LLM, tool, model, message, system prompt, streaming, token
- [observability.md](./observability.md) — full event taxonomy
- [extension-points.md](./extension-points.md) — all hook reference
- [features/agent-loop.md](./features/agent-loop.md) — loop state machine
- [features/tool-execution.md](./features/tool-execution.md) — tool lifecycle
- [features/steering-and-followup.md](./features/steering-and-followup.md) — steering and follow-up
