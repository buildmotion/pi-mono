# Observability

**Learning objectives.** After reading this document you should be able to:

- Explain why agentic systems have higher observability requirements than stateless request/response services.
- Map every `AgentEvent` type to a concrete log entry and the information it should capture.
- Draw the trace boundary for a single agent turn and for a single tool execution.
- List the five key metrics to track for a production agent deployment.
- Write a structured-logging subscriber that hooks into the `Agent` event system with zero changes to agent internals.

---

## 1. Why Observability Matters More in Agentic Systems

A conventional REST API call succeeds or fails in milliseconds. An agent turn can take tens of seconds, invoke multiple tools, spawn sub-agents, and fail partway through a complex multi-step plan. The following failure modes are unique to agentic workloads:

- **Silent degradation.** The LLM continues to produce output after a context window overflow that truncated earlier instructions. The output looks plausible but is wrong.
- **Tool call storms.** A bug in a tool's result schema causes the model to keep retrying the same tool call in a loop until `maxTokens` is exhausted.
- **Cost runaway.** A long-running agent accumulates large context windows across many turns, producing token usage that is orders of magnitude higher than expected.
- **Partial success.** Three out of four tool calls in a batch succeed. The failing one produces an error result that the model silently ignores on the next turn.

None of these failure modes produce an exception or an HTTP 500. They are only visible through structured event logs, token usage metrics, and trace spans that capture the full turn lifecycle.

---

## 2. Agent Event Taxonomy

Every significant runtime event in the agent loop is emitted as an `AgentEvent`. The table below maps each event type to what it signals and what information should be captured.

| Event type | Signals | Capture in logs |
|------------|---------|-----------------|
| `agent_start` | A new prompt/continuation run has started | `runId`, `sessionId`, `model.id`, `model.provider`, timestamp |
| `agent_end` | The run finished (success, error, or abort) | `runId`, `stopReason`, final message count, total turns, elapsed ms |
| `turn_start` | A new LLM call is about to be made | `runId`, `turnIndex`, context size (message count), tool count |
| `turn_end` | One LLM call + tool batch completed | `runId`, `turnIndex`, `StopReason`, tool result count, elapsed ms |
| `message_start` | A new message is being constructed | `runId`, `turnIndex`, `message.role` |
| `message_update` | A streaming delta arrived | `runId`, `turnIndex`, event subtype (`text_delta`, `thinking_delta`, `toolcall_delta`) |
| `message_end` | A message is fully assembled | `runId`, `turnIndex`, `message.role`, `message.stopReason`, input/output token counts |
| `tool_execution_start` | A tool call is about to execute | `runId`, `turnIndex`, `toolCallId`, `toolName`, argument summary |
| `tool_execution_update` | A tool emitted a partial result | `runId`, `toolCallId`, partial result summary |
| `tool_execution_end` | A tool call finished | `runId`, `toolCallId`, `toolName`, `isError`, elapsed ms |

`message_update` events fire at high frequency (once per streaming token). **Do not log every `message_update` event in production.** Log only `message_start`, `message_end`, and `tool_execution_*` events at full fidelity; sample or aggregate `message_update` events.

---

## 3. Suggested Log Schema

Use structured JSON logs. The examples below use a flat schema compatible with Datadog, Elasticsearch, and most cloud logging services.

### `agent_start`

```json
{
  "event": "agent_start",
  "runId": "run_01j9xk...",
  "sessionId": "sess_abc...",
  "modelId": "claude-opus-4-5",
  "provider": "anthropic",
  "timestamp": 1718000000000
}
```

### `agent_end`

```json
{
  "event": "agent_end",
  "runId": "run_01j9xk...",
  "elapsedMs": 14320,
  "totalTurns": 3,
  "finalMessageCount": 12,
  "stopReason": "stop",
  "errorMessage": null,
  "timestamp": 1718000014320
}
```

### `turn_end`

```json
{
  "event": "turn_end",
  "runId": "run_01j9xk...",
  "turnIndex": 1,
  "elapsedMs": 5210,
  "stopReason": "toolUse",
  "toolResultCount": 2,
  "inputTokens": 4812,
  "outputTokens": 347,
  "cacheReadTokens": 1200,
  "inputCostUsd": 0.0144,
  "outputCostUsd": 0.0052,
  "timestamp": 1718000005210
}
```

