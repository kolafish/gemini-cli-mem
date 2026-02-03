# A2A Server and WeChat Integration (Experimental)

This document explains what the Gemini CLI A2A server is, when to use it,
how to run it, and how to integrate it with WeChat via a gateway service.

## What the A2A server is
`packages/a2a-server/` exposes Gemini CLI as an Agent-to-Agent (A2A) protocol
server using `@a2a-js/sdk`. It wraps the CLI core agent loop and streams task
events to clients. It also provides a small set of management endpoints for
commands like `init`, `memory`, `extensions`, and `restore`.

Key implementation files:
- `packages/a2a-server/src/http/app.ts`: Express app, A2A wiring, extra routes.
- `packages/a2a-server/src/agent/executor.ts`: Agent executor loop.
- `packages/a2a-server/src/commands/*`: Server-side command implementations.

Status: experimental and subject to change.

## When to use it
Use the A2A server when you want Gemini CLI to run as a remote agent and be
called by another client or tool. Common scenarios:
- IDE integrations that talk to Gemini CLI over a stable protocol.
- Remote subagents for another Gemini CLI instance (A2A remote agents).
- Internal tooling that needs a streaming agent API for code tasks.

If you only need simple chat or single-turn text generation, consider calling
the Gemini API directly instead of running a full CLI agent server.

## How to run it (local or server)
Requirements:
- Node.js >= 20
- Auth configured via `GEMINI_API_KEY` or `USE_CCPA` (ADC)

Optional environment variables:
- `CODER_AGENT_PORT`: Port to bind. Defaults to an OS-assigned port; the repo
  uses 41242 in `npm run start:a2a-server`.
- `CODER_AGENT_WORKSPACE_PATH`: Workspace path for commands that need a repo.
- `GCS_BUCKET_NAME`: Enables GCS-backed task persistence.

Run from the repo root:
```bash
npm run start:a2a-server
```

The server logs the A2A Agent Card URL, typically:
```
http://localhost:41242/.well-known/agent-card.json
```

### Non-A2A management endpoints
The server also exposes several helper routes:
- `POST /executeCommand` for server-side commands (supports streaming).
- `GET /listCommands` to enumerate available commands.
- `POST /tasks` to create a task and return its id.
- `GET /tasks/metadata` and `GET /tasks/:taskId/metadata` for metadata.

These are implemented in `packages/a2a-server/src/http/app.ts`.

## WeChat integration overview
WeChat does not speak A2A directly. You need a gateway service that:
1. Receives WeChat callbacks (HTTP/HTTPS).
2. Converts each message into an A2A request to the A2A server.
3. Streams or buffers the A2A response.
4. Sends the final response back to WeChat.

### Recommended architecture
```
WeChat -> Your Gateway (HTTPS) -> A2A Server (internal) -> Gateway -> WeChat
```

Keep the A2A server private and only expose the gateway publicly.

### Practical constraints
- WeChat callback responses are time-limited. The gateway should acknowledge
  quickly and send the final response using the async messaging API.
- WeChat UI is text-first. The gateway should summarize or truncate long
  streaming output and send multiple messages if needed.

### Minimal gateway responsibilities
- Signature verification and optional message encryption.
- Session mapping: map WeChat user id to A2A `contextId` and `taskId`.
- Retry and idempotency handling to avoid duplicate responses.
- Safety controls: restrict tools and file access for public inputs.

### Example flow
1. Incoming WeChat message arrives at the gateway.
2. Gateway calls A2A `message/stream` (or equivalent) on the A2A server with:
   - Text content.
   - `AgentSettings` metadata (workspace path if needed).
3. Gateway aggregates streamed events into a final response.
4. Gateway sends the response to WeChat (async message API).

## Security and safety
The A2A server can execute tools with file and shell access. Do not expose it
directly to the internet. Put it behind a gateway with:
- Authentication and rate limiting.
- A restricted workspace path.
- A strict tool policy (or disable risky tools entirely).

## Related docs
- Remote agents (A2A): `docs/core/remote-agents.md`
- Subagents overview: `docs/core/subagents.md`
