# Observability — pi-mom

mom uses a structured logger (`src/log.ts`) that writes JSON to stderr. This is the primary observability surface.

---

## Structured Logging

```typescript
// src/log.ts (simplified)
export function info(message: string, ...args: unknown[]) {
  process.stderr.write(JSON.stringify({ level: "info", ts: Date.now(), message, args }) + "\n");
}

export function error(message: string, ...args: unknown[]) {
  process.stderr.write(JSON.stringify({ level: "error", ts: Date.now(), message, args }) + "\n");
}
```

Collect with any log aggregator (Loki, Splunk, CloudWatch) by capturing stderr.

---

## What Is Logged by Default

| Event | Level | Fields |
|-------|-------|--------|
| Slack event received | `info` | `channelId`, `userId`, `eventType` |
| Agent run start | `info` | `channelId`, `sessionId` |
| Bash command executed | `info` | `command`, `exitCode`, `durationMs` |
| Slack reply sent | `info` | `channelId`, `messageTs` |
| Error | `error` | `channelId`, `error.message`, `error.stack` |

---

## Adding Agent Event Logging

Subscribe to `AgentEvent`s inside `AgentRunner`:

```typescript
// src/agent.ts (simplified)
this.agent.subscribe((event) => {
  if (event.type === "turn_end") {
    log.info("turn_end", {
      channel: ctx.channel,
      stopReason: (event.message as any).stopReason,
      usage: (event.message as any).usage,
    });
  }
  if (event.type === "tool_execution_end") {
    log.info("tool_end", {
      channel: ctx.channel,
      tool: event.toolName,
      isError: event.isError,
    });
  }
});
```

---

## Workspace Audit Trail

All files written by mom to the workspace are inherently an audit trail:

```
workspace/
  C08ABCDEF/
    session.json     ← full conversation history
    memory.md        ← agent's self-maintained notes
    tools/           ← installed packages
```

Reading `session.json` gives you the complete history of every turn, tool call, and result for a channel.

---

## Suggested Metrics

| Metric | Source |
|--------|--------|
| Slack events received per hour | Log `channelId` + `eventType` |
| Agent runs per channel per day | Log `agent run start` |
| Tool call latency | `tool_end.durationMs` |
| LLM token usage | `turn_end.usage` |
| Error rate | `error` log level count |

---

**What's Next:** [Design Patterns](./design-patterns.md) | [Terminology](./terminology.md)