### `tool_execution_end`

```json
{
  "event": "tool_execution_end",
  "runId": "run_01j9xk...",
  "turnIndex": 1,
  "toolCallId": "call_xyz...",
  "toolName": "bash",
  "isError": false,
  "elapsedMs": 820,
  "contentLengthChars": 1340,
  "timestamp": 1718000005100
}
```

### `message_end` (assistant)

```json
{
  "event": "message_end",
  "runId": "run_01j9xk...",
  "turnIndex": 1,
  "role": "assistant",
  "stopReason": "toolUse",
  "inputTokens": 4812,
  "outputTokens": 347,
  "cacheReadTokens": 1200,
  "cacheWriteTokens": 0,
  "totalCostUsd": 0.0196,
  "thinkingEnabled": false,
  "toolCallCount": 2,
  "timestamp": 1718000004900
}
```

---

## 4. Trace Boundary Recommendations

Structure distributed traces around two boundaries:

### One span per turn

Start a trace span on `turn_start`. End it on `turn_end`. Set the following span attributes:

- `ai.model.id`
- `ai.model.provider`
- `ai.turn.index`
- `ai.turn.stop_reason`
- `ai.tokens.input`, `ai.tokens.output`, `ai.tokens.cache_read`
- `ai.cost.total_usd`

Child spans for tool executions should be nested under this turn span.

### One span per tool execution

Start a child span on `tool_execution_start`. End it on `tool_execution_end`. Set:

- `ai.tool.name`
- `ai.tool.call_id`
- `ai.tool.is_error`
- `ai.tool.elapsed_ms`

The parent context (turn span) is propagated via the `runId` field in both events. If you use OpenTelemetry, store the active span context in a `Map<runId, Span>` that your subscriber maintains.

### Parent span for the full run

Wrap the entire `agent.prompt()` call in a root span. Start it before the call; end it when the `agent_end` event arrives. Link the root span as parent of all turn spans.

---

## 5. Key Metrics

Track the following metrics in your metrics backend (Prometheus, Datadog, CloudWatch, etc.):

| Metric | Type | Labels | Why it matters |
|--------|------|--------|----------------|
| `agent.turn.duration_ms` | histogram | `model_id`, `provider`, `stop_reason` | Detects slow LLM responses; segment by stop reason to separate tool-use turns from final turns |
| `agent.turn.input_tokens` | histogram | `model_id`, `provider` | Context window utilization; early warning for context overflow |
| `agent.turn.output_tokens` | histogram | `model_id`, `provider` | Output length distribution; high variance signals inconsistent model behavior |
| `agent.turn.cost_usd` | counter | `model_id`, `provider` | Cost accounting per run/session/user |
| `agent.tool.duration_ms` | histogram | `tool_name`, `is_error` | Tool latency profiling; high error rate signals broken tool implementations |
| `agent.run.turns_total` | histogram | `model_id` | Detects tool call storms (runs with abnormally high turn counts) |
| `agent.run.error_rate` | counter | `model_id`, `provider` | Overall reliability tracking |

---

## 6. Subscribing to Agent Events

The `Agent` class exposes a `subscribe` method that accepts a callback receiving `AgentEvent` values. The callback can be synchronous or async. If it returns a `Promise`, the agent loop awaits the `agent_end` listener before marking the run as complete.

```typescript
import type { Agent } from "@mariozechner/pi-agent-core";
import type { AgentEvent } from "@mariozechner/pi-agent-core";

function attachTelemetry(agent: Agent): void {
  agent.subscribe((event: AgentEvent) => {
    switch (event.type) {
      case "agent_start":
      case "agent_end":
      case "turn_start":
      case "turn_end":
      case "message_start":
      case "message_end":
      case "tool_execution_start":
      case "tool_execution_end":
        logEvent(event);
        break;
      // message_update fires once per token — skip in production
    }
  });
}
```

Call `attachTelemetry(agent)` once after constructing the `Agent` instance, before the first `agent.prompt()` call. The subscriber is retained for the lifetime of the agent instance and receives events from all future runs.

---

## 7. Structured Logger — Complete Example

The following example shows a self-contained structured logger that attaches to an `Agent` and emits JSON log lines to stdout. It tracks `runId`, `turnIndex`, and per-turn token accumulation.

