# Gemini CLI Codebase Overview

## What this repo is
Gemini CLI is an open-source terminal AI agent for Google's Gemini models. It provides an interactive TUI (and a non-interactive mode) with built-in tools (file ops, shell, search, web fetch), MCP integrations, and auth options (OAuth, API key, Vertex AI).

## High-level structure
- `packages/cli/`: User-facing CLI and TUI (Ink/React). Handles argument parsing, config/settings, auth, UI state, extensions, and tool wiring. Main entrypoints:
  - `packages/cli/src/gemini.tsx`: interactive TUI runtime.
  - `packages/cli/src/nonInteractiveCli.ts`: `gemini -p` style non-interactive runs.
- `packages/core/`: Shared core library used by the CLI. Provides Gemini client abstraction, prompt/tool orchestration, MCP client/server utilities, telemetry, file discovery, shell execution, and workspace context helpers. (See the core deep dive below.)
- `packages/a2a-server/`: Experimental A2A server implementation.
- `packages/vscode-ide-companion/`: VS Code extension that feeds editor context and diffs into Gemini CLI.
- `packages/test-utils/`: Shared testing helpers.
- `integration-tests/`: Vitest-based integration tests (including sandboxed variants).
- `scripts/`: Build, bundle, release, and utility scripts.
- `bundle/`: Built CLI artifacts used by the root package `bin` entry.

## Typical runtime flow (interactive)
1. CLI parses args + loads settings.
2. Auth is validated (OAuth / API key / Vertex).
3. Core is initialized (Gemini client, tool registry, MCP connections).
4. TUI renders and drives the conversation loop with tool execution.

## Core package deep dive (`packages/core`)
The core package is the backend engine that turns a user prompt into model
calls, tool executions, and structured outputs. It is organized as several
subsystems that the CLI wires together.

### Core execution pipeline (conceptual)
1. **Config + model selection** from `config/`, `availability/`, and
   `routing/` determine the active model, fallbacks, and routing strategy.
2. **Prompt + request assembly** in `core/` builds the model request, applies
   prompt templates, tracks token limits, and manages the conversation turn.
3. **Tool orchestration** via `core/coreToolScheduler.ts` and `scheduler/`
   interprets tool calls, applies confirmation/policy checks, and executes
   tools through the tool registry.
4. **Output formatting + telemetry** via `output/` and `telemetry/` serialize
   results, track events, and record sessions.

### Subsystem map (by directory)
- `core/`: Gemini client abstractions, content generators, chat orchestration,
  request/turn handling, token limits, retries, and logging/recording wrappers.
- `tools/`: Built-in tool definitions and registry (read/write/edit files,
  ls/grep/ripgrep/glob, shell, web fetch/search, memory, MCP, write-todos).
- `scheduler/` + `confirmation-bus/`: Tool execution state machine, policy
  checks, and confirmation workflows.
- `policy/`: Policy engine and TOML config loader for tool safety and approval
  rules.
- `availability/` + `fallback/` + `routing/`: Model availability checks,
  fallback handling, and model routing strategy selection.
- `mcp/`: MCP auth providers, token storage, and OAuth utilities; used by
  MCP tool wiring in `tools/`.
- `agents/` + `skills/`: Sub-agent framework and skill loading/activation used
  for specialized tasks.
- `hooks/`: Hook registry/execution for intercepting or augmenting tool and
  model events.
- `services/`: File discovery, git service, context/session management, chat
  recording/compression, model config service, shell execution, and loop
  detection utilities.
- `safety/`: Safety checkers and context builders for guarding execution.
- `commands/`: Core implementations for CLI commands like `init`, `memory`,
  `extensions`, and `restore`.
- `output/`: JSON + streaming JSON formatters for machine-readable outputs.
- `resources/`: Resource registry used by tools and integrations.
- `utils/`: Shared helpers (filesystem, workspace context, retries, encoding,
  prompt IDs, checkpoints, etc.).

### Key entry points worth knowing
- `packages/core/src/core/client.ts`: LLM client wrapper used by the CLI.
- `packages/core/src/core/geminiChat.ts`: Chat loop logic + streaming handling.
- `packages/core/src/core/coreToolScheduler.ts`: Tool call orchestration.
- `packages/core/src/tools/tool-registry.ts`: Central tool registration.
- `packages/core/src/services/contextManager.ts`: Session context assembly.

