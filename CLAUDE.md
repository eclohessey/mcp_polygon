# CLAUDE.md

This file provides guidance for AI assistants working in this repository.

## Project Overview

`mcp_polygon` is a [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) server that exposes [Polygon.io](https://polygon.io) financial market data APIs as tools available to AI models such as Claude. It provides 67+ tools covering stocks, options, forex, crypto, futures, and economic data.

- **Current version:** 0.5.1
- **Python requirement:** >=3.10 (project uses 3.13)
- **Status:** Experimental — subject to breaking changes
- **License:** MIT

## Repository Structure

```
mcp_polygon/
├── src/mcp_polygon/
│   ├── __init__.py        # Exports `run` from server.py
│   ├── server.py          # All MCP tool definitions (~2000 lines)
│   └── formatters.py      # JSON→CSV conversion utilities
├── tests/
│   ├── __init__.py
│   └── test_formatters.py # Tests for formatters only
├── .github/workflows/
│   └── ci.yml             # CI: lint + test on PRs to master
├── assets/                # Banner images for README
├── entrypoint.py          # Docker entry point; reads MCP_TRANSPORT env var
├── Dockerfile             # python:3.13-slim image, uses UV
├── docker-compose.yml     # Passes POLYGON_API_KEY; mounts repo into /app
├── justfile               # Task runner: `lint` and `test` recipes
├── pyproject.toml         # Project metadata, deps, build config
├── manifest.json          # MCP server manifest (for registries)
├── glama.json             # Glama registry metadata
└── README.md              # Installation, usage, full tool list
```

## Development Commands

All commands use [UV](https://docs.astral.sh/uv/) for environment and dependency management and [just](https://just.systems/) as the task runner.

```bash
# Install all dependencies (including dev)
uv sync

# Run linter (ruff format + ruff check --fix)
uv run just lint

# Run tests
uv run just test

# Run the MCP server locally (stdio transport)
POLYGON_API_KEY=<your_key> uv run mcp_polygon

# Run via Docker
POLYGON_API_KEY=<your_key> docker compose up
```

Individual commands without `just`:

```bash
uv run ruff format          # Format code
uv run ruff check --fix     # Lint and auto-fix
uv run pytest -v tests      # Run tests verbosely
```

## Architecture

### MCP Server (`src/mcp_polygon/server.py`)

The server is built on `FastMCP` from the `mcp[cli]` package. The main instance is:

```python
poly_mcp = FastMCP("Polygon", dependencies=["polygon"])
```

A single `RESTClient` from `polygon-api-client` is instantiated at module load time using `POLYGON_API_KEY` from the environment. The client's `User-Agent` header is augmented with the package version.

**Entry point:** `run(transport: Literal["stdio", "sse", "streamable-http"] = "stdio")` — called from `__init__.py` and `entrypoint.py`.

### Tool Definition Pattern

Every tool follows this exact pattern — no exceptions:

```python
@poly_mcp.tool(annotations=ToolAnnotations(readOnlyHint=True))
async def tool_name(param: type, optional_param: Optional[type] = None) -> str:
    """
    One-line description of what the tool does.
    """
    try:
        results = polygon_client.some_method(..., raw=True)
        return json_to_csv(results.data.decode("utf-8"))
    except Exception as e:
        return f"Error: {e}"
```

Key conventions:
- All tools are `async` functions returning `str`
- `readOnlyHint=True` is set on every tool (all calls are read-only API fetches)
- Always pass `raw=True` to the Polygon client to get the raw HTTP response
- Decode the binary response with `.data.decode("utf-8")` before passing to `json_to_csv`
- Wrap the entire body in `try/except Exception` and return `f"Error: {e}"` on failure
- Default limits (e.g., `limit: Optional[int] = 10`) are set where appropriate

### Formatters (`src/mcp_polygon/formatters.py`)

**Purpose:** Convert Polygon.io JSON API responses to CSV for LLM token efficiency.

- `json_to_csv(json_input: str | dict) -> str`: Top-level converter. If the input has a `results` key, extracts it; otherwise wraps the data in a list. Returns a flat CSV string.
- `_flatten_dict(d, parent_key, sep) -> dict`: Internal recursive helper. Joins nested keys with `_`, converts lists to their string representation.

### Transport (`entrypoint.py`)

The `MCP_TRANSPORT` environment variable selects the transport. Supported values:

| Value | Description |
|---|---|
| `stdio` (default) | Standard I/O — used by Claude Desktop and Claude Code |
| `sse` | Server-Sent Events |
| `streamable-http` | Streamable HTTP |

## Tool Categories

| Category | Tools |
|---|---|
| Aggregates & Bars | `get_aggs`, `list_aggs`, `get_grouped_daily_aggs`, `get_daily_open_close_agg`, `get_previous_close_agg` |
| Trades | `list_trades`, `get_last_trade`, `get_last_crypto_trade` |
| Quotes | `list_quotes`, `get_last_quote`, `get_last_forex_quote` |
| Forex | `get_real_time_currency_conversion` |
| Snapshots | `list_universal_snapshots`, `get_snapshot_all`, `get_snapshot_direction`, `get_snapshot_ticker`, `get_snapshot_option`, `get_snapshot_crypto_book` |
| Market Status | `get_market_holidays`, `get_market_status` |
| Ticker Reference | `list_tickers`, `get_ticker_details`, `list_ticker_news`, `get_ticker_types` |
| Corporate Actions | `list_splits`, `list_dividends` |
| Reference Data | `list_conditions`, `get_exchanges` |
| Financials | `list_stock_financials` |
| IPOs & Short Interest | `list_ipos`, `list_short_interest`, `list_short_volume` |
| Economic Data | `list_treasury_yields`, `list_inflation` |
| Benzinga Analyst Data | `list_benzinga_analyst_insights`, `list_benzinga_analysts`, `list_benzinga_consensus_ratings`, `list_benzinga_earnings`, `list_benzinga_firms`, `list_benzinga_guidance`, `list_benzinga_news`, `list_benzinga_ratings` |
| Futures | `list_futures_aggregates`, `list_futures_contracts`, `get_futures_contract_details`, `list_futures_products`, `get_futures_product_details`, `list_futures_quotes`, `list_futures_trades`, `list_futures_schedules`, `list_futures_schedules_by_product_code`, `list_futures_market_statuses`, `get_futures_snapshot` |

## Testing

Tests live in `tests/test_formatters.py` and cover only the `formatters` module. There are no integration or server-level tests.

Test classes:
- `TestFlattenDict` — 8 tests for `_flatten_dict` covering flat, nested, deeply nested, lists, mixed, empty, and `None` values
- `TestJsonToCsvStdlib` — 24 tests for `json_to_csv` covering real API shapes, input variants (dict/string/list), edge cases, data types, and Unicode

When adding new functionality to `formatters.py`, add corresponding tests in `test_formatters.py`.

## CI/CD

GitHub Actions runs on every pull request targeting `master`:

1. Install UV and Python (version from `pyproject.toml`)
2. `uv sync` — install all dependencies
3. `uv run just lint` — ruff format + ruff check
4. `uv run just test` — pytest

Permissions are read-only; the workflow cannot push or modify the repository.

## Adding a New Tool

1. Open `src/mcp_polygon/server.py`.
2. Add a new `async` function decorated with `@poly_mcp.tool(annotations=ToolAnnotations(readOnlyHint=True))`.
3. Follow the established pattern: accept typed parameters, call the appropriate `polygon_client` method with `raw=True`, decode, call `json_to_csv`, and wrap in `try/except`.
4. Use `Optional[type] = None` for all optional parameters. Add a sensible default `limit` if the endpoint is paginated.
5. Run `uv run just lint` and `uv run just test` before committing.

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `POLYGON_API_KEY` | Yes | Polygon.io API key — passed to `RESTClient` |
| `MCP_TRANSPORT` | No | Transport type: `stdio` (default), `sse`, `streamable-http` |

## Dependencies

**Runtime:**
- `mcp[cli]>=1.15.0` — FastMCP server framework
- `polygon-api-client>=1.15.4` — Official Polygon.io Python client

**Development:**
- `ruff>=0.13.2` — Linter and formatter
- `pytest>=8.4.0` — Test runner
- `rust-just>=1.42.4` — Task runner (`just`)

Dependencies are managed by UV. The lockfile (`uv.lock`) must be kept in sync. Run `uv sync` after any change to `pyproject.toml`.

## Code Style

Formatting and linting are enforced by `ruff`. Run `uv run just lint` to auto-format and fix issues before committing. The CI will fail if the code is not properly formatted.

There are no custom ruff configuration sections in `pyproject.toml`; default ruff rules apply.

## Key Constraints

- **Do not** add tools that perform write operations to Polygon.io. All tools must be read-only.
- **Do not** store or log API keys anywhere in the codebase.
- **Do not** change the return type of tools away from `str`. The MCP protocol and CSV formatting depend on string returns.
- **Do not** remove the `try/except Exception` wrapper from tools — errors must be returned as strings, not raised.
- The `polygon_client` is a module-level singleton. Do not instantiate additional clients inside individual tool functions.
