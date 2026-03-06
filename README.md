# Symphony on jaato

An implementation of the [Symphony spec](https://github.com/openai/symphony) using jaato-sdk. Turns Linear issues into autonomous coding agent runs — no polling, no daemon scaffold, just a profile.

## Architecture

```
Linear ──webhook POST──► jaato webhook plugin ──► orchestrator agent
                                                       │
                                            spawn_subagent(profile="symphony-worker")
                                                       │
                                              ┌────────┴────────┐
                                              │    per-issue    │
                                              │  coding agent   │
                                              │                 │
                                              │  clone → fix →  │
                                              │  test → push    │
                                              └────────┬────────┘
                                                       │
                                         call_service ──► Linear API
                                         (comment, state transition)
```

**Two profiles, one prompt:**

| File | Role |
|------|------|
| `.jaato/profiles/symphony-orchestrator.json` | Long-running daemon — listens for webhooks, dispatches workers |
| `.jaato/profiles/symphony-worker.json` | Per-issue coding agent — clones, fixes, tests, pushes |
| `.jaato/prompts/symphony-orchestrator.md` | Full operational prompt loaded at startup |
| `.jaato/webhook.json` | Linear webhook route configuration |

## Setup

### 1. Environment

```bash
cp .env.example .env
# Fill in LINEAR_API_KEY, LINEAR_WEBHOOK_SECRET, and your model provider credentials
```

### 2. Linear Webhook

Create a webhook in Linear at **Settings > API > Webhooks**:

- **URL**: Your public endpoint followed by `/webhook/linear` (see below)
- **Events**: Issues (create, update, remove)
- **Secret**: Copy the signing secret into `LINEAR_WEBHOOK_SECRET`

**Production:** Use your server's public URL directly, e.g. `https://myserver.example.com:9100/webhook/linear`

**Local development:** Expose port 9100 via a tunnel, then use the tunnel URL as the webhook endpoint:
```bash
# ngrok
ngrok http 9100
# Webhook URL: https://a1b2c3d4.ngrok-free.app/webhook/linear

# or cloudflare tunnel
cloudflared tunnel --url http://localhost:9100
# Webhook URL: https://some-random-words.trycloudflare.com/webhook/linear
```

Note: the tunnel URL has no port — the tunnel provider handles TLS and routing to your local port 9100.

### 3. Linear Labels

Create a label called `symphony` (or your preferred trigger label) in your Linear project. Only issues with this label will be picked up by the orchestrator.

### 4. Run

Start the jaato server, then create a session with the orchestrator profile:

```bash
# Source environment variables first
source .env

# Start the server
jaato-server

# In another terminal (also source .env)
source .env
jaato --profile symphony-orchestrator -i "%symphony-orchestrator project_slug=MY-PROJECT workspace_root=~/code/workspaces max_concurrent_agents=5"
```

The orchestrator will:
1. Verify the Linear API connection
2. Start the webhook listener on port 9100
3. Subscribe to events on the shared TaskEventBus — then wait for issues (no polling)

## How It Works

1. A team member creates or labels a Linear issue with `symphony`
2. Linear fires a webhook to the orchestrator
3. The orchestrator checks eligibility (correct label, not already in progress, under concurrency limit)
4. If eligible: moves issue to "In Progress", spawns a `symphony-worker` subagent with the issue details
5. The worker clones the repo, creates a branch, implements the fix, runs tests, pushes
6. On completion: orchestrator comments on the issue with the result and transitions the state

## Configuration

### Concurrency

Set `max_concurrent_agents` when starting the orchestrator. Default is 5.

### Worker Capabilities

Edit `.jaato/profiles/symphony-worker.json` to change what tools the coding agent has access to. The default set includes CLI, file editing, LSP, AST search, and filesystem query.

### Webhook Port

Change `port` in `.jaato/webhook.json` or in the orchestrator profile's `plugin_configs.webhook.port`.
