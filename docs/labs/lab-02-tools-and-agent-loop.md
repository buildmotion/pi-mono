# Lab 02: Defining Tools and Running the Agent Loop with `pi-agent-core`

The streaming API from Lab 01 gives you raw LLM output. Tools give the model the ability to act — to call code you write, read data from external systems, and return structured results back into the conversation. This lab wires tools to an agent loop and makes the control flow visible.

## Learning Objectives

- Define an `AgentTool` with a TypeBox parameter schema
- Create an `Agent` instance and attach tools to it
- Call `agent.prompt()` and handle the full `AgentEvent` sequence
- Intercept tool calls before execution with a `beforeToolCall` hook
- Reason about parallel vs sequential tool execution

## Prerequisites

- Lab 01 complete — you should be comfortable with `streamSimple()` and `Context`
- Node.js 20 or later
- An API key exported in your shell (`ANTHROPIC_API_KEY` or equivalent)

```bash
mkdir lab-02 && cd lab-02
npm init -y
npm install @mariozechner/pi-ai @mariozechner/pi-agent-core @sinclair/typebox tsx typescript
```

---

## Background

### Tool Schemas with TypeBox

LLM tool calls require a JSON Schema description of the parameters the model should supply. TypeBox generates JSON Schema from TypeScript types at runtime — no separate schema file, no drift between types and runtime validation.

```typescript
import { Type } from "@sinclair/typebox";

const schema = Type.Object({
  city: Type.String({ description: "City name" }),
  units: Type.Union([Type.Literal("celsius"), Type.Literal("fahrenheit")]),
});
// schema is valid JSON Schema and a TypeScript type simultaneously
```

### The Agent Loop Cycle

A single call to `agent.prompt("your message")` triggers this cycle:

```
user message
    │
    ▼
┌───────────────────────────────┐
│  LLM generates response       │  ← streaming
│  (text and/or tool calls)     │
└───────────┬───────────────────┘
            │ tool calls present?
            ├── no → emit agent_end, stop
            │
            ▼
┌───────────────────────────────┐
│  Execute each tool call       │  ← your execute() runs here
│  (parallel or sequential)     │
└───────────┬───────────────────┘
            │
            ▼
┌───────────────────────────────┐
│  Append ToolResultMessages    │
│  to context, call LLM again   │  ← next iteration
└───────────────────────────────┘
```

The loop continues until the LLM produces a turn with no tool calls. The agent handles all of this automatically; your job is to define tools and subscribe to events.

---

## Step 1: Define an AgentTool with TypeBox

Create `tools.ts`:

```typescript
import { Type } from "@sinclair/typebox";
import type { AgentTool } from "@mariozechner/pi-agent-core";

export const weatherTool: AgentTool = {
  name: "get_weather",
  label: "Get Weather",
  description:
    "Returns current weather conditions for a given city. " +
    "Use this when the user asks about weather or temperature.",
  parameters: Type.Object({
    city: Type.String({ description: "The city to get weather for" }),
    units: Type.Optional(
      Type.Union([Type.Literal("celsius"), Type.Literal("fahrenheit")], {
        description: "Temperature unit, defaults to celsius",
      })
    ),
  }),
  execute: async (id, params) => {
    // In a real implementation you would call a weather API here.
    // For this lab we return deterministic fake data so the loop is testable
    // without external dependencies.
    const unit = params.units ?? "celsius";
    const temp = unit === "celsius" ? 18 : 64;

    console.log(`  [tool] get_weather called for "${params.city}" (id=${id})`);

    return {
      content: [
        {
          type: "text",
          text: `Weather in ${params.city}: partly cloudy, ${temp}°${unit === "celsius" ? "C" : "F"}, wind 12 km/h`,
        },
      ],
      details: {
        city: params.city,
        temperature: temp,
        unit,
        condition: "partly cloudy",
      },
    };
  },
};
```

Three things to notice:

1. `parameters` is a TypeBox `TObject`. The agent serialises this to JSON Schema and sends it with each LLM request so the model knows what arguments to produce.
2. `execute` is async. It receives a unique call `id` (useful for correlating events) and the typed `params` object. Return `content` (what the model sees) and `details` (metadata for your application).
3. The fake implementation keeps this lab self-contained. Replace the body with a real HTTP call when you are ready.

---

