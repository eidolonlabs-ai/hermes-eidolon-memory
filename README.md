# Hermes Eidolon Memory Plugin

Long-term semantic memory plugin for [Hermes Agent](https://github.com/nousresearch/hermes-agent), backed by [Eidolon Agent Memory](https://github.com/eidolonlabs-ai/eidolon-agent-memory).

## Features

- **Fact triples** with pgvector embeddings for semantic search
- **Episodic memory** — journal entries, reflections, dreams
- **Companion profiles** — personalized memory per companion
- **Graceful omission** of high-salience crisis content in casual contexts
- **Background prefetch** — relevant facts recalled before each turn
- **Session fact extraction** — automatic fact extraction from conversations

## Installation

### Prerequisites

- Hermes Agent installed and configured
- Eidolon Agent Memory MCP server running (see [eidolon-agent-memory](https://github.com/eidolonlabs-ai/eidolon-agent-memory))

### Step 1: Install the plugin in Hermes' Python environment

Hermes uses its own virtual environment. Install the plugin there:

```bash
# Find Hermes' Python executable
HERMES_PYTHON=$(head -1 $(which hermes) | sed 's/#!//')

# Install the plugin in Hermes' environment
uv pip install -e . --python "$HERMES_PYTHON"
```

Or manually:

```bash
uv pip install -e . --python ~/.hermes/hermes-agent/venv/bin/python
```

### Step 2: Register the plugin with Hermes

Hermes discovers memory plugins from `~/.hermes/plugins/`. Create the plugin directory:

```bash
mkdir -p ~/.hermes/plugins/eidolon
```

Copy the plugin code:

```bash
cp src/hermes_eidolon_memory/__init__.py ~/.hermes/plugins/eidolon/__init__.py
```

Create `~/.hermes/plugins/eidolon/plugin.yaml`:

```yaml
name: eidolon
description: Eidolon Agent Memory — long-term semantic memory with fact triples, pgvector embeddings, episodic memory, and companion profiles.
version: 1.0.0
author: Eidolon Labs
pip_dependencies:
  - hermes-eidolon-memory
```

### Step 3: Verify installation

```bash
hermes memory setup
```

You should see `eidolon` in the list of available memory providers.

### Alternative: From PyPI (when published)

```bash
hermes_python=$(head -1 $(which hermes) | sed 's/#!//')
uv pip install hermes-eidolon-memory --python "$hermes_python"
```

Then follow Step 2 above to register the plugin.

## Configuration

### 1. Enable the plugin

Add to `~/.hermes/config.yaml`:

```yaml
plugins:
  enabled:
    - eidolon

memory:
  provider: eidolon
```

### 2. Run the setup wizard

```bash
hermes memory setup
```

This will:
- Connect to your Eidolon MCP server
- Provision a user (or use existing API key)
- Create or select a companion
- Save configuration to `$HERMES_HOME/eidolon/config.json`

### 3. Manual configuration (optional)

Create `$HERMES_HOME/eidolon/config.json`:

```json
{
  "api_url": "http://localhost:3100/mcp",
  "companion_id": "<your-companion-uuid>",
  "recall_intent": "factual",
  "auto_recall": true,
  "auto_extract": true
}
```

Set your API key in `$HERMES_HOME/.env`:

```
EIDOLON_API_KEY=mnemo-...
```

Or use environment variables:

```bash
export EIDOLON_API_KEY=mnemo-...
export EIDOLON_API_URL=http://localhost:3100/mcp
export EIDOLON_COMPANION_ID=<uuid>
```

## Tools

The plugin exposes these tools to the agent:

| Tool | Description |
|------|-------------|
| `eidolon_search` | Semantic search for facts about the user |
| `eidolon_store_fact` | Store a fact triple in long-term memory |
| `eidolon_journal` | Write a journal entry, diary, dream, or reflection |
| `eidolon_get_journal` | Retrieve the companion's journal |
| `eidolon_generate_insights` | Synthesize psychological insights |
| `eidolon_generate_musing` | Generate a spontaneous reflection |
| `eidolon_lookup_fact` | Direct fact lookup by subject/predicate |
| `eidolon_delete_fact` | Permanently delete a fact by edge_id |
| `eidolon_update_fact` | Update a fact's importance/confidence |
| `eidolon_get_episodic` | Search episodic memories (diary, dreams, conversations) |
| `eidolon_get_relationship` | Get trust, closeness, and interaction state |
| `eidolon_set_preference` | Store a user preference (key-value) |
| `eidolon_generate_diary` | Generate a diary entry from companion perspective |
| `eidolon_generate_dream` | Generate a dream-like narrative about the user |
| `eidolon_get_companion` | Get companion configuration (name, persona, traits) |

## Requirements

- Python 3.10+
- Hermes Agent (installed and configured)
- Eidolon Agent Memory MCP server running

## License

MIT
