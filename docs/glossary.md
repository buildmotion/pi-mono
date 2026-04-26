# Glossary

This file is the canonical vocabulary reference for pi-mono. When you read code, comments, or other docs and encounter an unfamiliar term, look it up here first. Terms are defined precisely, with pointers to the TypeScript source where the concept is encoded.

---

## Agent Loop

The **agent loop** is the iterative control cycle that drives an autonomous agent. In pi-mono the loop lives in `packages/agent/src/agent-loop.ts`. One iteration — called a **turn** — consists of:

1. Call the LLM with the current context.
2. Stream the assistant response.
3. If the response contains tool calls, execute them (in parallel or sequentially).
4. Append the tool results to the context.
5. Check stop conditions (`StopReason`, steering messages, follow-up messages).
6. If not stopping, go to step 1.

The loop does not throw. All failures are encoded in the event stream as `AgentEvent` values. See also: **Turn**, **StopReason**, **AgentEvent**.

---

## Tool Call

A **tool call** is a structured request emitted inside an assistant message asking the runtime to invoke a named function with typed arguments. In the type system a tool call is represented as `ToolCall` inside `AssistantMessage.content`:

```typescript
// packages/ai/src/types.ts
export interface ToolCall {
  type: "toolCall";
  id: string;
  name: string;
  arguments: Record<string, any>;
  thoughtSignature?: string;
}
```

The agent loop extracts tool calls from the completed assistant message, validates arguments against the tool's TypeBox schema, executes the tool, and appends a `ToolResultMessage` to the conversation. See also: **AgentTool**, **ToolResultMessage**.

---

## Context Window

The **context window** is the maximum number of tokens an LLM can process in a single request — the combined length of the system prompt, the conversation history, and the response. It is a hard limit enforced by the model. When the conversation grows past this limit, older messages must be dropped or summarised before the next call. In pi-mono the context window size is exposed on `Model.contextWindow`:

```typescript
// packages/ai/src/types.ts
export interface Model<TApi extends Api> {
  contextWindow: number; // max total tokens (input + output)
  maxTokens: number;     // max output tokens per response
  // ...
}
```

The `transformContext` hook in `AgentLoopConfig` is the correct place to implement context window management (pruning, summarisation).

---

## Context (the object)

`Context` is the plain data structure passed to a `StreamFunction` on every LLM call. It groups everything the model needs to generate a response:

```typescript
// packages/ai/src/types.ts
export interface Context {
  systemPrompt?: string;
  messages: Message[];
  tools?: Tool[];
}
```

Do not confuse this with the broader notion of "context" meaning "the state of the conversation." The `Context` object is a snapshot handed to the LLM; the agent manages a richer `AgentMessage[]` history internally and converts it to `Message[]` via `convertToLlm` before each call.

---

## Message

`Message` is the union type for everything that can appear in a conversation history:

```typescript
// packages/ai/src/types.ts
export type Message = UserMessage | AssistantMessage | ToolResultMessage;
```

- **`UserMessage`** — content from the human (text or image). Has `role: "user"`.
- **`AssistantMessage`** — content from the model. Has `role: "assistant"`. Content is an array that can include `TextContent`, `ThinkingContent`, `ImageContent`, and `ToolCall` blocks.
- **`ToolResultMessage`** — the outcome of a tool call returned to the model. Has `role: "tool"`. Contains one or more `TextContent` or `ImageContent` blocks and a `toolCallId` linking it back to the originating `ToolCall`.

All message types carry a `timestamp` (Unix ms). The agent layer extends this with `AgentMessage`, which is `Message | CustomAgentMessages[keyof CustomAgentMessages]`, allowing apps to inject custom message types via declaration merging.

---

## AssistantMessageEvent / Streaming Event

An **AssistantMessageEvent** is one element of the streaming event sequence emitted while an LLM is generating a response. The full union is:

```typescript
// packages/ai/src/types.ts
export type AssistantMessageEvent =
  | { type: "start"; partial: AssistantMessage }
  | { type: "text_start"; contentIndex: number; partial: AssistantMessage }
  | { type: "text_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "text_end"; contentIndex: number; partial: AssistantMessage }
  | { type: "thinking_start"; contentIndex: number; partial: AssistantMessage }
  | { type: "thinking_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "thinking_end"; contentIndex: number; partial: AssistantMessage }
  | { type: "toolcall_start"; contentIndex: number; toolCall: ToolCall; partial: AssistantMessage }
  | { type: "toolcall_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "toolcall_end"; contentIndex: number; toolCall: ToolCall; partial: AssistantMessage }
  | { type: "usage"; usage: Usage }
  | { type: "done"; reason: "stop" | "length" | "toolUse"; message: AssistantMessage }
  | { type: "error"; reason: "aborted" | "error"; error: AssistantMessage };
```

