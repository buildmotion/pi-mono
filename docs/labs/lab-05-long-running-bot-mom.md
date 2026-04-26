# Lab 05: Deploying a Self-Managing Slack Bot with `pi-mom`

`pi-mom` is an agent that lives in a Slack workspace. It reads and writes files, runs shell commands, and — when it encounters a task it cannot handle — installs new CLI tools and tries again. In this lab you wire it to a workspace, observe its first response, and watch it extend itself on demand.

## Learning Objectives

- Configure a Slack app with Socket Mode and the required OAuth scopes
- Set the environment variables `pi-mom` needs to operate
- Start `pi-mom` and confirm it is receiving Slack events
- Trigger a tool installation and verify it succeeds
- Inspect the per-channel memory file mom maintains between sessions
- Understand the security model of a self-extending, shell-capable bot

## Prerequisites

- A Slack workspace where you have admin rights (a personal free workspace is fine)
- Node.js 20 or later
- An Anthropic API key (`ANTHROPIC_API_KEY`)
- Basic familiarity with the Slack app configuration UI

```bash
mkdir lab-05 && cd lab-05
```

---

## Background

### Socket Mode vs HTTP Mode

Slack bots can receive events in two ways:

| Mode | How it works | Requirements |
|---|---|---|
| **HTTP mode** | Slack POSTs to a public URL you host | A public server with TLS |
| **Socket Mode** | Your process opens an outbound WebSocket to Slack | No public endpoint needed |

`pi-mom` uses Socket Mode. Your bot process initiates the connection outbound; Slack pushes events over that socket. This means you can run `pi-mom` on your laptop, behind NAT, without any port forwarding or ngrok tunnels.

### Per-Channel Memory

Every Slack channel gets its own memory file on disk, stored in your configured workspace directory. When `pi-mom` is mentioned in a channel, it:

1. Reads the channel's memory file (if it exists)
2. Includes the memory content in the system prompt
3. Responds
4. Updates the memory file with anything it decided to remember

This gives the bot persistent context across days and conversation threads without a database.

### Self-Extending Tooling

`pi-mom` ships with four built-in tools: `bash`, `read`, `write`, and `edit`. When a task requires additional capabilities (e.g., parsing JSON, converting files, calling a REST API), mom can:

1. Recognize the gap
2. Run `npm install -g <package>` or `apt install <package>` via the bash tool
3. Use the newly installed binary in subsequent tool calls

This loop happens autonomously. The user never needs to pre-configure tool lists.

---

## Step 1: Create a Slack App with Socket Mode Enabled

