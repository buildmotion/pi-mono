# Observability

Every significant event emitted by `pi-agent-core` is observable through the `AgentEvent` subscription API. This document describes the full event taxonomy, how to subscribe, and how to integrate with structured logging and OpenTelemetry.

---

## Subscribing to events

```typescript
import type { AgentEvent } from "@mariozechner/pi-agent-core";

const unsubscribe = agent.subscribe((event: AgentEvent) => {
  // called for every event while the agent is running
  console.log(event.type);
});

// Stop listening
unsubscribe();
```

Callbacks may be async. `Agent` awaits each listener before moving on when using `runAgentLoop` / `runAgentLoopContinue` (the `Agent`-class-level loop). This means your listener can perform async I/O (database writes, HTTP calls) and the agent will not advance until it settles.

---

## Event taxonomy

### Run lifecycle

| Event type | Fired when | Key fields |
|---|---|---|
| `agent_start` | `prompt()` or `continue()` begins a run | `messages` (snapshot at run start) |
| `agent_end` | Run completes (success or failure) | `messages`, `stopReason` |
| `agent_error` | Unrecoverable error — emitted then `agent_end` fires | `error` (Error object) |

### Turn lifecycle

| Event type | Fired when | Key fields |
|---|---|---|
| `turn_start` | Outer loop iteration begins | `turn` (0-based counter) |
| `turn_end` | Outer loop iteration ends | `turn`, assistant message |
| `message_start` | LLM streaming begins | `turn`, `model`, `options` |
| `message_end` | LLM streaming complete | `turn`, full `AssistantMessage` |

### Streaming (inner loop)

| Event type | Fired when | Key fields |
|---|---|---|
| `text` | Partial text token received | `delta`, `snapshot` |
| `thinking` | Thinking/reasoning token received | `delta`, `snapshot` |
| `tool_call` | Tool call delta or full call received | `toolCall` |
| `usage` | Token usage reported by provider | `usage` |
| `stop` | Stream finished | `stopReason`, full `AssistantMessage` |

### Tool execution

| Event type | Fired when | Key fields |
|---|---|---|
| `tool_execution_start` | A tool call is about to run | `toolCall`, `args` |
| `tool_execution_end` | A tool call finished (success or error) | `toolCall`, `result`, `isError` |
| `tool_execution_blocked` | `beforeToolCall` returned `block: true` | `toolCall`, `reason` |

### Steering and follow-up

| Event type | Fired when | Key fields |
|---|---|---|
| `steering` | Steering messages injected | `messages` (the injected messages) |
| `follow_up` | Follow-up messages injected | `messages` |

### Context

| Event type | Fired when | Key fields |
|---|---|---|
| `context_transformed` | `transformContext` returned | `before`, `after` message arrays |

---

## Structured logging example

```typescript
import type { AgentEvent } from "@mariozechner/pi-agent-core";

function structuredLogger(sessionId: string) {
  return (event: AgentEvent) => {
    const base = { sessionId, ts: Date.now(), type: event.type };

    switch (event.type) {
      case "agent_start":
        log.info({ ...base, messageCount: event.messages.length }, "run started");
        break;

      case "agent_end":
        log.info({ ...base, stopReason: event.stopReason }, "run ended");
        break;

      case "agent_error":
        log.error({ ...base, err: event.error }, "run failed");
        break;

      case "turn_start":
        log.debug({ ...base, turn: event.turn }, "turn started");
        break;

      case "message_end":
        log.info(
          {
            ...base,
            model: event.message.model?.id,
            inputTokens: event.message.usage?.inputTokens,
            outputTokens: event.message.usage?.outputTokens,
          },
          "llm turn complete"
        );
        break;

      case "tool_execution_start":
        log.info({ ...base, tool: event.toolCall.name, callId: event.toolCall.id }, "tool start");
        break;

      case "tool_execution_end":
        log.info(
          { ...base, tool: event.toolCall.name, callId: event.toolCall.id, isError: event.isError },
          "tool end"
        );
        break;

      case "tool_execution_blocked":
        log.warn({ ...base, tool: event.toolCall.name, reason: event.reason }, "tool blocked");
        break;
    }
  };
}

agent.subscribe(structuredLogger(sessionId));
```

