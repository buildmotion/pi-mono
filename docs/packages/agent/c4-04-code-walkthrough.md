# C4 Level 4 — Code Walkthrough

This document traces a complete call through the stack, from `agent.prompt("What is the weather in Berlin?")` all the way to `agent_end`, including a tool call. Read this after the [C4 Container view](./c4-02-container.md) and [C4 Component view](./c4-03-component.md).

The example assumes:
- A `getWeather` tool is registered on the agent.
- The model decides to call it once, then answers.

---

## Step 0: Setup

```typescript
import { Agent } from "@mariozechner/pi-agent-core";
import { getModel, Type } from "@mariozechner/pi-ai";

const agent = new Agent({
  initialState: {
    systemPrompt: "You are a helpful assistant.",
    model: getModel("anthropic", "claude-haiku-4-5"),
    thinkingLevel: "off",
    tools: [
      {
        name: "get_weather",
        label: "Get Weather",
        description: "Returns the current weather for a city.",
        parameters: Type.Object({ city: Type.String() }),
        execute: async (toolCallId, params, signal) => {
          const temp = await fetchWeatherApi(params.city); // your implementation
          return {
            content: [{ type: "text", text: `${params.city}: ${temp}°C` }],
            details: { city: params.city, temp },
          };
        },
      },
    ],
    messages: [],
  },
});

// Subscribe to print text as it streams
const unsubscribe = agent.subscribe((event) => {
  if (event.type === "message_update") {
    const e = event.assistantMessageEvent;
    if (e.type === "text_delta") process.stdout.write(e.delta);
  }
});
```

The `Agent` constructor stores options in `_state` (a `MutableAgentState`) and creates two `PendingMessageQueue` instances (steering and follow-up). The `streamFn` defaults to `streamSimple` from `pi-ai`.

---

## Step 1: `agent.prompt("What is the weather in Berlin?")`

```typescript
// agent.ts — Agent.prompt()
async prompt(input: string | AgentMessage | AgentMessage[], images?: ImageContent[]): Promise<void> {
  if (this.activeRun) {
    throw new Error("Agent is already processing a prompt.");
  }
  const messages = this.normalizePromptInput(input, images);
  // messages = [{ role: "user", content: [{ type: "text", text: "What is the weather in Berlin?" }], timestamp: ... }]
  await this.runPromptMessages(messages);
}
```

`normalizePromptInput` converts a plain string into a `UserMessage`. Then `runPromptMessages` is called.

---

## Step 2: `runPromptMessages` → `runWithLifecycle`

```typescript
// agent.ts — Agent.runPromptMessages()
private async runPromptMessages(messages: AgentMessage[]): Promise<void> {
  await this.runWithLifecycle(async (signal) => {
    await runAgentLoop(
      messages,                       // new user message(s)
      this.createContextSnapshot(),   // snapshot of current state.messages + tools
      this.createLoopConfig(),        // AgentLoopConfig with all callbacks wired
      (event) => this.processEvents(event),
      signal,
      this.streamFn,
    );
  });
}
```

`runWithLifecycle` creates an `AbortController`, sets `_state.isStreaming = true`, and stores the active run. The `emit` callback passed to `runAgentLoop` is `(event) => this.processEvents(event)` — this is how the loop feeds events back into `Agent` for state mutation and listener dispatch.

`createLoopConfig` builds the `AgentLoopConfig`, wiring:
- `getSteeringMessages` → drains the steering queue.
- `getFollowUpMessages` → drains the follow-up queue.
- All hooks (`beforeToolCall`, `afterToolCall`, `convertToLlm`, `transformContext`, `getApiKey`) from public fields.

---

## Step 3: `runAgentLoop` starts

```typescript
// agent-loop.ts — runAgentLoop()
export async function runAgentLoop(
  prompts: AgentMessage[],
  context: AgentContext,
  config: AgentLoopConfig,
  emit: AgentEventSink,
  signal?: AbortSignal,
  streamFn?: StreamFn,
): Promise<AgentMessage[]> {
  const newMessages: AgentMessage[] = [...prompts];
  const currentContext: AgentContext = {
    ...context,
    messages: [...context.messages, ...prompts],  // user message appended
  };

  await emit({ type: "agent_start" });   // ← subscriber sees this
  await emit({ type: "turn_start" });    // ← subscriber sees this

  // Emit message_start + message_end for each prompt
  for (const prompt of prompts) {
    await emit({ type: "message_start", message: prompt });
    await emit({ type: "message_end", message: prompt });
  }

  await runLoop(currentContext, newMessages, config, signal, emit, streamFn);
  return newMessages;
}
```