Every event includes a `partial` field containing the in-progress `AssistantMessage` built up so far. The sequence always starts with `start` and ends with either `done` or `error`. The agent loop in `packages/agent/src/agent-loop.ts` consumes this stream and re-emits higher-level `AgentEvent` values. The UI layer consumes `AgentEvent`; it never needs to parse raw `AssistantMessageEvent` values.

---

## StreamFunction / streamSimple

`StreamFunction` is the canonical type signature for making an LLM call in pi-mono:

```typescript
// packages/ai/src/types.ts
export type StreamFunction<TApi extends Api, TOptions extends StreamOptions> = (
  model: Model<TApi>,
  context: Context,
  options?: TOptions,
) => AssistantMessageEventStream;
```

`streamSimple` is a provider-agnostic wrapper exported from `packages/ai/src/index.ts` that accepts a `Model<any>` and a `SimpleStreamOptions` (which adds `reasoning` and `thinkingBudgets` on top of base `StreamOptions`). Internally it dispatches to the correct provider implementation based on `model.api`. The agent loop calls `streamSimple` exclusively. Direct provider-specific functions (`streamAnthropic`, `streamOpenAI`, etc.) are available for use cases that need provider-specific options.

---

## Model

A `Model` is a typed descriptor for one deployable LLM:

```typescript
// packages/ai/src/types.ts
export interface Model<TApi extends Api> {
  id: string;           // provider's canonical model id (e.g. "claude-opus-4-5")
  name: string;         // human-readable name
  api: TApi;            // which wire protocol to use
  provider: Provider;   // which provider hosts this model
  baseUrl: string;      // API endpoint
  cost: { input: number; output: number; cacheRead: number; cacheWrite: number };
  contextWindow: number;
  maxTokens: number;
  // ... feature flags (supportsImages, supportsToolUse, supportsThinking, etc.)
}
```

The `api` field determines which `StreamFunction` implementation is used. `provider` is purely informational (auth lookup, display). The generated model catalogue lives in `packages/ai/src/models.generated.ts` and is regenerated by `packages/ai/scripts/generate-models.ts`.

---

## Provider

A **provider** is a company or service that hosts LLMs — for example `"anthropic"`, `"openai"`, `"google"`, `"amazon-bedrock"`, `"groq"`. In pi-mono, `Provider` is a string union of known provider identifiers plus an open string escape hatch for custom deployments. Providers are distinct from **APIs**: one provider may expose multiple wire protocols (e.g., OpenAI exposes both `openai-completions` and `openai-responses`), and one wire protocol may be used by multiple providers (e.g., `openai-completions` is used by OpenAI, Groq, Cerebras, and many others).

---

## API (in pi-ai sense)

An **API** in pi-ai refers to a wire protocol and streaming format — not a provider. The `KnownApi` union lists every supported protocol:

```typescript
// packages/ai/src/types.ts
export type KnownApi =
  | "openai-completions"
  | "mistral-conversations"
  | "openai-responses"
  | "azure-openai-responses"
  | "openai-codex-responses"
  | "anthropic-messages"
  | "bedrock-converse-stream"
  | "google-generative-ai"
  | "google-gemini-cli"
  | "google-vertex";
```

`Model.api` picks one of these, and `streamSimple` uses it to dispatch to the correct implementation. Adding a new provider that speaks an existing wire protocol (e.g., a self-hosted OpenAI-compatible endpoint) requires only a new `Model` entry, not a new API implementation.

---

## Thinking / Reasoning

**Thinking** (also called reasoning) is extended internal monologue that some models generate before producing their visible response. Models like Claude 3.7 Sonnet and o1/o3 use this to work through complex problems step-by-step. In pi-mono, thinking content is represented as `ThinkingContent` blocks inside `AssistantMessage.content`:

```typescript
export interface ThinkingContent {
  type: "thinking";
  thinking: string;
  thinkingSignature?: string;
  redacted?: boolean;
}
```

The `SimpleStreamOptions.reasoning` field controls the thinking budget via a `ThinkingLevel` enum: `"minimal" | "low" | "medium" | "high" | "xhigh"`. Not all models support thinking; check `model.supportsThinking`. When thinking is enabled, the `AgentEvent` stream will include `message_update` events with `thinking_delta` inside the `assistantMessageEvent`.

---

## StopReason

`StopReason` is the terminal condition that ended an assistant message:

```typescript
// packages/ai/src/types.ts
export type StopReason = "stop" | "length" | "toolUse" | "error" | "aborted";
```

- `"stop"` — the model finished naturally.
- `"length"` — the response was truncated at `maxTokens`.
- `"toolUse"` — the model emitted one or more tool calls; the loop continues.
- `"error"` — an unrecoverable error occurred (network failure, invalid response, etc.).
- `"aborted"` — the caller cancelled via `AbortSignal`.

The agent loop reads `StopReason` to decide whether to execute tool calls, continue looping, or surface an error. Length truncation mid-tool-call is treated as an error.

---

## Steering Message

