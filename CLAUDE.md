# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Setup (uv recommended)
uv venv && source .venv/bin/activate && uv pip install -e .

# Run
uv run main.py
# or
python main.py
```

Requires `ANTHROPIC_API_KEY` in `.env`. Model is configured via `CLAUDE_MODEL` in `.env`.

## Architecture

This is an interactive CLI chat application built on the **Model Context Protocol (MCP)** with the Anthropic Claude API.

### Layered Structure

```
main.py → core/cli.py (UI) → core/cli_chat.py → core/chat.py → core/claude.py (API)
                                    ↓
                            core/tools.py → mcp_client.py → mcp_server.py
```

- **`main.py`**: Entry point. Loads env, creates `Claude` service, connects MCP clients, launches CLI.
- **`mcp_server.py`**: FastMCP server hosting document tools (`read_doc_contents`, `edit_doc`) and resources (`docs://documents`). Documents are stored in-memory.
- **`mcp_client.py`**: Stdio-based MCP client. Handles `connect()`, `list_tools()`, `call_tool()`, `read_resource()`.
- **`core/claude.py`**: Thin wrapper around the Anthropic SDK for chat calls.
- **`core/chat.py`**: Base `Chat` class implementing the agentic loop (Claude → tool calls → Claude → ... → final response).
- **`core/cli_chat.py`**: Extends `Chat` with document context injection (`@doc` mentions) and `/command` processing.
- **`core/cli.py`**: `prompt_toolkit`-based interactive UI with `UnifiedCompleter` for `@doc` and `/command` autocompletion.
- **`core/tools.py`**: `ToolManager` routes tool calls from Claude to the correct MCP client.

### Key Data Flow

1. User types input (optionally with `@document` references or `/commands`)
2. `CliChat._process_query()` extracts references, fetches document content via MCP resources, and injects them as context
3. `Claude.chat()` sends the message with available tools
4. If Claude requests tool use, `ToolManager.execute_tool_requests()` routes calls to the appropriate `MCPClient`
5. Results are fed back to Claude; loop continues until a final text response

### Known TODOs

- MCP prompts (`list_prompts`, `get_prompt`) are not implemented in `mcp_client.py` or `mcp_server.py`
- `/command` processing in `cli_chat.py` is incomplete
- Document storage is in-memory only (defined in `mcp_server.py`)