---

## OpenTelemetry integration

The three natural span boundaries are:

| Span name | Start event | End event | Suggested attributes |
|---|---|---|---|
| `pi.agent.run` | `agent_start` | `agent_end` | `session.id`, `stop_reason`, `message_count` |
| `pi.agent.turn` | `turn_start` | `turn_end` | `turn`, `model.id`, `input_tokens`, `output_tokens` |
| `pi.agent.tool` | `tool_execution_start` | `tool_execution_end` | `tool.name`, `tool.call_id`, `is_error`, `is_blocked` |

```typescript
import { trace, SpanStatusCode, context, ROOT_CONTEXT } from "@opentelemetry/api";
import type { AgentEvent } from "@mariozechner/pi-agent-core";

const tracer = trace.getTracer("pi-agent");

export function otelSubscriber(sessionId: string) {
  let runSpan: Span | undefined;
  const turnSpans = new Map<number, Span>();
  const toolSpans = new Map<string, Span>();

  return (event: AgentEvent) => {
    switch (event.type) {
      case "agent_start":
        runSpan = tracer.startSpan("pi.agent.run", {
          attributes: { "session.id": sessionId },
        });
        break;

      case "agent_end":
        runSpan?.setAttributes({ "stop_reason": event.stopReason ?? "unknown" });
        runSpan?.end();
        runSpan = undefined;
        break;

      case "agent_error":
        runSpan?.setStatus({ code: SpanStatusCode.ERROR, message: event.error.message });
        runSpan?.recordException(event.error);
        break;

      case "turn_start": {
        const span = tracer.startSpan("pi.agent.turn", {
          attributes: { "turn": event.turn },
        });
        turnSpans.set(event.turn, span);
        break;
      }

      case "message_end": {
        const span = turnSpans.get(event.turn);
        span?.setAttributes({
          "model.id": event.message.model?.id ?? "unknown",
          "input_tokens": event.message.usage?.inputTokens ?? 0,
          "output_tokens": event.message.usage?.outputTokens ?? 0,
        });
        break;
      }

      case "turn_end":
        turnSpans.get(event.turn)?.end();
        turnSpans.delete(event.turn);
        break;

      case "tool_execution_start": {
        const span = tracer.startSpan("pi.agent.tool", {
          attributes: {
            "tool.name": event.toolCall.name,
            "tool.call_id": event.toolCall.id ?? "",
          },
        });
        toolSpans.set(event.toolCall.id ?? event.toolCall.name, span);
        break;
      }

      case "tool_execution_end": {
        const key = event.toolCall.id ?? event.toolCall.name;
        const span = toolSpans.get(key);
        span?.setAttributes({ "is_error": event.isError });
        if (event.isError) {
          span?.setStatus({ code: SpanStatusCode.ERROR });
        }
        span?.end();
        toolSpans.delete(key);
        break;
      }

      case "tool_execution_blocked": {
        const span = tracer.startSpan("pi.agent.tool", {
          attributes: {
            "tool.name": event.toolCall.name,
            "is_blocked": true,
            "block_reason": event.reason ?? "",
          },
        });
        span.end();
        break;
      }
    }
  };
}

agent.subscribe(otelSubscriber(sessionId));
```

---

## Suggested metrics

| Metric name | Type | Labels | Notes |
|---|---|---|---|
| `pi_agent_run_total` | Counter | `stop_reason` | Completed runs by termination reason |
| `pi_agent_run_duration_ms` | Histogram | `stop_reason` | Wall time from `agent_start` to `agent_end` |
| `pi_agent_turns_per_run` | Histogram | — | Number of LLM turns in a run |
| `pi_agent_tokens_input` | Counter | `model` | Cumulative input tokens |
| `pi_agent_tokens_output` | Counter | `model` | Cumulative output tokens |
| `pi_agent_tool_calls_total` | Counter | `tool`, `is_error`, `is_blocked` | Tool invocation counts |
| `pi_agent_tool_duration_ms` | Histogram | `tool`, `is_error` | Tool execution latency |

Collect these from the same event subscriber pattern shown above.
