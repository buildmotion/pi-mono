# Observability — pi-coding-agent

## Built-in Telemetry

`pi-coding-agent` has a built-in telemetry flag (`PI_TELEMETRY=1` or `settingsManager.getEnableInstallTelemetry()`) that controls anonymous install metrics. This is separate from application-level observability.

Source: `src/core/telemetry.ts`

---

## Extension-Based Event Logging

The richest way to instrument `pi` is via a TypeScript Extension that hooks into all lifecycle events:

```typescript
// ~/.pi/extensions/logger/index.ts
import type { Extension } from "@mariozechner/pi-coding-agent";

export default {
  name: "structured-logger",
  version: "1.0.0",

  async onSessionStart(api) {
    api.log("[session_start]", { sessionId: api.sessionId, model: api.model.id });
  },

  async onTurnEnd(api, event) {
    api.log("[turn_end]", {
      stopReason: event.stopReason,
      inputTokens: event.usage?.input,
      outputTokens: event.usage?.output,
      costUsd: event.usage?.cost?.total,
    });
  },

  async onToolExecutionEnd(api, event) {
    api.log("[tool_end]", {
      tool: event.toolName,
      durationMs: event.durationMs,
      isError: event.isError,
    });
  },
} satisfies Extension;
```

---

## Timing Information

`AgentSession` tracks tool execution durations via `src/core/timings.ts`. Access them in `onToolExecutionEnd`:

```typescript
async onToolExecutionEnd(api, event) {
  console.error(JSON.stringify({
    ts: Date.now(),
    tool: event.toolName,
    ms: event.durationMs,
  }));
}
```

---

## Suggested Metrics

| Metric | Where to collect |
|--------|-----------------|
| Tokens per turn (input/output) | `onTurnEnd` → `event.usage` |
| Tool call latency (ms) | `onToolExecutionEnd` → `event.durationMs` |
| Tool error rate | `onToolExecutionEnd` → `event.isError` |
| Session duration | Timestamp diff between `onSessionStart` and `onSessionEnd` |
| Compaction count | `onBeforeCompact` invocation count |
| Context size at compaction | `onBeforeCompact` → `preparation.estimatedTokens` |

---

## Log Destination

In interactive mode, `api.log()` writes to `~/.pi/logs/`. In print/RPC mode, write to `stderr` to avoid corrupting stdout output:

```typescript
process.stderr.write(JSON.stringify(logEntry) + "\n");
```

---

**What's Next:** [Design Patterns](./design-patterns.md) | [Terminology](./terminology.md)