## Step 2: Create an Agent and Attach the Tool

Create `agent.ts`:

```typescript
import { models } from "@mariozechner/pi-ai";
import { Agent } from "@mariozechner/pi-agent-core";
import { weatherTool } from "./tools.js";

const model = models.find(
  (m) => m.provider === "anthropic" && m.id.includes("claude-3-5")
);
if (!model) throw new Error("Model not found");

const agent = new Agent({
  model,
  systemPrompt:
    "You are a helpful assistant. When the user asks about weather, " +
    "always call the get_weather tool rather than guessing.",
  tools: [weatherTool],
});

export { agent };
```

The `Agent` constructor wires the model, system prompt, and tool list into an internal `Context` that it manages across turns. You do not need to maintain the messages array yourself — the agent does it.

---

## Step 3: Call `agent.prompt()` and Subscribe to Events

Create `index.ts`:

```typescript
import { agent } from "./agent.js";
import type { AgentEvent } from "@mariozechner/pi-agent-core";

function handleEvent(event: AgentEvent): void {
  switch (event.type) {
    case "agent_start":
      console.log(`\n[agent_start] turn ${event.turn}`);
      break;

    case "turn_start":
      console.log(`[turn_start]`);
      break;

    case "message_update":
      // Called repeatedly as text streams in; event.message holds the partial reply
      if (event.message.content) {
        process.stdout.write("\r" + event.message.content.slice(0, 80));
      }
      break;

    case "tool_execution_start":
      console.log(`\n[tool_execution_start] ${event.toolCall.name}(${JSON.stringify(event.toolCall.input)})`);
      break;

    case "tool_execution_end":
      console.log(`[tool_execution_end] ${event.toolCall.name} completed`);
      break;

    case "agent_end":
      console.log(`\n[agent_end] stop=${event.stopReason}`);
      console.log(`  Final reply: ${event.message.content}`);
      break;
  }
}

// Subscribe before prompting so you catch all events including agent_start
const unsubscribe = agent.subscribe(handleEvent);

await agent.prompt("What is the weather like in Tokyo right now?");

unsubscribe();
```

Run it:

```bash
npx tsx index.ts
```

You should see the event sequence: `agent_start` → `turn_start` → `message_update` bursts → `tool_execution_start` → `tool_execution_end` → `turn_start` (second LLM call) → `message_update` → `agent_end`.

The model called your tool automatically because it was in the system prompt and the question made it relevant.

---

## Step 4: Print Each AgentEvent Type as It Arrives

Modify the handler to be exhaustive and formatted so you can study the sequence clearly:

```typescript
function verboseHandler(event: AgentEvent): void {
  const ts = new Date().toISOString().slice(11, 23); // HH:MM:SS.mmm

  switch (event.type) {
    case "agent_start":
      console.log(`${ts} AGENT_START   turn=${event.turn}`);
      break;

    case "turn_start":
      console.log(`${ts} TURN_START`);
      break;

    case "message_update":
      process.stdout.write(".");
      break;

    case "tool_execution_start":
      process.stdout.write("\n");
      console.log(
        `${ts} TOOL_START    name=${event.toolCall.name} id=${event.toolCall.id}`
      );
      console.log(`           input=${JSON.stringify(event.toolCall.input)}`);
      break;

    case "tool_execution_end":
      console.log(
        `${ts} TOOL_END      name=${event.toolCall.name} id=${event.toolCall.id}`
      );
      break;

    case "agent_end":
      process.stdout.write("\n");
      console.log(`${ts} AGENT_END     stop=${event.stopReason}`);
      break;
  }
}
```

The dots during `message_update` give you a visual sense of streaming throughput without flooding the terminal.

---

## Step 5: Add a `beforeToolCall` Hook for Logging and Control

The `Agent` constructor accepts a `beforeToolCall` callback. Use it to log, audit, or even block tool calls based on application policy:

```typescript
const agent = new Agent({
  model,
  systemPrompt:
    "You are a helpful assistant. When the user asks about weather, " +
    "always call the get_weather tool rather than guessing.",
  tools: [weatherTool],
  beforeToolCall: async (toolCall) => {
    console.log(`\n[beforeToolCall] About to call: ${toolCall.name}`);
    console.log(`  Arguments: ${JSON.stringify(toolCall.input, null, 2)}`);

    // Return "proceed" to allow, "cancel" to skip, or a replacement result
    return "proceed";
  },
});
```