**State after step 3:**
- `agent.state.messages` now includes the user message (pushed by `processEvents` on `message_end`).
- `agent.state.isStreaming === true`.

---

## Step 4: `runLoop` — first iteration, LLM call

```typescript
// agent-loop.ts — runLoop() (simplified)
async function runLoop(currentContext, newMessages, config, signal, emit, streamFn) {
  let pendingMessages = (await config.getSteeringMessages?.()) || []; // empty at start

  while (true) {                     // outer loop (follow-up messages)
    let hasMoreToolCalls = true;

    while (hasMoreToolCalls || pendingMessages.length > 0) {   // inner loop
      // ... emit turn_start for subsequent turns ...

      // LLM call
      const message = await streamAssistantResponse(currentContext, config, signal, emit, streamFn);
      // message = AssistantMessage with stopReason: "toolUse" and toolCall: get_weather
      newMessages.push(message);

      const toolCalls = message.content.filter((c) => c.type === "toolCall");
      // toolCalls = [{ type: "toolCall", id: "tc_01", name: "get_weather", arguments: { city: "Berlin" } }]
      // ...
    }
    // ...
  }
}
```

`streamAssistantResponse` does the heavy lifting:

```typescript
// agent-loop.ts — streamAssistantResponse() (simplified)
async function streamAssistantResponse(context, config, signal, emit, streamFn) {
  // 1. Optional context transform
  let messages = context.messages;
  if (config.transformContext) {
    messages = await config.transformContext(messages, signal);
  }

  // 2. Convert AgentMessage[] → Message[] for the LLM
  const llmMessages = await config.convertToLlm(messages);
  // Default: filters to role "user" | "assistant" | "toolResult"

  // 3. Build LLM context
  const llmContext = { systemPrompt: context.systemPrompt, messages: llmMessages, tools: context.tools };

  // 4. Resolve API key (important for expiring tokens)
  const resolvedApiKey = (config.getApiKey ? await config.getApiKey(config.model.provider) : undefined) || config.apiKey;

  // 5. Call streamFn (pi-ai)
  const response = await streamFn(config.model, llmContext, { ...config, apiKey: resolvedApiKey, signal });

  // 6. Drive the stream, emitting AgentEvents
  for await (const event of response) {
    switch (event.type) {
      case "start":
        await emit({ type: "message_start", message: { ...partialMessage } });
        break;
      case "text_delta":
        await emit({ type: "message_update", message: { ...partialMessage }, assistantMessageEvent: event });
        break;
      case "done":
        await emit({ type: "message_end", message: finalMessage });
        return finalMessage;
    }
  }
}
```

**Events emitted during the LLM call (first turn):**

```
message_start   { message: AssistantMessage (partial, empty) }
message_update  { assistantMessageEvent: { type: "text_delta", delta: "I'll" } }
message_update  { assistantMessageEvent: { type: "text_delta", delta: " check" } }
... (more text deltas) ...
message_update  { assistantMessageEvent: { type: "toolcall_start", ... } }
message_update  { assistantMessageEvent: { type: "toolcall_delta", delta: '{"city"' } }
message_update  { assistantMessageEvent: { type: "toolcall_end", ... } }
message_end     { message: AssistantMessage (final, stopReason: "toolUse") }
```

---

## Step 5: Tool execution

Back in `runLoop`, `stopReason === "toolUse"`, so `executeToolCalls` is called:

```typescript
// agent-loop.ts — executeToolCalls selects mode
async function executeToolCalls(currentContext, assistantMessage, config, signal, emit) {
  const hasSequentialTool = toolCalls.some(tc => currentContext.tools?.find(t => t.name === tc.name)?.executionMode === "sequential");
  if (config.toolExecution === "sequential" || hasSequentialTool) {
    return executeToolCallsSequential(...);
  }
  return executeToolCallsParallel(...);  // default
}
```

