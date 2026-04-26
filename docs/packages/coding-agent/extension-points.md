# Extension Points — pi-coding-agent

`pi` is built to be extended at every level. The four main axes are: tools, extensions, prompt customization, and run-mode integration.

---

## 1. TypeScript Extensions (deepest integration)

Extensions are TypeScript modules loaded at startup. They receive the full `ExtensionAPI`:

```typescript
// Extension lifecycle hooks available
interface Extension {
  name: string;
  version: string;
  onSessionStart?(api: ExtensionAPI): Promise<void>;
  onSessionEnd?(api: ExtensionAPI): Promise<void>;
  onTurnStart?(api: ExtensionAPI, event: TurnStartEvent): Promise<void>;
  onTurnEnd?(api: ExtensionAPI, event: TurnEndEvent): Promise<void>;
  onMessageStart?(api: ExtensionAPI, event: MessageStartEvent): Promise<void>;
  onMessageEnd?(api: ExtensionAPI, event: MessageEndEvent): Promise<void>;
  onToolExecutionStart?(api: ExtensionAPI, event: ToolExecutionStartEvent): Promise<void>;
  onToolExecutionEnd?(api: ExtensionAPI, event: ToolExecutionEndEvent): Promise<void>;
  onBeforeToolCall?(api: ExtensionAPI, toolName: string, args: unknown): Promise<{ block?: boolean; reason?: string } | undefined>;
  onAfterToolCall?(api: ExtensionAPI, toolName: string, args: unknown, result: unknown): Promise<void>;
  onBeforeCompact?(api: ExtensionAPI, preparation: CompactionPreparation): Promise<SessionBeforeCompactResult | undefined>;
}
```

Register in `~/.pi/extensions.json`:
```json
{ "extensions": ["~/.pi/extensions/my-extension/index.js"] }
```

---

## 2. Custom Tools via Extension

Inside `onSessionStart`, call `api.registerTool()`:

```typescript
api.registerTool({
  name: "deploy",
  label: "Deploy",
  description: "Runs the deployment pipeline.",
  parameters: Type.Object({ env: Type.String() }),
  execute: async (_id, { env }) => {
    const result = await api.exec(`./deploy.sh ${env}`);
    return { content: [{ type: "text", text: result.stdout }], details: {} };
  },
});
```

---

## 3. System Prompt Customization

Extensions can append to or replace the system prompt:

```typescript
api.appendToSystemPrompt("Always follow our team's coding standards: ...");
```

Or use a Prompt Template file — see [extensions-and-skills.md](./features/extensions-and-skills.md).

---

## 4. Custom Model Selection

Override the model resolver to pick models dynamically:

```typescript
// In SDK mode
const session = await createAgentSession({
  modelResolver: async () => models.find(m => m.id === "my-custom-model")!,
});
```

---

## 5. Slash Commands

Extensions can register `/commands` that appear in the TUI autocomplete:

```typescript
api.registerSlashCommand({
  name: "/deploy",
  description: "Trigger deployment",
  execute: async (api, args) => {
    await api.agent.prompt(`Run deployment for environment: ${args}`);
  },
});
```

---

## 6. SDK Mode (Full Programmatic Control)

Use `pi-coding-agent` as a library for maximum control:

```typescript
import { createAgentSession } from "@mariozechner/pi-coding-agent";

const session = await createAgentSession({ workingDir: "/my/project" });
await session.prompt("Add error handling to all async functions.");
```

---

**What's Next:** [Observability](./observability.md) | [Design Patterns](./design-patterns.md)
