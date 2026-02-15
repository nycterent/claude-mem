# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What is Claude-Mem?

Claude-mem is a Claude Code plugin providing persistent memory across sessions. It captures tool usage, compresses observations using the Claude Agent SDK, and injects relevant context into future sessions.

## Build & Development Commands

```bash
# Primary development workflow - builds everything, syncs to marketplace, restarts worker
npm run build-and-sync

# Individual build steps
npm run build                 # Build hooks/worker/viewer via esbuild
npm run sync-marketplace      # Copy plugin/ to ~/.claude/plugins/marketplaces/thedotmack/

# Worker management
npm run worker:start          # Start worker daemon
npm run worker:stop           # Stop worker
npm run worker:restart        # Restart worker
npm run worker:status         # Check worker status
npm run worker:logs           # View today's logs
npm run worker:tail           # Tail today's logs

# Testing
npm run test                  # Run all tests with Bun
npm run test:sqlite           # SQLite module tests
npm run test:agents           # Agent tests
npm run test:search           # Search tests
npm run test:context          # Context generation tests
npm run test:infra            # Infrastructure tests
bun test tests/path/to/file.test.ts  # Run single test file

# Utilities
npm run queue                 # Check pending observation queue
npm run queue:process         # Process pending queue
npm run claude-md:regenerate  # Regenerate CLAUDE.md files
```

## Architecture

**Data Flow**: Hook Events → Worker Service HTTP API → SQLite + Chroma → Context Injection

**4 Lifecycle Hooks** (defined in `plugin/hooks/hooks.json`):
- `SessionStart` - Injects context from previous sessions
- `UserPromptSubmit` - Initializes session tracking
- `PostToolUse` - Captures tool observations asynchronously
- `Stop` - Generates session summary

**Worker Service** (`src/services/worker-service.ts`):
- Express server on port 37777, runs as Bun-managed daemon
- Orchestrates specialized modules in `src/services/worker/`:
  - `SessionManager` - Session lifecycle
  - `SDKAgent`/`GeminiAgent`/`OpenRouterAgent` - AI summarization
  - `SearchManager` - Hybrid SQLite FTS5 + Chroma vector search
  - `SSEBroadcaster` - Real-time UI updates
- HTTP routes in `src/services/worker/http/routes/`

**Database Layer** (`src/services/sqlite/`):
- SQLite3 at `~/.claude-mem/claude-mem.db`
- Modules: `Sessions`, `Observations`, `Summaries`, `Prompts`, `Timeline`
- FTS5 for text search, transactions support

**Context System** (`src/services/context/`):
- `ContextBuilder` - Assembles context for injection
- `ObservationCompiler` - Formats observations
- `TokenCalculator` - Manages context budget
- Formatters in `formatters/`, section renderers in `sections/`

**Build System**:
- `scripts/build-hooks.js` - esbuild bundles TypeScript → CJS
- Outputs to `plugin/scripts/`: `worker-service.cjs`, `mcp-server.cjs`, `context-generator.cjs`
- `scripts/build-viewer.js` - Bundles React UI to `plugin/ui/viewer.html`

## File Locations

| Purpose | Location |
|---------|----------|
| Source | `src/` |
| Built Plugin | `plugin/` |
| Installed Plugin | `~/.claude/plugins/marketplaces/thedotmack/` |
| Database | `~/.claude-mem/claude-mem.db` |
| Chroma vectors | `~/.claude-mem/chroma/` |
| Settings | `~/.claude-mem/settings.json` |
| Logs | `~/.claude-mem/logs/` |

## Key Implementation Details

**Privacy Tags**: `<private>content</private>` prevents storage. Tag stripping at hook layer (`src/utils/tag-stripping.ts`).

**Exit Codes** (Claude Code hook contract):
- Exit 0: Success or graceful shutdown
- Exit 1: Non-blocking error (stderr shown)
- Exit 2: Blocking error (stderr fed to Claude)

Worker errors exit 0 to prevent Windows Terminal tab accumulation.

**MCP Server** (`src/servers/mcp-server.ts`): Exposes search tools via Model Context Protocol - `search`, `timeline`, `get_observations`.

## Testing

Tests use Bun's built-in test runner. Test files in `tests/` mirror `src/` structure. Run specific test categories with `npm run test:<category>`.

## Important Notes

- Changelog is auto-generated - never edit manually
- Settings auto-created with defaults on first run
- Bun and uv are auto-installed if missing
