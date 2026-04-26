# `@mariozechner/pi-mom` — Self-Managing Slack Bot

## What is it?

`mom` (Master of Mischief) is a Node.js process that connects to Slack via Socket Mode and responds to messages using an LLM agent backed by Anthropic's Claude. What makes it different from ordinary bots is that mom manages herself: she installs the tools she needs, writes her own helper scripts, remembers context between sessions, and maintains a persistent workspace — all autonomously.

**npm package:** `@mariozechner/pi-mom`  
**CLI binary:** `mom`  
**Version:** 0.70.2  
**Node requirement:** >= 20.0.0  
**License:** MIT

---

## The Problem It Solves

Most AI Slack bots are narrow: they answer questions in a read-only way, or they execute a fixed set of pre-wired integrations. Getting them to do real work — run a build, grep a log, write a script, remember what you told them last week — requires significant custom DevOps.

`mom` inverts this: she starts with nothing but `bash`, `read`, and `write` tools and builds everything else herself. Need to check your email? She installs `himalaya`. Need to query your database? She writes the SQL script herself and saves it as a reusable tool. Need a reminder in three hours? She schedules it with the built-in events system.

---

## Audience

- Engineering teams that want an always-on AI assistant in their Slack workspace
- DevOps teams that want to automate runbooks and on-call queries through Slack
- Small teams that want a self-extending bot without hiring a platform engineer to maintain it

---

## How It Works: The Full Flow

`mom` runs as a persistent Node.js process on your host machine (or in a VM/container). Here is what happens from first message to response:

### 1. Startup

```
mom --sandbox=docker:mom-sandbox ./data
```

- Loads the config from `./data/` (the workspace root)
- Connects to Slack via Socket Mode using `SLACK_APP_TOKEN`
- Registers event listeners for `app_mention`, `message.channels`, `message.groups`, `message.im`

### 2. Message Arrives

When a message arrives in a channel where mom is present:

1. The raw message is **appended to `<channel>/log.jsonl`** — the permanent, searchable history file.
2. If the message has file attachments, they are downloaded and saved to `<channel>/attachments/`.
3. **No LLM call happens yet.** Log-only messages do not consume tokens.

### 3. @mention or DM triggers a response

When you `@mention` mom or DM her:

1. **Context sync:** Unseen messages from `log.jsonl` are appended to `context.jsonl`. `context.jsonl` is what the LLM actually sees.
2. **Memory load:** `MEMORY.md` (global) and `<channel>/MEMORY.md` (channel-specific) are injected into the system prompt. These are markdown files mom can write to herself to persist facts across sessions.
3. **Agent invocation:** The `pi-agent-core` `Agent` runs with mom's tools:
   - `bash` — execute any shell command
   - `read` — read a file
   - `write` — write a file
   - `list` — list directory contents
   - `glob` / `rg` — find files by name or content
4. **Tool execution:** Mom executes tools in a sandbox (Docker or host), reads results, and loops back to the LLM until she has a complete answer.
5. **Response to Slack:** The main answer goes to the channel. Tool call details (commands run, file contents read) go to a Slack thread to keep the channel clean.

### 4. Context Management (compaction)

When the accumulated context exceeds the model's token limit:
- Recent messages and tool results are kept verbatim.
- Older entries are summarized by the LLM into a condensed block.
- For history older than context, mom can search `log.jsonl` directly with `bash grep`.

---

## Self-Management: How Mom Builds Her Own Tools

The key differentiator is the `bash` tool with full shell access. Mom can:

1. **Install packages:** `apk add git`, `npm install -g jq`, etc.
2. **Write scripts:** Create a shell or Python script, make it executable, store it in `<channel>/skills/`.
3. **Invoke scripts:** On the next request, use the script she wrote earlier.

Example: "Set up a tool to check our GitHub CI status."

Mom will:
1. Install the `gh` CLI if not present.
2. Write a script `check-ci.sh` that calls `gh run list --repo=…`.
3. Store it in her skills directory.
4. Use it immediately to answer the question.
5. Remember this skill in `MEMORY.md` so she uses it in future sessions.

This is the "self-managing" property: the operator never has to pre-install anything.

---

## Sandbox Modes

Because `bash` is unrestricted, running mom on the host machine is a significant security risk. The recommended approach:

```bash
# Create a Docker container (lightweight Alpine) as the sandbox
docker run -d --name mom-sandbox -v $(pwd)/data:/workspace alpine:latest tail -f /dev/null

# Run mom in Docker mode
mom --sandbox=docker:mom-sandbox ./data
```

In Docker mode, all `bash` tool executions are piped into the container via `docker exec`. The host machine's filesystem is not directly accessible (only the mounted `./data` volume).

`--sandbox=host` is available for trusted environments but is explicitly not recommended.

---

## Events System

