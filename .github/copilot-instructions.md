# B&R Help MCP Server - Copilot Instructions

## Project Overview

This is a **Model Context Protocol (MCP) server** that provides full-text search and retrieval for B&R Automation Studio help documentation. Built with Python 3.12+, FastMCP SDK, and SQLite FTS5 for production-grade search performance.

**Key Architecture Decision:** SQLite FTS5 was chosen over Whoosh/other libraries because it's built into Python, actively maintained, blazingly fast, and requires zero external dependencies for 100K+ local files.

## Core Architecture

### Three-Layer Design

1. **`indexer.py`** - XML Parser & HTML Extractor
   - Parses `brhelpcontent.xml` (abbreviated tags: `S`=Section, `P`=Page, `t`=Text, `p`=File, `I`=Identifiers)
   - Builds in-memory page tree with parent-child relationships
   - Extracts breadcrumbs with **cycle detection** and **depth limit (100)**
   - Uses **lxml** for fast HTML text extraction (2-3x faster than BeautifulSoup)
   - Uses MD5 hash for change detection (stored in `.ashelp_metadata/index_metadata.json`)

2. **`search_engine.py`** - SQLite FTS5 Search
   - Creates FTS5 virtual table with `porter unicode61` tokenizer
   - **BM25 ranking** with title weighted **10x** higher than content
   - **Prefix matching** for partial words (`"term"*` syntax)
   - **Query sanitization** - escapes FTS5 special characters
   - Parallel text extraction using **8 threads** (ThreadPoolExecutor)
   - Stores index in single `.ashelp_search.db` file (50-100MB)
   - Batch inserts (1000 docs) in single transaction (~10-11 min initial build)

3. **`server.py`** - FastMCP Server
   - Exposes 5 tools: `search_help`, `get_page_by_id`, `get_page_by_help_id`, `get_breadcrumb`, `get_help_statistics`
   - **Intentionally truncated previews** (~100 chars) to force LLM to call `get_page_by_id`
   - Server instructions guide LLM to make **multiple searches and page retrievals**
   - Uses Pydantic models for structured responses
   - Resource endpoint: `help://page/{page_id}` for direct HTML access

### Data Flow

```
brhelpcontent.xml → Indexer → SQLite FTS5 → BM25 Ranked Results → MCP Tools
                        ↓
                  HTML Files → lxml → Plain Text → Index Content
```

### Search Ranking (BM25)

Results are ranked using BM25 (Best Match 25):
- **Title matches**: Weighted **10x** higher than content
- **Term frequency**: More occurrences = higher rank
- **Document length**: Short documents with matches rank higher
- **Prefix matching**: `"motor"*` matches "motor", "motors", "motoring"

## Development Workflows

### First-Time Setup

**Docker (Production/Distribution):**
```bash
docker build -t as-help:local .
# See README.md for complete Docker guide
```

**Local Development:**
```bash
# Auto-setup with script (recommended)
./setup.sh

# Manual setup
uv sync  # Creates .venv and installs deps
# Create .env with: AS_HELP_ROOT=/mnt/c/BRAutomation/AS412/Help-en/Data
```

### Testing in VS Code (Recommended)

Use **Run and Debug (Ctrl+Shift+D)** - NOT MCP Inspector (stdio issues on Windows):

1. **"Rebuild BR Help Index"** - First run only 
2. **"Run BR Help MCP Server"** - Fast startup with existing index (<1s)
3. **"Test BR Help Indexer"** - Quick XML parse test (~1s)

**Why not terminal?** Server uses stdio transport for MCP clients. Debugger provides interactive testing without needing a full MCP client.

### Quick Validation

```bash
# Test XML parsing only (1 second)
uv run python test_parsing.py  # Outputs: "Total pages: 107332"

# Test search functionality
uv run python test_search.py   # Runs sample queries
```

## Critical Conventions

### Environment Variables (Required)

- `AS_HELP_ROOT` - Path to `Data` folder containing `brhelpcontent.xml`
  - WSL: `/mnt/c/BRAutomation/AS412/Help-en/Data`
  - Windows: `C:\BRAutomation\AS412\Help-en\Data`
- `AS_HELP_FORCE_REBUILD` - Set `true` for first run, then `false` (auto-rebuild on XML changes)
- `AS_HELP_DB_PATH` - Optional custom DB location (defaults to `{AS_HELP_ROOT}/.ashelp_search.db`)

### Abbreviated XML Tags (Critical!)