## Build/test entrypoints
- Build: `npm run build` (and `npm run bundle` for packaged output)
- Tests: `npm run test`, `npm run test:integration:*`
- Lint/typecheck: `npm run lint`, `npm run typecheck`

## Notes
- Node.js >= 20 is required (see root `package.json`).
- MCP support is wired through `packages/core` and surfaced in the CLI config.

## Complex request execution flow (end-to-end)
Below is a detailed, code-oriented walkthrough of how a complex request
(multi-step reasoning + multiple tool calls) is handled from the CLI to the
core and back.

### 1) CLI bootstraps and loads configuration
- **Entry point**: `packages/cli/src/gemini.tsx`
- **Config load**: `packages/cli/src/config/config.ts` and
  `packages/core/src/config/config.ts` parse CLI args + settings.
- **Settings + model selection**: core config + model tables in
  `packages/core/src/config/models.ts` and `packages/core/src/config/defaultModelConfigs.ts`.

### 2) Session context and memory are assembled
- **Memory discovery** (GEMINI.md chain): `packages/core/src/utils/memoryDiscovery.ts`.
- **Workspace context**: `packages/core/src/utils/workspaceContext.ts`.
- **Context manager** builds what gets injected into prompts:
  `packages/core/src/services/contextManager.ts`.

### 3) Prompt + request are constructed
- **Prompt template assembly**: `packages/core/src/core/prompts.ts`.
- **Turn state + token accounting**: `packages/core/src/core/turn.ts` and
  `packages/core/src/core/tokenLimits.ts`.
- **Request building**: `packages/core/src/core/geminiRequest.ts`.

### 4) Model routing, availability, and fallback
- **Availability + policy**: `packages/core/src/availability/*`.
- **Model routing strategy**: `packages/core/src/routing/modelRouterService.ts`.
- **Fallback logic**: `packages/core/src/fallback/handler.ts` and
  `packages/core/src/utils/quotaErrorDetection.ts`.

### 5) Chat execution and streaming
- **Primary chat loop**: `packages/core/src/core/geminiChat.ts`
  (streaming chunks + retry handling).
- **Content generators**:
  `packages/core/src/core/contentGenerator.ts`,
  `packages/core/src/core/loggingContentGenerator.ts`,
  `packages/core/src/core/recordingContentGenerator.ts`.

### 6) Tool call extraction and scheduling
- **Tool registry + definitions**: `packages/core/src/tools/tool-registry.ts`,
  `packages/core/src/tools/tools.ts`.
- **Scheduler**: `packages/core/src/core/coreToolScheduler.ts` and
  `packages/core/src/scheduler/scheduler.ts`.
- **Confirmation flow**: `packages/core/src/scheduler/confirmation.ts` and
  `packages/core/src/confirmation-bus/message-bus.ts`.
- **Policy checks**: `packages/core/src/policy/policy-engine.ts` and
  `packages/core/src/policy/config.ts`.

### 7) Tool execution (including MCP tools)
- **Local tools** live in `packages/core/src/tools/`:
  `read-file.ts`, `edit.ts`, `write-file.ts`, `shell.ts`, `web-fetch.ts`, etc.
- **MCP client/tool bridge**:
  `packages/core/src/tools/mcp-client.ts`,
  `packages/core/src/tools/mcp-tool.ts`,
  with auth helpers in `packages/core/src/mcp/`.
- **Shell + file execution services**:
  `packages/core/src/services/shellExecutionService.ts`,
  `packages/core/src/services/fileSystemService.ts`.

### 8) Tool results are fed back into the model
- Tool results are wrapped and appended to the turn in
  `packages/core/src/core/turn.ts`.
- The chat loop in `packages/core/src/core/geminiChat.ts` sends the next model
  request, potentially triggering more tools (iterative reasoning cycle).

### 9) Output formatting + telemetry
- **Output formatters**:
  `packages/core/src/output/json-formatter.ts` and
  `packages/core/src/output/stream-json-formatter.ts`.
- **Telemetry events**: `packages/core/src/telemetry/`.
- **Session recording + summaries**:
  `packages/core/src/services/chatRecordingService.ts` and
  `packages/core/src/services/sessionSummaryService.ts`.

### 10) CLI renders responses and updates UI
- **UI rendering**: `packages/cli/src/ui/*` and `packages/cli/src/gemini.tsx`.
- **Non-interactive mode**: `packages/cli/src/nonInteractiveCli.ts` renders
  structured output (JSON, streaming JSON, etc.).
