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

### From source (development)

```bash
git clone https://github.com/eidolonlabs-ai/hermes-eidolon-memory.git
cd hermes-eidolon-memory
pip install -e .
```

### From PyPI (when published)

```bash
pip install hermes-eidolon-memory
```

### From GitHub

```bash
pip install git+https://github.com/eidolonlabs-ai/hermes-eidolon-memory.git
```

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
| `eidolon_context` | Get a formatted memory context block |
| `eidolon_journal` | Write a journal entry or reflection |
| `eidolon_get_journal` | Retrieve the companion's journal |
| `eidolon_generate_insights` | Synthesize psychological insights |
| `eidolon_generate_musing` | Generate a spontaneous reflection |
| `eidolon_lookup_fact` | Direct fact lookup by subject/predicate |

## Requirements

- Python 3.10+
- Hermes Agent (installed and configured)
- Eidolon Agent Memory MCP server running

## License

MIT