Mom includes a cron-based events system (`croner`) for scheduled tasks:

- **Reminders:** "Remind me in 3 hours to check the deployment."
- **Periodic tasks:** "Every Monday morning, summarize last week's pull requests."

Events are stored in the workspace and survive restarts.

---

## Artifacts Server

Mom can generate HTML/JS visualizations (charts, dashboards) and serve them publicly via a built-in artifacts server with live reload. Useful for "show me a graph of our API error rates."

---

## Workspace Structure

```
./data/
├── MEMORY.md                   Global memory (facts mom remembers across all channels)
├── <channel-id>/
│   ├── log.jsonl               Permanent message log (append-only)
│   ├── context.jsonl           LLM context (recent messages + tool results)
│   ├── MEMORY.md               Channel-specific memory
│   ├── attachments/            Downloaded file attachments
│   └── skills/                 Scripts mom wrote for this channel
└── …
```

You have full visibility into everything: logs, memory, scripts. Nothing is hidden.

---

## Source Layout

```
packages/mom/src/
├── main.ts            Entry point; CLI arg parsing, startup
├── agent.ts           LLM agent setup and prompt loop
├── slack.ts           Slack Socket Mode client, message routing
├── context.ts         context.jsonl management and compaction
├── log.ts             log.jsonl append and search
├── store.ts           Workspace directory management
├── sandbox.ts         Docker / host sandbox abstraction
├── events.ts          Cron-based events (reminders, periodic tasks)
├── fs-watch.ts        Watch workspace for external changes
├── download.ts        Slack attachment downloader
└── tools/             bash, read, write, list, glob, rg tool definitions
```

---

## Authentication

Mom needs Anthropic API access. Two options:

**Option A — Environment variable (API key):**
```bash
export ANTHROPIC_API_KEY=sk-ant-...
```

**Option B — OAuth (Claude Pro/Max subscription):**
```bash
npx @mariozechner/pi-coding-agent   # run pi interactively
/login                               # authenticate via browser
# Then link the token:
ln -s ~/.pi/agent/auth.json ~/.pi/mom/auth.json
```

---

## Key Dependencies

| Dependency | Purpose |
|-----------|---------|
| `@mariozechner/pi-agent-core` | LLM agent loop and event handling |
| `@mariozechner/pi-ai` | Anthropic API streaming |
| `@mariozechner/pi-coding-agent` | Reuses coding agent's file tools |
| `@slack/socket-mode` | Slack real-time Socket Mode connection |
| `@slack/web-api` | Slack REST API (posting messages, uploading files) |
| `croner` | Cron scheduling for events |
| `@anthropic-ai/sandbox-runtime` | Sandbox runtime utilities |

---

## Pros and Cons

**Pros**
- Self-managing: no tool pre-configuration required
- Per-channel isolation with independent memory and workspace
- Permanent searchable history (`log.jsonl`) independent of Slack's retention limits
- Docker sandbox for safe bash execution
- Artifacts server for inline visualizations
- Built-in scheduled events/reminders
- Full transparency — all files, memory, and logs are inspectable

**Cons**
- LLM provider locked to Anthropic by default (Claude)
- Requires Slack App setup with Socket Mode (cannot use webhook-based bots)
- `--sandbox=host` is genuinely dangerous — `bash` has no restrictions
- Docker sandbox adds container-management overhead
- No web UI; management is entirely via Slack messages

---

## How to Use It (Junior Developer Walkthrough)

Think of mom as a junior DevOps engineer who lives in Slack. She has a laptop (the Docker container), a notebook (MEMORY.md), and can run any shell command. You @mention her and describe what you need; she figures out how to do it.

1. **Set up the Slack app** at https://api.slack.com/apps:
   - Enable Socket Mode
   - Generate an App-Level Token (`xapp-...`)
   - Add Bot Token Scopes and subscribe to events (see README for full list)
   - Install to workspace, copy Bot Token (`xoxb-...`)

2. **Set environment variables:**
   ```bash
   export MOM_SLACK_APP_TOKEN=xapp-...
   export MOM_SLACK_BOT_TOKEN=xoxb-...
   export ANTHROPIC_API_KEY=sk-ant-...
   ```

3. **Create the sandbox container:**
   ```bash
   docker run -d --name mom-sandbox -v $(pwd)/data:/workspace alpine:latest tail -f /dev/null
   ```

4. **Start mom:**
   ```bash
   npm install -g @mariozechner/pi-mom
   mom --sandbox=docker:mom-sandbox ./data
   ```

5. **Add mom to a Slack channel**, then @mention her:
   ```
   @mom what's our git log for the last 24 hours?
   ```
   She will install `git` if needed and return the log.

6. **Give her a persistent task:**
   ```
   @mom every morning at 9am, post our open GitHub issues count
   ```
   She sets up a cron job and remembers it in MEMORY.md.