The B&R XML uses shortened tags - **both formats must be handled**:
- `Section` or `S` (with `Text`/`t`, `File`/`p`)
- `Page` or `P` (with `Text`/`t`, `File`/`p`)
- `Identifiers` or `I` → `HelpID` or `H` (with `Value`/`v`)

See `_process_section()` and `_process_page()` in `indexer.py` for implementation.

### Index Rebuild Logic

**Do NOT rebuild unnecessarily** - costs 1-2 minutes:
1. Check metadata file exists (`.ashelp_metadata/index_metadata.json`)
2. Compare MD5 hash of current XML vs. stored hash
3. Rebuild only if: `force_rebuild=True` OR hash mismatch OR no metadata

See `needs_reindex()` in `indexer.py` and `_needs_reindex()` in `search_engine.py`.

### Content Extraction Strategy

- **Sections**: Index title only (no HTML content)
- **Pages**: Index title + full plain text from HTML
- **Lazy loading**: HTML/text extracted on-demand and cached in `HelpPage` objects
- **lxml**: Uses `text_content()` for fast extraction, strips `<script>` and `<style>` via XPath

## Integration Points

### MCP Client Configuration

Add to client config (e.g., `claude_desktop_config.json` or VS Code settings):

```json
{
  "mcpServers": {
    "as-help": {
      "command": "uv",
      "args": ["run", "as-help-server"],
      "cwd": "/home/username/projects/as-help",
      "env": {
        "AS_HELP_ROOT": "/mnt/c/BRAutomation/AS412/Help-en/Data",
        "AS_HELP_FORCE_REBUILD": "false"
      }
    }
  }
}
```

### Entry Points

- `uv run as-help-server` - Script entry (defined in `pyproject.toml`)
- `python src/server.py` - Direct execution (set PYTHONPATH or run from src/)

## Performance Expectations

| Operation | Time | Notes |
|-----------|------|-------|
| XML parse | ~2s | pages in-memory |
| First index build | 10-11 min | Parallel HTML extraction + FTS5 indexing |
| Subsequent startup | <3s | Load existing DB |
| Search query | 10-50ms | FTS5 with BM25 ranking |
| Memory usage | 10-30MB | Runtime after index load |

**Log progress every 5000 docs** during indexing to show it's working.

## Common Pitfalls

1. **Don't test with MCP Inspector on Windows** - stdio transport has compatibility issues. Use VS Code debugger.
2. **Always handle both XML tag formats** - Some XMLs use `Section`, others use `S`.
3. **Check metadata hash before rebuilding** - Rebuilding pages is unnecessary waste of time.
4. **Path handling** - B&R uses Windows paths (backslashes), ensure cross-platform Path usage.
5. **Database locking** - Close SQLite connections properly (see `__del__` in `search_engine.py`).

## File Structure Patterns

```
src/
  __init__.py           # Package marker
  __main__.py           # Module entry point
  server.py             # FastMCP server + tools (main logic)
  indexer.py            # XML parsing + HTML extraction
  search_engine.py      # SQLite FTS5 search
  
Root level:
  test_*.py             # Standalone test scripts (no MCP server)
  setup.ps1             # Auto-setup for team distribution
  mcp.json              # Example MCP client config
  .env.example          # Template for environment vars
```

## Testing Strategy

- **Unit tests**: Use `test_parsing.py` (indexer only) and `test_search.py` (search only)
- **Integration**: Use VS Code debugger with breakpoints in `server.py` tools
- **Production**: Configure in MCP client and test via Copilot chat
- **Never**: Run MCP Inspector on Windows (stdio issues)

## Distribution to Team

Use `setup.sh` (WSL/Linux) or `setup.ps1` (Windows) for local installation, or use Docker for containerized deployment:

```bash
docker pull ghcr.io/YOUR_USERNAME/as-help:latest
```

See README.md for complete setup instructions.

## Key Dependencies

- `mcp[cli]` - FastMCP SDK and MCP protocol
- `lxml` - Fast HTML parsing (2-3x faster than BeautifulSoup)
- `python-dotenv` - .env file loading (optional, env vars work directly)
- **Zero search dependencies** - Uses Python's built-in `sqlite3` module

## When Modifying Code

- **Adding tools**: Add `@mcp.tool()` decorated function in `server.py` with Pydantic models
- **Changing XML parsing**: Update both tag formats in `indexer.py` (`Section`/`S`, etc.)
- **Search improvements**: Modify FTS5 query syntax in `search_engine.py` (see SQLite FTS5 docs)
- **New metadata**: Update `_save_metadata()` and `_load_metadata()` in `indexer.py`
