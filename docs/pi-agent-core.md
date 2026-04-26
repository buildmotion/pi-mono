# `@mariozechner/pi-agent-core` — Stateful Agent Engine

## What is it?

`pi-agent-core` sits one layer above `pi-ai`. Where `pi-ai` handles a single LLM streaming call, `pi-agent-core` manages the full **agentic loop**: sending a message, waiting for the model to respond, executing any tools the model requested, feeding results back to the model, and repeating until the model stops asking for tools. It also provides a rich event stream so UIs can render progress in real time.

**npm package:** `@mariozechner/pi-agent-core`  
**Version:** 0.70.2  
**Node requirement:** >= 20.0.0  
**License:** MIT

---

## The Problem It Solves

A raw `pi-ai` stream gives you one LLM response. Real applications need:

- Conversation history that accumulates across turns
- Automatic tool-call → execute → result → LLM cycle
- UI-ready events emitted at each phase (so you can show typing indicators, tool progress bars, etc.)
- Ability to interrupt the agent mid-stream ("stop, do this instead")
- Queue follow-up messages without blocking

`pi-agent-core` provides all of this without tying you to any particular UI framework.

---

## Audience

- Developers building any AI-powered application: coding assistants, customer support bots, research tools
- Consumers of `pi-tui` or `pi-web-ui` who need to wire an agent to a chat interface
- Library authors building higher-level abstractions (e.g., `pi-coding-agent` and `pi-mom` both use this package)

---

## Core Concepts

### Agent

The `Agent` class is the main entry point. It holds all state and emits events.

```typescript
const agent = new Agent({
  initialState: {
    systemPrompt: 'You are a helpful assistant.',
    model: getModel('anthropic', 'claude-sonnet-4-20250514'),
    thinkingLevel: 'off',
    tools: [],
    messages: [],
  },
});
```

### AgentState

The agent exposes its mutable state via `agent.state`:

```typescript
interface AgentState {
  systemPrompt: string;
  model: Model<any>;
  thinkingLevel: ThinkingLevel;      // 'off' | 'minimal' | 'low' | 'medium' | 'high' | 'xhigh'
  tools: AgentTool<any>[];
  messages: AgentMessage[];
  readonly isStreaming: boolean;
  readonly streamingMessage?: AgentMessage;
  readonly pendingToolCalls: ReadonlySet<string>;
  readonly errorMessage?: string;
}
```

You can change the model, system prompt, or tools at any time between calls. Changes take effect on the next `prompt()`.

### AgentMessage vs LLM Message

The agent stores **AgentMessage** objects — a superset of what LLMs understand. You can attach custom message types (notifications, UI state, etc.) via TypeScript declaration merging. Before each LLM call, a `convertToLlm` function filters down to the standard LLM formats (`user`, `assistant`, `toolResult`).

```
AgentMessage[] → transformContext() → AgentMessage[] → convertToLlm() → Message[] → LLM
                    (optional)                           (required)
```

### Tools

Tools are defined as `AgentTool` objects with a TypeBox parameter schema and an `execute` async function:

```typescript
const myTool: AgentTool = {
  name: 'read_file',
  label: 'Read File',         // shown in UI
  description: 'Read a file',
  parameters: Type.Object({ path: Type.String() }),
  execute: async (toolCallId, params, signal, onUpdate) => {
    const content = await fs.readFile(params.path, 'utf-8');
    return { content: [{ type: 'text', text: content }], details: { path: params.path } };
  },
};
```

Throw an error from `execute` if the tool fails — the agent catches it and sends an error result to the LLM, which can then try a different approach.

---

## The Agentic Loop: Step by Step

Here is exactly what happens when you call `agent.prompt("Hello")`:

1. The user message is appended to `agent.state.messages`.
2. **`agent_start`** event fires.
3. A new turn begins: **`turn_start`** fires.
4. `transformContext` (if provided) can prune or inject messages.
5. `convertToLlm` filters messages to LLM-compatible format.
6. `pi-ai`'s `stream()` is called. **`message_start`** fires for the assistant message.
7. As LLM text arrives, **`message_update`** events fire (each carries the latest text delta).
8. When the LLM finishes, **`message_end`** fires for the assistant message.
9. If the model requested tools:
   - **`tool_execution_start`** fires for each tool call.
   - Tool's `execute()` runs (in parallel by default, or sequentially).
   - As tools run, they can emit **`tool_execution_update`** events for progress.
   - **`tool_execution_end`** fires when each tool finishes.
   - Tool result messages are appended to `agent.state.messages`.
   - **`turn_end`** fires with tool results.
   - Loop restarts at step 3 (next turn with tool results).
10. When the model responds without requesting tools:
    - **`turn_end`** fires.
    - Any pending follow-up messages are checked; if present, another turn runs.
    - Otherwise **`agent_end`** fires.

### Visual Sequence (no tools)

```
agent.prompt("Hello")
├─ agent_start
├─ turn_start
├─ message_start   (user message)
├─ message_end     (user message)
├─ message_start   (assistant starts responding)
├─ message_update  (chunk 1)
├─ message_update  (chunk 2)
├─ message_end     (assistant done)
├─ turn_end
└─ agent_end
```

### Visual Sequence (with tool call)

```
agent.prompt("Read config.json")
├─ agent_start
├─ turn_start
├─ message_start / message_end (user)
├─ message_start               (assistant — requests tool call)
├─ message_update...
├─ message_end
├─ tool_execution_start        (read_file called)
├─ tool_execution_end          (file content returned)
├─ turn_end
│
├─ turn_start                  (new turn with tool result in context)
├─ message_start               (assistant — final answer)
├─ message_update...
├─ message_end
├─ turn_end
└─ agent_end
```

