# Extension Points — pi-mom

---

## 1. Custom Tools

Add tools to the `AgentRunner`'s Agent by editing `src/tools/index.ts` or by forking `createMomTools()`:

```typescript
// src/tools/index.ts (simplified)
export function createMomTools(executor: SandboxExecutor): AgentTool[] {
  return [
    ...builtinTools(executor),
    myCustomTool(executor), // add your tool here
  ];
}
```

Tools receive the `SandboxExecutor` so they can route execution through host or Docker consistently.

---

## 2. Custom Memory Format

`memory.md` is plain Markdown. You can structure it however you like:

```markdown
## Structured YAML block
```yaml
team:
  backend_url: https://api.example.com
  slack_channels: ["#alerts", "#releases"]
```
```

The entire file is injected as-is into the system prompt. Structured formats (YAML, JSON inside code blocks) work fine because the LLM can parse them.

---

## 3. Custom System Prompt

Override `buildSystemPrompt` in `src/agent.ts` to add org-specific instructions:

```typescript
function buildSystemPrompt(memory: string): string {
  return `You are mom, ${companyName}'s AI assistant.

## Rules
- Never commit secrets.
- Always use the company coding standards.

## Channel Memory
${memory}`;
}
```

---

## 4. Multi-Provider Support

`src/agent.ts` hardcodes Anthropic (`claude-sonnet-4-5`). To use a different provider:

```typescript
// src/agent.ts (simplified)
const model = getModel(
  process.env.MOM_PROVIDER ?? "anthropic",
  process.env.MOM_MODEL ?? "claude-sonnet-4-5",
);
```

---

## 5. Custom Artifact Serving

mom can upload artifacts to Slack directly via the `attach` tool, or serve them from a local HTTP server. To serve from a custom CDN instead, override the upload function:

```typescript
setUploadFunction(async (filePath) => {
  const url = await uploadToMyCDN(filePath);
  return url;
});
```

Source: `src/tools/index.ts` — `setUploadFunction()`.

---

## 6. Slack Event Types

mom currently handles `app_mention` events. To handle other events (reactions, DMs, slash commands), add handlers in `src/slack.ts`:

```typescript
this.socket.on("message", async (raw) => {
  const event = JSON.parse(raw).payload?.event;
  if (event?.type === "reaction_added") {
    await handleReaction(event);
  }
});
```

---

**What's Next:** [Observability](./observability.md) | [Design Patterns](./design-patterns.md)
