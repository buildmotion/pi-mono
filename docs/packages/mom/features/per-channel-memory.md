# Feature: Per-Channel Memory

## Learning Objectives

- Understand how mom gives each Slack channel its own persistent memory.
- Know the memory file format and how it is loaded into the agent's system prompt.
- See how live edits to memory.md are picked up without restarting mom.

---

## Background

Every Slack channel that mom participates in gets its own subdirectory inside the workspace:

```
workspace/
  C08ABCDEF/         ← channel ID
    session.json     ← serialized AgentMessage[] history
    memory.md        ← free-form markdown memory file
    tools/           ← npm packages installed for this channel
```

The `memory.md` file is injected verbatim into mom's system prompt at the start of each conversation turn. This gives the agent persistent, human-editable context that survives process restarts.

Source: `src/store.ts`, `src/agent.ts`, `src/fs-watch.ts`

---

## Memory File Format

`memory.md` is plain Markdown. There is no schema — mom and humans can write whatever is useful:

```markdown
# #engineering channel memory

## Team context
- Backend stack: Node.js + PostgreSQL
- Deployment: Kubernetes on GCP
- On-call rotation: Alice (this week), Bob (next week)

## Recurring tasks
- Every Monday: check CI failures and post summary
- Daily standup reminder at 09:00 UTC

## Installed tools
- jq (via apt)
- gh (GitHub CLI)
```

---

## How Memory Is Loaded

```typescript
// src/agent.ts (simplified)
const memory = await store.loadMemory(ctx.channel); // reads memory.md

const systemPrompt = `You are mom, a helpful assistant in a Slack workspace.

## Channel Memory
${memory}

## Instructions
- Use the bash tool to run commands.
- Update memory.md when you learn something new about this channel.
`;

this.agent.state.systemPrompt = systemPrompt;
```

---

## Live Memory Reload (`src/fs-watch.ts`)

A file-system watcher monitors `memory.md` for external changes. When an operator edits the file directly (e.g., to correct a mistake), the watcher fires and mom's next turn uses the updated content without a restart:

```typescript
// src/fs-watch.ts (simplified)
watch(memoryPath, () => {
  const updated = readFileSync(memoryPath, "utf-8");
  store.setMemoryCache(channelId, updated);
  log.info("Memory reloaded for channel", channelId);
});
```

---

## Check-Yourself Questions

1. Where is `memory.md` stored for a channel with ID `C08ABCDEF`?
2. At what point in a run does mom inject the memory file into the system prompt?
3. Can mom update `memory.md` during a run? How?
4. What happens if you delete `memory.md` manually?
5. What is the advantage of plain Markdown over a structured database for channel memory?
