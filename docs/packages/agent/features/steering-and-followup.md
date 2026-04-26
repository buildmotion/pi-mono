# Steering and Follow-up

While an agent is running, you often need to send it new information or queue up the next task. `pi-agent-core` provides two complementary mechanisms: **steering** (mid-run injection) and **follow-up** (post-run queueing).

See also: [agent-loop.md](./agent-loop.md) for where these hooks fit in the loop.

---

## The difference

| | Steering | Follow-up |
|---|---|---|
| **When injected** | After the current tool batch, before the next LLM call | After the agent would otherwise stop (no more tools, no steering) |
| **Interrupts the run?** | No — current tool calls finish first | No — the full run finishes first |
| **Typical use case** | Course-correct a running agent | Queue the next user message |

---

## Steering

### What it means

A **steering message** is injected into the conversation while the agent is still running, between tool batches. The agent finishes executing all tools from the current assistant message, then polls for steering messages before making the next LLM call. If any are available, they are prepended to the next turn's context.

The agent is not interrupted. Steering redirects, it does not stop.

### API

```typescript
// Queue a steering message at any time while the agent is running
agent.steer({
  role: "user",
  content: [{ type: "text", text: "Focus on the authentication module, ignore the rest." }],
  timestamp: Date.now(),
});
```

`steer()` enqueues a message in the internal `PendingMessageQueue` (steering). The queue is drained by the loop's `getSteeringMessages` callback after each tool batch.

### Queue modes

The steering queue has two drain modes:

```typescript
// Change mode at any time
agent.steeringMode = "one-at-a-time"; // default: drain one message per poll
agent.steeringMode = "all";            // drain all queued messages in one poll
```

In `"one-at-a-time"` mode, each poll returns at most one message, even if multiple are queued. The loop will poll again on the next tool batch. In `"all"` mode, all pending messages are injected at once.

Set the mode in the constructor:

```typescript
const agent = new Agent({ steeringMode: "all" });
```

### Use case: progress injection

Some agent designs have a monitoring layer that reads tool results and sends guidance:

```typescript
agent.subscribe(async (event) => {
  if (event.type === "tool_execution_end" && event.toolName === "run_tests") {
    const result = event.result;
    if (result.details.failCount > 10) {
      agent.steer({
        role: "user",
        content: [{ type: "text", text: "Too many failures. Stop and summarise what you know so far." }],
        timestamp: Date.now(),
      });
    }
  }
});
```

The steering message will be picked up after the current tool batch completes and before the next LLM call.

### Use case: multi-step plan injection

```typescript
// Before starting the run, pre-load a plan as steering messages
agent.steer(buildPlanMessage("Step 1: gather requirements"));
agent.steer(buildPlanMessage("Step 2: write tests"));
agent.steer(buildPlanMessage("Step 3: implement"));
agent.steeringMode = "one-at-a-time"; // inject one step at a time

await agent.prompt("Begin the project.");
// The agent will receive one plan step between each tool batch
```

---

## Follow-up

### What it means

A **follow-up message** is queued for processing **after** the agent would otherwise stop. The loop checks for follow-up messages when there are no more tool calls and no steering messages. If follow-up messages are found, they are added to the context and the loop runs another turn as if the user had sent a new message.

This is the correct mechanism for handling user input that arrives while the agent is busy. Instead of starting a new run (which would conflict with the active one), the message waits in the queue and is processed as part of the same run.

### API

```typescript
// Queue a follow-up message to be processed after the current work finishes
agent.followUp({
  role: "user",
  content: [{ type: "text", text: "Now explain what you did in plain English." }],
  timestamp: Date.now(),
});
```

`followUp()` enqueues a message in the internal `PendingMessageQueue` (follow-up). The queue is drained by the loop's `getFollowUpMessages` callback when the agent would otherwise emit `agent_end`.

### Queue modes

Follow-up has the same two modes as steering:

```typescript
agent.followUpMode = "one-at-a-time"; // default
agent.followUpMode = "all";
```

```typescript
const agent = new Agent({ followUpMode: "all" });
```

### Use case: user types while agent is working

```typescript
// User sends a message while the agent is busy
chatInput.on("submit", (text) => {
  if (agent.state.isStreaming) {
    // Cannot prompt() — use followUp() instead
    agent.followUp({
      role: "user",
      content: [{ type: "text", text }],
      timestamp: Date.now(),
    });
  } else {
    agent.prompt(text);
  }
});
```

The follow-up message will be processed once the current run completes, as part of the same logical session, without starting a second concurrent run.

### Use case: chaining tasks

```typescript
// Pre-queue a sequence of tasks
agent.followUp(buildMessage("Summarise the findings."));
agent.followUp(buildMessage("Generate a report."));
agent.followUp(buildMessage("Send the report."));
agent.followUpMode = "one-at-a-time"; // process one at a time (default)

await agent.prompt("Analyse the codebase.");
// The agent will work through all four messages sequentially in one run
```

---

## `continue()` and queued messages

`agent.continue()` replays the current transcript from where it left off. If the last message is an assistant message, `continue()` first checks the steering queue, then the follow-up queue:

```typescript
// agent.ts — Agent.continue()
async continue(): Promise<void> {
  const lastMessage = this._state.messages.at(-1);
  if (lastMessage?.role === "assistant") {
    const steeringMessages = this.steeringQueue.drain();
    if (steeringMessages.length > 0) {
      await this.runPromptMessages(steeringMessages, { skipInitialSteeringPoll: true });
      return;
    }
    const followUpMessages = this.followUpQueue.drain();
    if (followUpMessages.length > 0) {
      await this.runPromptMessages(followUpMessages);
      return;
    }
    throw new Error("Cannot continue from message role: assistant");
  }
  await this.runContinuation();
}
```

This makes `continue()` the natural way to resume an agent that has stopped with queued work remaining.

---

## Clearing queues

```typescript
agent.clearSteeringQueue();   // remove all steering messages
agent.clearFollowUpQueue();   // remove all follow-up messages
agent.clearAllQueues();        // both

// Check before deciding whether to start a new run vs use followUp
if (agent.hasQueuedMessages()) {
  console.log("Messages pending, agent will continue.");
}
```

`reset()` also clears both queues:

```typescript
agent.reset(); // clears messages, streaming state, and both queues
```

---

## Event sequence with steering

```
agent.prompt("Analyse errors.")
  agent_start
  turn_start
    message_start / message_end (user)
    message_start (assistant — requests tools)
    message_update ×N
    message_end
    tool_execution_start (run_tests)
    tool_execution_end
    message_start / message_end (tool result)
  turn_end
  ← getSteeringMessages() called → steering message available
  turn_start
    message_start / message_end (steering: "Focus on auth module")
    message_start (assistant — second response)
    message_update ×N
    message_end
  turn_end
  ← getSteeringMessages() called → empty
  ← getFollowUpMessages() called → empty
  agent_end
```

## Event sequence with follow-up

```
agent.followUp("Now write the report.")
agent.prompt("Analyse errors.")
  agent_start
  turn_start
    message_start / message_end (user)
    message_start (assistant — final answer)
    message_update ×N
    message_end
  turn_end
  ← getSteeringMessages() → empty
  ← no more tool calls, inner loop exits
  ← getFollowUpMessages() → ["Now write the report."]
  turn_start
    message_start / message_end (follow-up user message)
    message_start (assistant)
    message_update ×N
    message_end
  turn_end
  ← getFollowUpMessages() → empty
  agent_end
```
