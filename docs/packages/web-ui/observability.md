# Observability — pi-web-ui

Browser AI applications have unique observability needs: you cannot write to a log file, and you must be careful not to log private user content. This page shows how to instrument `pi-web-ui` using agent events, performance marks, and structured console logging.

---

## 1. Subscribing to Agent Events

All significant lifecycle moments surface as `AgentEvent` objects. Subscribe once after creating the agent:

```typescript
agent.subscribe((event) => {
  switch (event.type) {
    case "agent_start":
      performance.mark("agent_start");
      break;
    case "turn_end":
      console.debug("[pi-web-ui] turn_end", {
        model: agent.state.model.id,
        stopReason: event.message.role === "assistant"
          ? (event.message as any).stopReason
          : undefined,
      });
      break;
    case "tool_execution_end":
      console.debug("[pi-web-ui] tool", {
        name: event.toolName,
        isError: event.isError,
      });
      break;
    case "agent_end":
      performance.mark("agent_end");
      performance.measure("agent_run", "agent_start", "agent_end");
      const [measure] = performance.getEntriesByName("agent_run");
      console.info("[pi-web-ui] run complete", { durationMs: measure.duration });
      break;
  }
});
```

---

## 2. Token Usage Logging

After `agent_end`, inspect the last assistant message for token counts:

```typescript
agent.subscribe((event) => {
  if (event.type === "agent_end") {
    for (const msg of event.messages) {
      if (msg.role === "assistant") {
        const { input, output, cost } = msg.usage;
        console.info("[pi-web-ui] usage", { input, output, totalCostUsd: cost.total });
      }
    }
  }
});
```

---

## 3. Performance Marks for Streaming Latency

Measure time-to-first-token:

```typescript
let firstToken = false;

agent.subscribe((event) => {
  if (event.type === "agent_start") {
    firstToken = false;
    performance.mark("prompt_sent");
  }
  if (event.type === "message_update" && !firstToken) {
    firstToken = true;
    performance.mark("first_token");
    performance.measure("ttft", "prompt_sent", "first_token");
    const [m] = performance.getEntriesByName("ttft");
    console.info("[pi-web-ui] TTFT", { ms: m.duration });
  }
});
```

---

## 4. Error Tracking

```typescript
agent.subscribe((event) => {
  if (event.type === "turn_end" && event.message.role === "assistant") {
    const msg = event.message as any;
    if (msg.stopReason === "error" || msg.stopReason === "aborted") {
      console.error("[pi-web-ui] agent error", { errorMessage: msg.errorMessage });
      // Forward to Sentry, Datadog, etc.
    }
  }
});
```

---

## Suggested Metrics

| Metric | How to collect |
|--------|---------------|
| Time-to-first-token (ms) | `performance.measure` between `agent_start` and first `message_update` |
| Total run duration (ms) | `performance.measure` between `agent_start` and `agent_end` |
| Token cost (USD) | `AssistantMessage.usage.cost.total` |
| Tool calls per run | Count `tool_execution_end` events per `agent_start`/`agent_end` pair |
| Tool error rate | `tool_execution_end.isError === true` count / total tool calls |

---

**What's Next:** [Design Patterns](./design-patterns.md) | [Terminology](./terminology.md)