---

## Parallel Tool Execution

By default, when the LLM requests multiple tool calls in one response, the agent runs them concurrently:

```typescript
const agent = new Agent({ toolExecution: 'parallel', … });
```

You can force sequential execution globally or per tool:

```typescript
const myTool: AgentTool = {
  executionMode: 'sequential',  // overrides global setting for this tool
  …
};
```

If any tool in a batch is `sequential`, the entire batch runs sequentially.

---

## Steering and Follow-up

**Steering** lets you interrupt the agent while tools are running and inject a new user message for the next LLM turn:

```typescript
agent.steer({ role: 'user', content: 'Stop, do this instead', timestamp: Date.now() });
```

**Follow-up** queues a message to be processed after the current work completes:

```typescript
agent.followUp({ role: 'user', content: 'Also summarize the result', timestamp: Date.now() });
```

Both queues are drainable:

```typescript
agent.clearSteeringQueue();
agent.clearFollowUpQueue();
agent.clearAllQueues();
```

---

## Hooks: `beforeToolCall` and `afterToolCall`

These intercept tool execution for auditing, access control, or post-processing:

```typescript
const agent = new Agent({
  beforeToolCall: async ({ toolCall, args, context }) => {
    if (toolCall.name === 'bash') {
      return { block: true, reason: 'bash is disabled' };
    }
  },
  afterToolCall: async ({ toolCall, result, isError, context }) => {
    if (!isError) {
      return { details: { ...result.details, audited: true } };
    }
  },
});
```

---

## Custom Message Types

Extend `AgentMessage` without modifying the library:

```typescript
declare module '@mariozechner/pi-agent-core' {
  interface CustomAgentMessages {
    notification: { role: 'notification'; text: string; timestamp: number };
  }
}

// Now valid as an AgentMessage
const msg: AgentMessage = { role: 'notification', text: 'Info', timestamp: Date.now() };
```

Custom messages are filtered out by `convertToLlm` before LLM calls, so the LLM never sees them.

---

## Proxy Support

For browser apps that cannot call provider APIs directly (CORS):

```typescript
import { Agent, streamProxy } from '@mariozechner/pi-agent-core';

const agent = new Agent({
  streamFn: (model, context, options) =>
    streamProxy(model, context, {
      ...options,
      authToken: '...',
      proxyUrl: 'https://your-backend.com',
    }),
});
```

The proxy function re-routes all LLM calls through your own backend.

---

## Low-Level API

If you want direct control without the `Agent` class, the underlying loop functions are exported:

```typescript
import { agentLoop, agentLoopContinue } from '@mariozechner/pi-agent-core';

for await (const event of agentLoop([userMessage], context, config)) {
  console.log(event.type);
}
```

These are observational streams — they do not wait for your async event handlers before moving to the next phase. Use the `Agent` class if you need the `message_end` barrier before tool preflight.

---

## Source Layout

```
packages/agent/src/
├── index.ts         Public API re-exports
├── types.ts         AgentMessage, AgentTool, AgentEvent, AgentState interfaces
├── agent.ts         Agent class — state management, event emission, steering/follow-up
├── agent-loop.ts    agentLoop() / agentLoopContinue() — the raw async generator loop
└── proxy.ts         streamProxy() helper for browser proxy backends
```

---

## Key Dependencies

| Dependency | Purpose |
|-----------|---------|
| `@mariozechner/pi-ai` | LLM streaming calls |
| `typebox` | Tool parameter schema definition and validation |

---

## Pros and Cons

**Pros**
- Handles the entire agentic loop automatically
- Rich, typed event stream suitable for any UI
- Parallel tool execution reduces latency
- Extensible message types without forking
- Steering and follow-up enable reactive, conversational experiences
- Proxy support enables browser deployment

**Cons**
- No built-in message persistence — callers must serialize `agent.state.messages`
- No built-in context window management — callers provide `transformContext` to prune old messages
- Low-level `agentLoop` lacks the `message_end` barrier; use `Agent` for anything production-grade

---

## How to Use It (Junior Developer Walkthrough)

1. **Install:**
   ```bash
   npm install @mariozechner/pi-agent-core @mariozechner/pi-ai
   ```

2. **Create and subscribe to an agent:**
   ```typescript
   import { Agent } from '@mariozechner/pi-agent-core';
   import { getModel } from '@mariozechner/pi-ai';

   const agent = new Agent({
     initialState: {
       systemPrompt: 'You are helpful.',
       model: getModel('anthropic', 'claude-haiku-4-5'),
       thinkingLevel: 'off',
       tools: [],
       messages: [],
     },
   });

   agent.subscribe((event) => {
     if (event.type === 'message_update' && event.assistantMessageEvent.type === 'text_delta') {
       process.stdout.write(event.assistantMessageEvent.delta);
     }
   });
   ```

3. **Send a prompt and wait:**
   ```typescript
   await agent.prompt('Hello!');
   ```

4. **Add a tool:**
   ```typescript
   import { Type } from '@mariozechner/pi-ai';

   agent.state.tools = [{
     name: 'get_time',
     description: 'Return the current UTC time',
     parameters: Type.Object({}),
     execute: async () => ({
       content: [{ type: 'text', text: new Date().toISOString() }],
     }),
   }];
   ```

5. **The agent now automatically calls the tool when the model requests it** — no extra code needed.