Since `get_weather` has no `executionMode` override and the agent defaults to `"parallel"`, parallel execution runs. For a single tool call, parallel and sequential are equivalent.

```typescript
// For each tool call (here, just get_weather):
await emit({ type: "tool_execution_start", toolCallId: "tc_01", toolName: "get_weather", args: { city: "Berlin" } });
// ↑ Agent sets pendingToolCalls.add("tc_01")

// prepareToolCall: find tool, prepareArguments (if any), validateToolArguments, beforeToolCall
// → returns PreparedToolCall { kind: "prepared", tool, args: { city: "Berlin" } }

// executePreparedToolCall: calls tool.execute()
const result = await tool.execute("tc_01", { city: "Berlin" }, signal, onUpdate);
// result = { content: [{ type: "text", text: "Berlin: 18°C" }], details: { city: "Berlin", temp: 18 } }

// finalizeExecutedToolCall: calls afterToolCall (if configured)
// → no override in this example, result passes through unchanged

await emit({ type: "tool_execution_end", toolCallId: "tc_01", toolName: "get_weather", result, isError: false });
// ↑ Agent sets pendingToolCalls.delete("tc_01")
```

Then a `ToolResultMessage` is built and emitted:

```
message_start  { message: ToolResultMessage { role: "toolResult", toolCallId: "tc_01", ... } }
message_end    { message: ToolResultMessage }
```

The tool result is appended to `currentContext.messages` and `newMessages`.

---

## Step 6: `turn_end` and steering poll

```typescript
await emit({ type: "turn_end", message: assistantMessage, toolResults: [toolResultMessage] });

// Poll for steering messages
pendingMessages = (await config.getSteeringMessages?.()) || [];
// → drains steering queue: empty in this example
```

Since there are tool results and no early termination, `hasMoreToolCalls` remains `true` and the inner loop continues.

---

## Step 7: Second LLM call — final response

The inner loop runs again. `turn_start` fires. `streamAssistantResponse` is called with the updated context (now containing the tool result). The model generates the final answer:

```
turn_start
message_start   (assistant — final answer)
message_update  { type: "text_delta", delta: "The current" }
message_update  { type: "text_delta", delta: " weather in Berlin is 18°C." }
message_end     (assistant — stopReason: "stop")
turn_end        (no tool results)
```

`hasMoreToolCalls` is now `false` and `pendingMessages` is empty, so the inner loop exits.

---

## Step 8: Follow-up poll and `agent_end`

```typescript
// outer loop in runLoop
const followUpMessages = (await config.getFollowUpMessages?.()) || [];
// → drains follow-up queue: empty in this example
// No follow-up messages → break

await emit({ type: "agent_end", messages: newMessages });
```

**State after step 8:**
- `agent.state.messages` contains: `[userMessage, assistantMessage(turn1), toolResultMessage, assistantMessage(turn2)]`.
- `agent.state.isStreaming` is still `true` — it remains true until all `agent_end` listeners settle.

---

## Step 9: Run settlement

After `agent_end` is emitted, `processEvents` in `Agent` awaits all listeners. Once they all resolve, `finishRun()` is called:

```typescript
// agent.ts — finishRun()
private finishRun(): void {
  this._state.isStreaming = false;
  this._state.streamingMessage = undefined;
  this._state.pendingToolCalls = new Set<string>();
  this.activeRun?.resolve();   // ← resolves waitForIdle()
  this.activeRun = undefined;
}
```

The `await agent.prompt(...)` call in the application code returns. `waitForIdle()` is now resolved.

---

## Summary: complete event sequence

```
agent_start
turn_start
  message_start    (user: "What is the weather in Berlin?")
  message_end      (user)
  message_start    (assistant — streams tool call request)
  message_update × N
  message_end      (assistant — stopReason: "toolUse")
  tool_execution_start  (get_weather, city: "Berlin")
  tool_execution_end    (result: "Berlin: 18°C", isError: false)
  message_start    (toolResult)
  message_end      (toolResult)
turn_end
turn_start
  message_start    (assistant — final answer)
  message_update × N
  message_end      (assistant — stopReason: "stop")
turn_end
agent_end
```