1. Go to [api.slack.com/apps](https://api.slack.com/apps) and click **Create New App → From scratch**
2. Name it `mom` (or any name you prefer) and select your workspace
3. In the left sidebar, click **Socket Mode** and toggle it **On**
4. You will be asked to generate an **App-Level Token** — name it `socket-token`, grant the `connections:write` scope, and save the token. It starts with `xapp-`
5. In the left sidebar, click **OAuth & Permissions**
6. Under **Bot Token Scopes**, add these scopes:

   | Scope | Why |
   |---|---|
   | `app_mentions:read` | Receive events when @mom is mentioned |
   | `chat:write` | Post messages |
   | `channels:history` | Read channel message history |
   | `files:read` | Access file attachments (optional, for file-aware responses) |

7. Click **Install to Workspace** and authorize the app
8. Copy the **Bot User OAuth Token** from the OAuth page — it starts with `xoxb-`
9. In **App Home → Show Tabs**, enable the **Messages Tab** and check **Allow users to send Slash commands and messages from the messages tab** so you can DM mom directly

---

## Step 2: Set Environment Variables

```bash
export SLACK_APP_TOKEN="xapp-1-..."   # App-level token from Socket Mode setup
export SLACK_BOT_TOKEN="xoxb-..."     # Bot OAuth token
export ANTHROPIC_API_KEY="sk-ant-..."
```

Add these to your shell profile (`~/.bashrc`, `~/.zshrc`) or a `.env` file in `lab-05/`. Do not commit the `.env` file.

Verify:

```bash
echo "App token: ${SLACK_APP_TOKEN:0:10}..."
echo "Bot token: ${SLACK_BOT_TOKEN:0:10}..."
```

---

## Step 3: Configure the Workspace Directory

The workspace directory is where `pi-mom` stores:
- Per-channel memory files (`<channel-id>.md`)
- Any files mom creates or downloads during tool use

```bash
mkdir workspace
export MOM_WORKSPACE="$(pwd)/workspace"
```

The workspace directory must be writable by the process running `pi-mom`. Treat it as a working directory — mom may create subdirectories and install files here.

> **Security note:** `pi-mom` has full read/write access to this directory and can run arbitrary shell commands. In Step 8 you will look at how to constrain this with a Docker sandbox.

---

## Step 4: Start `pi-mom`

```bash
npx @mariozechner/pi-mom \
  --workspace "$MOM_WORKSPACE"
```

You should see output like:

```
[mom] Connecting to Slack via Socket Mode...
[mom] Connected. Listening for mentions.
[mom] Workspace: /path/to/lab-05/workspace
```

If you see a `socket_mode_client_not_enabled` error, go back to Step 1 and verify Socket Mode is toggled on in the Slack app settings.

Leave this process running in a terminal. All subsequent steps interact via Slack.

---

## Step 5: Mention @mom in a Slack Channel and Observe the First Response

1. Open your Slack workspace
2. Invite `@mom` to any channel: `/invite @mom`
3. Send a message: `@mom What can you help me with?`

Within a few seconds, mom should reply with a description of its capabilities. Watch the terminal — you will see the agent loop events logged there.

Now try a task that uses the bash tool:

```
@mom What operating system is this running on? What is the hostname?
```

Mom will call the `bash` tool with `uname -a` and `hostname`, read the output, and reply. You should see `[tool] bash` logged in your terminal.

---

## Step 6: Ask Mom to Install a CLI Tool and Use It

Ask mom to install `jq` and use it to parse some JSON:

```
@mom Please install jq if it's not already available, then use it to extract
the "name" field from this JSON: {"name": "pi-mom", "version": "1.0", "active": true}
```

Mom will:
1. Run `which jq || apt install -y jq` (or equivalent) via bash
2. Confirm the installation
3. Run `echo '{"name":"pi-mom",...}' | jq '.name'`
4. Reply with `"pi-mom"`

Watch the terminal to see each tool call logged. This is the self-extending loop in action: mom identified a missing capability, installed the tool, and used it — all in one conversation turn.

Try a second installation:

```
@mom Can you install httpie and make a GET request to https://httpbin.org/get?
```

---

## Step 7: Inspect the Memory File Mom Created

After a few exchanges, look at the workspace directory:

```bash
ls workspace/
```

You should see a `.md` file named after the Slack channel ID (a string like `C07XXXXXXXX.md`). Read it:

```bash
cat workspace/C07XXXXXXXX.md
```

This file contains anything mom decided to remember from the conversation — the user's preferences, facts it learned, tool installations it performed, and context it wants to carry into future sessions.

You can edit this file directly to inject information mom should know:

```bash
cat >> workspace/C07XXXXXXXX.md << 'EOF'

## Instructor notes
This channel is used for Lab 05. The workspace machine runs Ubuntu 22.04.
The user's preferred output format is Markdown tables.
EOF
```

Mention `@mom` again and ask it to use this new context. It should incorporate the instructor notes into its reply.

---

## Step 8 (Stretch): Add a Docker Sandbox for Safer Execution

By default, mom's `bash` tool runs commands directly on the host system. In a shared or production environment this is a significant security risk.

Create a `Dockerfile` for a restricted execution environment:

```dockerfile
FROM ubuntu:22.04

# Minimal toolset — add more as needed
RUN apt-get update && apt-get install -y \
    curl \
    jq \
    python3 \
    nodejs \
    npm \
    && rm -rf /var/lib/apt/lists/*

# Non-root user for execution
RUN useradd -m sandbox
USER sandbox
WORKDIR /home/sandbox
```

Build the image:

```bash
docker build -t mom-sandbox .
```

Configure `pi-mom` to route bash commands through Docker:

```bash
npx @mariozechner/pi-mom \
  --workspace "$MOM_WORKSPACE" \
  --sandbox "docker run --rm -i --network=host mom-sandbox bash -c"
```

Now every `bash` tool call executes inside a fresh, short-lived container. The sandbox is destroyed after each command. Mom's file read/write tools still operate on the host workspace directory (mounted into the container if needed).

Test that sandbox isolation works:

```
@mom Please run: cat /etc/passwd | head -5
```

The output should be the container's `/etc/passwd`, not the host's.

---

## Expected Outcomes

By the end of this lab you should have:

- A running `pi-mom` instance connected to Slack via Socket Mode
- Confirmed that mom responds to @mentions and uses shell tools
- Triggered a self-install of a new CLI tool (`jq`) mid-conversation
- Inspected and manually edited a per-channel memory file
- (Stretch) Sandboxed tool execution inside a Docker container

---

## Check Yourself

1. What is the difference between the App-Level Token (`xapp-`) and the Bot Token (`xoxb-`)? Why does Socket Mode require both?
2. `pi-mom` can run arbitrary shell commands on the host machine. List three specific threat vectors a malicious Slack user could exploit if they can mention `@mom`.
3. The channel memory file is plain Markdown. What would happen if you deleted it? What would happen if you truncated it to 100 characters?
4. Mom installs tools with `npm install -g` or `apt install`. What permissions does the process need for each, and which one is safer to allow in a shared environment?
5. A user in the channel asks mom to `rm -rf ~/` using natural language (not a code block). Will mom comply? How would you add a policy layer to prevent destructive commands?

---

## What's Next

- **Lab 06** — [Self-Hosted Models with pi-pods](./lab-06-self-hosted-models-with-pods.md): replace the Anthropic API with a model you control on your own GPU hardware
- **`@mariozechner/pi-mom` README** — configuration options, tool extension API, and security guidance
- **`packages/mom/src/`** — source for the memory system, tool definitions, and Slack event handling
