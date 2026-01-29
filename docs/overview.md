# Gemini CLI Codebase Overview

## What this repo is
Gemini CLI is an open-source terminal AI agent for Google's Gemini models. It provides an interactive TUI (and a non-interactive mode) with built-in tools (file ops, shell, search, web fetch), MCP integrations, and auth options (OAuth, API key, Vertex AI).

## High-level structure
- `packages/cli/`: User-facing CLI and TUI (Ink/React). Handles argument parsing, config/settings, auth, UI state, extensions, and tool wiring. Main entrypoints:
  - `packages/cli/src/gemini.tsx`: interactive TUI runtime.
  - `packages/cli/src/nonInteractiveCli.ts`: `gemini -p` style non-interactive runs.
- `packages/core/`: Shared core library used by the CLI. Provides Gemini client abstraction, prompt/tool orchestration, MCP client/server utilities, telemetry, file discovery, shell execution, and workspace context helpers.
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

## Build/test entrypoints
- Build: `npm run build` (and `npm run bundle` for packaged output)
- Tests: `npm run test`, `npm run test:integration:*`
- Lint/typecheck: `npm run lint`, `npm run typecheck`

## Notes
- Node.js >= 20 is required (see root `package.json`).
- MCP support is wired through `packages/core` and surfaced in the CLI config.
