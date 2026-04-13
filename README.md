# Pinboard MCP Server

[![CI](https://github.com/rossshannon/pinboard-bookmarks-mcp-server/actions/workflows/ci.yml/badge.svg)](https://github.com/rossshannon/pinboard-bookmarks-mcp-server/actions/workflows/ci.yml)
[![Python 3.13](https://img.shields.io/badge/python-3.13-blue.svg)](https://www.python.org/downloads/)

Read and write access to Pinboard.in bookmarks for LLMs via Model Context Protocol (MCP).

Forked from [rossshannon/pinboard-bookmarks-mcp-server](https://github.com/rossshannon/pinboard-bookmarks-mcp-server) with write tools added.

## Overview

This server provides LLMs with the ability to search, filter, retrieve, add, and delete bookmarks in Pinboard.in at inference time. Built on FastMCP 2.0, it offers seven tools for bookmark interaction while respecting Pinboard's rate limits and implementing intelligent caching.

## Features

- **Full read/write access** to Pinboard bookmarks
- **Seven MCP tools**: `search_bookmarks`, `search_bookmarks_extended`, `list_recent_bookmarks`, `list_bookmarks_by_tags`, `list_tags`, `add_bookmark`, `delete_bookmark`
- **Smart caching** with LRU cache and automatic invalidation using `posts/update` endpoint
- **Write-through cache invalidation** — cache clears immediately on add/delete
- **Rate limiting** respects Pinboard's 3-second guideline between API calls
- **Field mapping** converts Pinboard's legacy field names to intuitive ones (description→title, extended→notes)
- **Flexible token config** — accepts `PINBOARD_TOKEN` or `PINBOARD_API_TOKEN`
- **Requires Python 3.13** (pinned for pydantic-core compatibility)

## Installation

### From this fork (recommended)
```bash
pip install git+https://github.com/brysmi/pinboard-bookmarks-mcp-server.git
```

### From source
```bash
git clone https://github.com/brysmi/pinboard-bookmarks-mcp-server.git
cd pinboard-bookmarks-mcp-server
pip install -e .
```

## Quick Start

1. **Get your Pinboard API token** from https://pinboard.in/settings/password
2. **Set environment variable**:
   ```bash
   export PINBOARD_TOKEN="username:1234567890ABCDEF"
   # or equivalently:
   export PINBOARD_API_TOKEN="username:1234567890ABCDEF"
   ```
3. **Start the server**:
   ```bash
   pinboard-mcp-server
   ```

## Usage with Claude Desktop

Add this to `~/.config/Claude/claude_desktop_config.json` (Linux) or the equivalent on your platform:

```json
{
  "mcpServers": {
    "pinboard": {
      "command": "uv",
      "args": [
        "run",
        "--python", "3.13",
        "--project", "/path/to/pinboard-bookmarks-mcp-server",
        "pinboard-mcp-server"
      ],
      "env": {
        "PINBOARD_TOKEN": "your-username:your-token-here"
      }
    }
  }
}
```

Or if installed via pip:

```json
{
  "mcpServers": {
    "pinboard": {
      "command": "pinboard-mcp-server",
      "env": {
        "PINBOARD_TOKEN": "your-username:your-token-here"
      }
    }
  }
}
```

## Available Tools

### Read tools

#### `search_bookmarks`
Search bookmarks by keyword across titles, notes, and tags (searches recent bookmarks first).

**Parameters:**
- `query` (string): Search query
- `limit` (int, optional): Maximum results (default: 20, max: 100)

#### `search_bookmarks_extended`
Comprehensive historical search across titles, notes, and tags.

**Parameters:**
- `query` (string): Search query
- `days_back` (int, optional): How far back to search (default: 365, max: 730)
- `limit` (int, optional): Maximum results (default: 100, max: 200)

#### `list_recent_bookmarks`
List bookmarks saved in the last N days.

**Parameters:**
- `days` (int, optional): Days to look back (default: 7, max: 30)
- `limit` (int, optional): Maximum results (default: 20, max: 100)

#### `list_bookmarks_by_tags`
List all bookmarks filtered by tags with optional date range.

**Parameters:**
- `tags` (array): Tags to filter by (1–3 tags)
- `from_date` (string, optional): Start date in ISO format (YYYY-MM-DD)
- `to_date` (string, optional): End date in ISO format (YYYY-MM-DD)
- `limit` (int, optional): Maximum results (default: 100, max: 200)

#### `list_tags`
List all tags with their usage counts.

### Write tools

#### `add_bookmark`
Add a new bookmark or update an existing one.

**Parameters:**
- `url` (string): URL to bookmark
- `title` (string): Title of the bookmark
- `description` (string, optional): Extended notes
- `tags` (array, optional): List of tags
- `shared` (bool, optional): Public bookmark (default: true)
- `toread` (bool, optional): Mark as unread/to-read (default: false)
- `replace` (bool, optional): Replace if URL already exists (default: true)

**Example:**
```
Add a bookmark for https://example.com with title "Example" and tags ["python", "reference"]
```

#### `delete_bookmark`
Delete a bookmark by URL.

**Parameters:**
- `url` (string): URL of the bookmark to delete

**Example:**
```
Delete the bookmark for https://example.com
```

## Configuration

### Environment Variables

- `PINBOARD_TOKEN` or `PINBOARD_API_TOKEN` (required): Your Pinboard API token in format `username:token`

### Rate Limiting

The server automatically enforces a 3-second delay between Pinboard API calls to respect their guidelines. Cached responses are returned immediately.

### Caching Strategy

- **Query cache**: LRU cache with 1000 entries for search results
- **Bookmark cache**: Full bookmark list cached for 1 hour
- **Cache invalidation**: Uses `posts/update` endpoint to detect remote changes
- **Write-through invalidation**: Cache clears immediately after `add_bookmark` or `delete_bookmark`
- **Tag cache**: Tag list cached until manually refreshed

## Testing

### Run all tests
```bash
cd pinboard-bookmarks-mcp-server
uv run pytest --cov=src --cov-report=term-missing
```

### Real API testing
```bash
PINBOARD_TOKEN="username:token" uv run python tests/debug_bookmarks.py
```

## Development

### Setup
```bash
git clone https://github.com/brysmi/pinboard-bookmarks-mcp-server.git
cd pinboard-bookmarks-mcp-server
uv sync --extra dev
```

### Code Quality
```bash
uv run ruff check src/ tests/
uv run ruff format src/ tests/
uv run mypy src/
uv run pytest -v
```

### Architecture

- **FastMCP 2.0**: MCP scaffolding with Tool abstraction and async server
- **pinboard.py**: Pinboard API client wrapper with error handling
- **Pydantic**: Data validation and serialization
- **ThreadPoolExecutor**: Bridges async MCP with sync pinboard.py library
- **LRU Cache**: In-memory caching with intelligent invalidation

### Key Files

- `src/pinboard_mcp_server/main.py` — MCP server entry point and tool implementations
- `src/pinboard_mcp_server/client.py` — Pinboard API client with caching and rate limiting
- `src/pinboard_mcp_server/models.py` — Pydantic data models
- `tests/` — Test suite
- `tests/debug_bookmarks.py` — Debug utility for real API testing

## Performance

- **P50 response time**: <250ms (cached)
- **P95 response time**: <600ms (cold cache)
- **Rate limiting**: 3-second intervals between API calls
- **Cache hit ratio**: >90% for typical usage

## Security

- API tokens are never logged or exposed in error messages
- Input validation on all tool parameters
- Secure environment variable handling

## License

MIT License - see [LICENSE](LICENSE) file for details.