Try returning `"cancel"` and observe what happens to the agent loop — the model receives no tool result and must respond without that information.

---

## Step 6: Observe Parallel vs Sequential Tool Execution

Add a second tool to see how the agent handles multiple simultaneous calls:

```typescript
export const timezoneTool: AgentTool = {
  name: "get_timezone",
  label: "Get Timezone",
  description: "Returns the current local time for a given city.",
  parameters: Type.Object({
    city: Type.String({ description: "City name" }),
  }),
  execute: async (id, params) => {
    console.log(`  [tool] get_timezone called for "${params.city}" (id=${id})`);
    // Simulate async work
    await new Promise((r) => setTimeout(r, 200));
    return {
      content: [{ type: "text", text: `Current time in ${params.city}: 14:32 JST` }],
      details: { city: params.city, time: "14:32", timezone: "JST" },
    };
  },
};
```

Update your agent to include both tools and ask a question that requires both:

```typescript
const agent = new Agent({
  model,
  systemPrompt: "You are a helpful assistant with weather and timezone tools.",
  tools: [weatherTool, timezoneTool],
});

await agent.prompt(
  "What is the weather and current local time in Tokyo and London?"
);
```

Watch the `tool_execution_start` and `tool_execution_end` events. If the model emits all four tool calls in one turn, the agent may execute them in parallel. Add `console.time` / `console.timeEnd` around the `execute` bodies to measure whether they overlap.

---

## Step 7 (Stretch): Chain Two Tools in Sequence

Design a prompt where the output of tool A must be the input to tool B. For example:

```typescript
export const convertTemperatureTool: AgentTool = {
  name: "convert_temperature",
  label: "Convert Temperature",
  description: "Converts a temperature value between celsius and fahrenheit.",
  parameters: Type.Object({
    value: Type.Number({ description: "The temperature value to convert" }),
    from: Type.Union([Type.Literal("celsius"), Type.Literal("fahrenheit")]),
    to: Type.Union([Type.Literal("celsius"), Type.Literal("fahrenheit")]),
  }),
  execute: async (_id, params) => {
    let result: number;
    if (params.from === "celsius" && params.to === "fahrenheit") {
      result = (params.value * 9) / 5 + 32;
    } else if (params.from === "fahrenheit" && params.to === "celsius") {
      result = ((params.value - 32) * 5) / 9;
    } else {
      result = params.value;
    }
    return {
      content: [{ type: "text", text: `${params.value}°${params.from} = ${result.toFixed(1)}°${params.to}` }],
      details: { input: params.value, output: result, from: params.from, to: params.to },
    };
  },
};
```

Then ask: `"What is the weather in Berlin in fahrenheit?"`. The model should first call `get_weather` (which returns celsius), then call `convert_temperature` with the result. Count the number of LLM turns from the event log.

---

## Expected Outcomes

By the end of this lab you should have:

- A working tool defined with a TypeBox schema and an async `execute` function
- An agent that calls the tool automatically when the user prompt warrants it
- A clear view of the `AgentEvent` sequence across a multi-turn tool-calling loop
- Experience intercepting tool calls with `beforeToolCall`
- An intuition for when tools execute in parallel vs in sequence

---

## Check Yourself

1. What JSON Schema does TypeBox generate for the `weatherTool` parameters? Print `JSON.stringify(weatherTool.parameters, null, 2)` and inspect it.
2. If `execute` throws an exception, what does the agent do? Test it by throwing inside `get_weather`.
3. How many LLM API calls does a single `agent.prompt()` make when the model requests two tools? Instrument the code to count them.
4. What is the difference between `beforeToolCall` returning `"proceed"` and returning `"cancel"`? What does the model see in each case?
5. The `details` field in the tool result is not sent to the model — only `content` is. What is `details` for?

---

## What's Next

- **Lab 03** — [Building a TUI Agent](./lab-03-building-a-tui-agent.md): render the agent loop in a terminal UI
- **`@mariozechner/pi-agent-core` README** — full `Agent` constructor options, `steer()`, `followUp()`, and `abort()`
- **`packages/agent/src/types.ts`** — canonical definitions for `AgentTool`, `AgentEvent`, and `AgentState`
