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
