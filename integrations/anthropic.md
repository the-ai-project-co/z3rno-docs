# Anthropic / Claude Integration

Use Z3rno to give Claude persistent, cross-session memory via MCP or the Anthropic SDK directly.

## What is MCP?

The [Model Context Protocol](https://modelcontextprotocol.io/) (MCP) is an open standard that lets AI assistants call external tools over a lightweight transport (stdio or HTTP). Z3rno ships an MCP server (`z3rno-mcp`) that exposes four memory tools:

| Tool | Purpose |
|------|---------|
| `z3rno.store` | Persist facts, preferences, decisions |
| `z3rno.recall` | Semantic search over stored memories |
| `z3rno.forget` | Soft or GDPR-compliant hard delete |
| `z3rno.audit` | Query the audit log for compliance |

**Why Z3rno + MCP?** Claude has no built-in persistent memory. Z3rno gives it durable, vector-searchable, graph-aware memory that survives across conversations, agents, and sessions.

---

## Installation

```bash
# pip
pip install z3rno-mcp

# or with uv
uv pip install z3rno-mcp
```

The package installs a CLI entry point: `z3rno-mcp`.

---

## Claude Desktop Configuration

Edit (or create) `~/Library/Application Support/Claude/claude_desktop_config.json` on macOS, or `%APPDATA%\Claude\claude_desktop_config.json` on Windows:

```json
{
  "mcpServers": {
    "z3rno": {
      "command": "z3rno-mcp",
      "env": {
        "Z3RNO_API_KEY": "z3rno_sk_...",
        "Z3RNO_AGENT_ID": "claude-desktop",
        "Z3RNO_BASE_URL": "https://api.z3rno.dev"
      }
    }
  }
}
```

Restart Claude Desktop. You should see the Z3rno tools listed in the tool picker.

---

## Cursor Configuration

Create `.cursor/mcp.json` in your project root (or `~/.cursor/mcp.json` for global):

```json
{
  "mcpServers": {
    "z3rno": {
      "command": "z3rno-mcp",
      "env": {
        "Z3RNO_API_KEY": "z3rno_sk_...",
        "Z3RNO_AGENT_ID": "cursor-agent",
        "Z3RNO_BASE_URL": "https://api.z3rno.dev"
      }
    }
  }
}
```

Reload Cursor. The Z3rno tools will appear in Cursor's agent mode.

---

## Claude Code

```bash
claude mcp add z3rno-mcp \
  --env Z3RNO_API_KEY=z3rno_sk_... \
  --env Z3RNO_AGENT_ID=claude-code \
  --env Z3RNO_BASE_URL=https://api.z3rno.dev
```

---

## Demo: Storing and Recalling Memories

Below is an example conversation once the MCP server is connected:

**You:** Remember that I prefer dark mode and use Neovim as my editor.

**Claude** *(calls z3rno.store)*:
```json
{
  "content": "User prefers dark mode and uses Neovim as their editor.",
  "memory_type": "semantic",
  "importance": 0.8
}
```

**Claude:** Got it — I've stored your preference for dark mode and Neovim.

---

*(New conversation, days later)*

**You:** Set up my terminal config.

**Claude** *(calls z3rno.recall)*:
```json
{
  "query": "user editor and theme preferences"
}
```

**Claude:** Based on what I remember, you prefer dark mode and use Neovim. I'll generate a config optimized for that setup...

---

## Alternative: Direct Anthropic SDK Integration (No MCP)

If you are building an application using the Anthropic API directly and want to manage tool calls yourself (without an MCP client), use `z3rno.integrations.anthropic`.

### Setup

```bash
pip install z3rno anthropic
```

### Code Example

```python
import anthropic
from z3rno import Z3rnoClient
from z3rno.integrations.anthropic import get_memory_tools, handle_tool_call

# Initialize clients
z3rno = Z3rnoClient(
    base_url="https://api.z3rno.dev",
    api_key="z3rno_sk_...",
)
claude = anthropic.Anthropic()

# Get Z3rno tools in Anthropic tool_use format
tools = get_memory_tools(agent_id="my-agent")

# Send a message with memory tools available
response = claude.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=tools,
    messages=[
        {"role": "user", "content": "Remember that my deploy target is Kubernetes on GCP."}
    ],
)

# Handle tool calls
for block in response.content:
    if block.type == "tool_use":
        result = handle_tool_call(z3rno, block)
        print(f"Tool {block.name} returned: {result}")
```

### What `get_memory_tools()` returns

A list of Anthropic-formatted tool definitions for `z3rno.store`, `z3rno.recall`, `z3rno.forget`, and `z3rno.audit` — ready to pass directly to `client.messages.create(tools=...)`.

---

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `Z3RNO_API_KEY` | Yes | — | Your Z3rno API key |
| `Z3RNO_BASE_URL` | No | `https://api.z3rno.dev` | Server URL |
| `Z3RNO_AGENT_ID` | No | — | Default agent ID (avoids passing it per call) |

---

## Next Steps

- [Python SDK reference](/sdks/python)
- [TypeScript SDK reference](/sdks/typescript)
- [Self-hosting guide](/self-hosting)
