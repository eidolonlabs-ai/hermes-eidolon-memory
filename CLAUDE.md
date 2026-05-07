# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**hermes-eidolon-memory** is a Hermes Agent memory plugin providing long-term semantic memory backed by Eidolon Agent Memory. It integrates:

- **Fact triples** with pgvector embeddings for semantic search
- **Episodic memory** (journal entries, reflections, dreams)
- **Companion profiles** with personalized memory contexts
- **Graceful omission** of high-salience crisis content in casual search modes
- **Background prefetch** of relevant facts before each agent turn
- **Automatic fact extraction** from conversations at session end

The plugin implements Hermes Agent's `MemoryProvider` interface and communicates with the Eidolon MCP server over JSON-RPC 2.0 via HTTP.

## Architecture

**Single-file design**: All logic is in `src/hermes_eidolon_memory/__init__.py` (~1200 lines).

### Key Components

1. **JSON-RPC helpers** (`_mcp_initialize`, `_rpc_call`, `_parse_sse`)
   - HTTP client for MCP server communication
   - Handles FastMCP's SSE-wrapped and raw responses
   - Session ID management for streamable-http

2. **Config management** (`_load_config`, `_save_config`)
   - Loads from `$HERMES_HOME/eidolon/config.json` (non-secret settings)
   - Falls back to environment variables (EIDOLON_API_URL, EIDOLON_API_KEY, EIDOLON_COMPANION_ID)
   - Reads `$HERMES_HOME/.env` explicitly for secrets not sourced by shell

3. **EidolonMemoryProvider class**
   - Implements Hermes `MemoryProvider` interface
   - Provides 14 tool schemas (search, store_fact, journal, lookup, etc.)
   - Handles tool dispatch via `handle_tool_call()`
   - Thread-safe prefetch system: queues background recall, blocks on retrieval
   - Session-end fact extraction: buffers turns, extracts at session end

4. **Setup wizard** (`post_setup`, `_wizard_create_companion`)
   - Interactive configuration on `hermes memory setup`
   - Provisions users and companions via Eidolon API
   - Saves config to correct locations (secrets vs. non-secrets)

### Data Flow

```
User Query
  ↓
queue_prefetch() — starts background thread to fetch relevant facts
  ↓
Hermes calls handle_tool_call() for memory tools
  ↓
_call() injects API key and makes JSON-RPC call to MCP server
  ↓
_parse_sse() extracts JSON from FastMCP response
  ↓
Handler formats result and returns JSON string
  ↓
sync_turn() buffers user/assistant content
  ↓
on_session_end() extracts facts from full transcript
```

## Development

### Environment Setup

```bash
# Create virtual environment
python -m venv .venv
source .venv/bin/activate

# Install in editable mode (setuptools)
pip install -e .
```

Uses `uv` for lock file management; `uv.lock` is in .gitignore (not needed for library packages).

### Dependencies

- **Runtime**: Python 3.10+, no external dependencies (only stdlib)
- **Build**: setuptools>=61.0, wheel
- **Hermes**: Must be installed and running; this plugin integrates via the `hermes_agent.plugins` entry point

### Building & Distribution

```bash
# Build wheel and sdist
python -m build

# Install built wheel
pip install dist/hermes_eidolon_memory-1.0.0-py3-none-any.whl
```

Entry point registered in `pyproject.toml`:
```toml
[project.entry-points."hermes_agent.plugins"]
eidolon = "hermes_eidolon_memory"
```

When Hermes starts, it auto-discovers and imports `hermes_eidolon_memory`, calling `register()` to register the provider.

## Key Patterns & Conventions

### JSON-RPC over MCP

- All server calls go through `_rpc_call()` with session ID
- Session ID obtained lazily via `_get_mcp_session()`, cached and reset on 400/session errors
- Arguments always include `"api_key"` (injected by `_call()`)
- Response parsing: FastMCP wraps results in `{"result": {"content": [{"text": "..."}]}}` — `_parse_sse()` extracts the JSON string

### Threading

- **Prefetch**: `queue_prefetch()` spawns daemon thread, `prefetch()` joins with 3s timeout
- **Fact extraction**: `on_session_end()` spawns daemon thread, joins with 65s timeout
- All MCP operations use timeouts (default 30s, 60s for generation tools)
- Thread-safe state via locks (`_prefetch_lock`, `_mcp_session_lock`)

### Tool Naming & Schema

- All tools have schema dicts: `{name, description, parameters, ...}`
- Handler methods named `_handle_<tool_name>()` — lowercase with underscores
- Each handler validates inputs, calls `_call()`, and returns JSON string
- Errors returned via `tool_error()` (Hermes convention) or exception message

### Configuration Priority

1. Environment variables (if shell sourced `.env`)
2. Explicitly read `.env` file (fallback)
3. `eidolon/config.json` (overrides)
4. Built-in defaults (api_url)

Non-secret keys live in `config.json`; secrets live in `.env` only.

## Common Tasks

### Adding a New Tool

1. Add schema constant (e.g., `NEW_TOOL_SCHEMA`)
2. Include it in `get_tool_schemas()` return list
3. Add handler method `_handle_new_tool(args: dict) -> str`
4. Add dispatch case in `handle_tool_call()`
5. Handler validates args, calls `_call()`, returns JSON string

### Debugging MCP Communication

- Set `logger.setLevel(logging.DEBUG)` to see low-level RPC calls
- Check Eidolon server logs (docker-compose) for actual failures
- Verify MCP session ID in response header — if missing, `_mcp_initialize()` failed
- Use `_parse_sse()` to inspect raw response format

### Testing Against Eidolon Server

Requires Eidolon MCP server running locally (see README for docker-compose setup):

```bash
# Verify connectivity
python -c "from hermes_eidolon_memory import _mcp_initialize; print(_mcp_initialize('http://localhost:3100/mcp'))"

# Test tool call
from hermes_eidolon_memory import EidolonMemoryProvider
p = EidolonMemoryProvider()
p.initialize("session-id")
# Make sure config is set: ~/.hermes/eidolon/config.json
result = p._call("search_memory", {...})
```

## Important Notes

- **Single file**: All logic in one module. Keep related helpers near their consumers.
- **No test suite**: Plugin is integration-tested via Hermes. Manual testing with live Eidolon server is required.
- **Thread safety**: MCP session and prefetch state are guarded by locks. Fact extraction is async to avoid blocking.
- **Error handling**: All RPC errors are caught, logged, and returned to the agent via tool_error(). Network timeouts are explicit.
- **Configuration**: API keys never go in config.json. The .env file is explicitly read (not just via os.environ) to catch keys that weren't shell-sourced.