A **steering message** is a `AgentMessage` injected into the conversation **while the agent is running**, after the current turn's tool calls complete and before the next LLM call. This allows an external controller (e.g., a UI, another agent) to redirect or constrain the agent mid-flight without aborting it. Steering is implemented via the `getSteeringMessages` callback in `AgentLoopConfig`:

```typescript
getSteeringMessages?: () => Promise<AgentMessage[]>;
```

The loop calls this callback after every tool batch. If it returns non-empty messages, they are appended to the context and the loop continues with another LLM call. If the callback returns `[]`, the loop checks `getFollowUpMessages` before stopping.

---

## Follow-Up Message

A **follow-up message** is a `AgentMessage` that waits until the agent has finished its current work before being processed. The `getFollowUpMessages` callback is consulted only when the agent has no pending tool calls and no steering messages — i.e., it would otherwise stop. This is the correct mechanism for queuing user input that arrives while the agent is busy:

```typescript
getFollowUpMessages?: () => Promise<AgentMessage[]>;
```

If follow-up messages are returned, they are appended to the context and the loop runs another turn. This allows single-agent instances to handle queued user messages without spawning multiple concurrent runs.

---

## Extension Point

An **extension point** is a callback or interface in `AgentLoopConfig` that lets calling code inject custom logic into the agent loop without modifying the loop itself. The extension points are:

| Callback | When it is called | Use case |
|----------|-------------------|----------|
| `convertToLlm` | Before every LLM call | Custom message format conversion |
| `transformContext` | Before `convertToLlm` | Context window management, injection |
| `getApiKey` | Before every LLM call | Dynamic / rotating credentials |
| `getSteeringMessages` | After every tool batch | Mid-run redirection |
| `getFollowUpMessages` | When the agent would stop | Queued input handling |
| `beforeToolCall` | Before each tool executes | Approval gates, sandboxing, logging |
| `afterToolCall` | After each tool executes | Result transformation, audit logging |

Each extension point is documented in detail in `packages/agent/src/types.ts`.

---

## Telemetry / Observability

**Telemetry** in pi-mono is achieved by subscribing to the `AgentEvent` stream that the `Agent` class emits. Because every significant runtime event — turn start/end, message start/end, tool execution start/update/end — is surfaced as a typed `AgentEvent`, a telemetry subscriber never needs to instrument the agent internals. See `reference/observability.md` for the full event taxonomy, suggested log schemas, and trace boundary recommendations.

---

## C4 Model

The **C4 model** is a hierarchical approach to software architecture diagrams (Context, Container, Component, Code). It is referenced in architecture discussions to indicate the level of abstraction being described. pi-mono's `reference/architecture-overview.md` describes the system at the Container level (packages) and the Component level (major modules within each package).

---

## TUI

**TUI** stands for Text User Interface — a terminal-based UI rendered using ANSI escape codes. pi-mono's `packages/tui` (`@mariozechner/pi-tui`) is a custom TUI framework designed for flicker-free rendering. It manages a virtual screen buffer, computes diffs against the previous frame, and emits only the necessary escape sequences. This eliminates the full-screen repaint flicker common in naive terminal UIs. The coding agent's interactive mode uses pi-tui to render the chat interface, editor, and overlays.

---

## Artifacts

**Artifacts** are structured outputs produced by tools and stored alongside the conversation history. In the coding agent context, an artifact is typically a file or code block written to disk by the `write` or `edit` tool. The `AgentToolResult.details` field carries arbitrary structured data (typed by `TDetails`) that the UI layer can render as an artifact card rather than raw text.

---

## Skills (coding-agent)

A **skill** in `packages/coding-agent` is a composable unit of agent capability — a named set of tools plus a prompt fragment that teaches the model how and when to use those tools. Skills are loaded and composed at startup. The coding agent's interactive mode allows skills to be registered at runtime. This is how the agent's toolset is extended without forking the agent loop.

---

## Pi Package

A **pi package** (lowercase) refers to any of the individually published npm packages in this monorepo (`@mariozechner/pi-ai`, `@mariozechner/pi-agent-core`, etc.). The term is used to distinguish internal monorepo packages from third-party dependencies. All pi packages are versioned in lockstep and released together by the release script.

---

## vLLM

**vLLM** is an open-source inference engine for self-hosting large language models on GPU hardware. It exposes an OpenAI-compatible HTTP API, so models running on vLLM can be addressed in pi-ai using `api: "openai-completions"` with a custom `baseUrl`. The `packages/pods` package automates provisioning GPU pods and deploying vLLM on them.

---

## GPU Pod

A **GPU pod** is a cloud-provisioned virtual machine with one or more GPUs, used to run self-hosted LLMs via vLLM or similar inference engines. `packages/pods` manages the lifecycle of GPU pods: creation, vLLM deployment, health checking, and teardown. GPU pods allow teams to run open-weight models (LLaMA, Mistral, etc.) without sending data to third-party providers.