```typescript
import type { Agent, AgentEvent } from "@mariozechner/pi-agent-core";

interface RunState {
  runId: string;
  startMs: number;
  turnIndex: number;
  turnStartMs: number;
}

function attachStructuredLogger(agent: Agent, sessionId: string): void {
  let state: RunState | null = null;

  function emit(record: Record<string, unknown>): void {
    process.stdout.write(JSON.stringify({ sessionId, ...record }) + "\n");
  }

  agent.subscribe((event: AgentEvent) => {
    switch (event.type) {
      case "agent_start": {
        state = {
          runId: crypto.randomUUID(),
          startMs: Date.now(),
          turnIndex: 0,
          turnStartMs: Date.now(),
        };
        emit({
          event: "agent_start",
          runId: state.runId,
          timestamp: state.startMs,
        });
        break;
      }

      case "agent_end": {
        if (!state) break;
        emit({
          event: "agent_end",
          runId: state.runId,
          elapsedMs: Date.now() - state.startMs,
          totalTurns: state.turnIndex,
          finalMessageCount: event.messages.length,
          timestamp: Date.now(),
        });
        state = null;
        break;
      }

      case "turn_start": {
        if (!state) break;
        state.turnStartMs = Date.now();
        state.turnIndex++;
        emit({
          event: "turn_start",
          runId: state.runId,
          turnIndex: state.turnIndex,
          timestamp: Date.now(),
        });
        break;
      }

      case "turn_end": {
        if (!state) break;
        emit({
          event: "turn_end",
          runId: state.runId,
          turnIndex: state.turnIndex,
          elapsedMs: Date.now() - state.turnStartMs,
          toolResultCount: event.toolResults.length,
          timestamp: Date.now(),
        });
        break;
      }

      case "message_end": {
        if (!state) break;
        const msg = event.message;
        if ("role" in msg && msg.role === "assistant" && "usage" in msg) {
          emit({
            event: "message_end",
            runId: state.runId,
            turnIndex: state.turnIndex,
            role: "assistant",
            stopReason: msg.stopReason,
            inputTokens: msg.usage?.input ?? 0,
            outputTokens: msg.usage?.output ?? 0,
            totalCostUsd: msg.usage?.cost?.total ?? 0,
            timestamp: Date.now(),
          });
        }
        break;
      }

      case "tool_execution_start": {
        if (!state) break;
        emit({
          event: "tool_execution_start",
          runId: state.runId,
          turnIndex: state.turnIndex,
          toolCallId: event.toolCallId,
          toolName: event.toolName,
          timestamp: Date.now(),
        });
        break;
      }

      case "tool_execution_end": {
        if (!state) break;
        emit({
          event: "tool_execution_end",
          runId: state.runId,
          turnIndex: state.turnIndex,
          toolCallId: event.toolCallId,
          toolName: event.toolName,
          isError: event.isError,
          timestamp: Date.now(),
        });
        break;
      }
    }
  });
}
```

---

## 8. Extension Points for Telemetry

Each package provides hooks where telemetry code can be inserted without modifying core logic.

### `packages/ai`

- **`StreamOptions.onPayload`** — inspect the raw request payload before it is sent. Log request body size, model parameters, tool count.
- **`StreamOptions.onResponse`** — inspect HTTP response headers. Log rate-limit headers (`x-ratelimit-remaining-requests`, `retry-after`) before the stream is consumed.

### `packages/agent`

- **`AgentLoopConfig.beforeToolCall`** — log tool call initiation with validated arguments. Record start time for latency measurement.
- **`AgentLoopConfig.afterToolCall`** — log tool call completion with result metadata. Compute elapsed time.
- **`agent.subscribe()`** — the primary telemetry hook. Attach one subscriber per concern (logging, metrics, tracing) to keep each concern isolated.

### `packages/coding-agent`

- The coding agent emits all agent events through its `Agent` instance. Attach subscribers at session initialization time (after `createCodingAgent()`, before the first `prompt()` call).

### `packages/mom`

- One `Agent` instance per Slack channel. Attach a subscriber when the channel's agent is created. Include `channelId` in the `sessionId` so all logs for a channel are correlated.

### `packages/tui` / `packages/web-ui`

- UI packages do not instrument agent behavior; they are purely rendering layers. Instrument them separately for UX metrics (render latency, input lag) using framework-native tooling.
