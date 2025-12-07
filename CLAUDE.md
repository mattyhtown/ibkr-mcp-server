# CLAUDE.md - IBKR MCP Server

This document provides essential information for AI assistants working with this codebase.

## Project Overview

**IBKR MCP Server** is a Model Context Protocol (MCP) server for Interactive Brokers TWS API integration. It enables AI assistants (like Claude) to interact with Interactive Brokers for algorithmic trading, with a focus on 0DTE (zero days to expiration) SPX options trading.

- **Version**: 3.0.0
- **Language**: Python 3.9+
- **License**: MIT

## Repository Structure

```
ibkr-mcp-server/
├── src/
│   └── ibkr_mcp_server.py     # Main MCP server implementation (~2600 lines)
├── tests/
│   └── test_ibkr_server.py    # Pytest test suite with mocked IBKR bridge
├── docs/
│   └── 0DTE_Trading_Guide.md  # Comprehensive 0DTE options trading guide
├── config/
│   └── claude_desktop_config.json  # Example Claude Desktop configuration
├── requirements.txt           # Python dependencies
├── setup.py                   # Package setup with entry point
├── .env.template              # Environment variables template
├── .gitignore
├── LICENSE
└── README.md
```

## Key Technologies

- **MCP SDK**: `mcp>=1.0.0` - Model Context Protocol for AI integration
- **IBKR API**: `ibapi>=10.19.04` - Interactive Brokers TWS API client
- **Pydantic**: `pydantic>=2.0.0` - Data validation
- **Async**: asyncio, aiofiles - Asynchronous operations
- **HTTP**: httpx - HTTP client

## Architecture

### Core Components

1. **IBKRBridge** (`src/ibkr_mcp_server.py:89-1417`)
   - Extends `EWrapper` and `EClient` from ibapi
   - Handles all TWS/IB Gateway communication
   - Manages async events, market data, orders, positions
   - Key data stores:
     - `market_data`: Real-time quotes and Greeks
     - `orders`: Order status tracking
     - `positions`: Portfolio positions
     - `account_values`: Account summary data
     - `option_chains`: Option chain data
     - `historical_data`: OHLCV bars

2. **MCP Server** (`src/ibkr_mcp_server.py:1420-2600+`)
   - Server instance: `server = Server("ibkr-mcp")`
   - Tool definitions via `@server.list_tools()`
   - Tool handlers via `@server.call_tool()`

### Connection Ports
- **Paper Trading**: Port 7497 (default)
- **Live Trading**: Port 7496

## MCP Tools Reference

### Connection Management
| Tool | Description |
|------|-------------|
| `ibkr_connect` | Connect to TWS/IB Gateway (mode: paper/live) |
| `ibkr_disconnect` | Disconnect from TWS |
| `ibkr_connection_status` | Check connection status |

### Market Data
| Tool | Description |
|------|-------------|
| `ibkr_get_quote` | Real-time quote with optional Greeks |
| `ibkr_get_option_chain` | Option chain with strikes, expiries |
| `ibkr_get_historical_data` | Historical OHLCV candlestick data |
| `ibkr_subscribe_realtime_bars` | Subscribe to 5-second real-time bars |
| `ibkr_cancel_realtime_bars` | Cancel real-time subscription |

### Order Management
| Tool | Description |
|------|-------------|
| `ibkr_place_order` | Place market/limit/stop orders |
| `ibkr_place_bracket_order` | Entry + take profit + stop loss |
| `ibkr_place_vertical_spread` | Vertical spread combo orders |
| `ibkr_get_order_status` | Check specific order status |
| `ibkr_cancel_order` | Cancel pending order |
| `ibkr_get_all_open_orders` | Get all open orders |
| `ibkr_cancel_all_orders` | Emergency: cancel ALL orders |

### Account & Positions
| Tool | Description |
|------|-------------|
| `ibkr_get_positions` | Get all current positions |
| `ibkr_get_account_summary` | Account balances and buying power |
| `ibkr_subscribe_pnl` | Real-time P&L for account |
| `ibkr_subscribe_pnl_single` | Real-time P&L for position |
| `ibkr_get_executions` | Today's executions with commissions |

### Contract Discovery
| Tool | Description |
|------|-------------|
| `ibkr_get_option_params` | Available strikes and expirations |
| `ibkr_get_contract_details` | Detailed contract info (conId, etc.) |

## Development Commands

### Setup
```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # Linux/Mac
# venv\Scripts\activate   # Windows

# Install dependencies
pip install -r requirements.txt

# Install in development mode
pip install -e .
```

### Running the Server
```bash
# Direct execution
python src/ibkr_mcp_server.py

# Or via entry point (after pip install -e .)
ibkr-mcp-server
```

### Testing
```bash
# Run all tests
python -m pytest tests/test_ibkr_server.py -v

# Run with coverage
python -m pytest tests/test_ibkr_server.py --cov=src

# Run integration tests (requires running TWS)
RUN_INTEGRATION_TESTS=1 python -m pytest tests/test_ibkr_server.py -v -m integration
```

## Code Conventions

### Async Patterns
- All bridge methods that interact with TWS use async/await
- Events are managed via `asyncio.Event()` for signaling
- Timeouts are used for all async operations (default: 5-10 seconds)

### Error Handling
- All tool handlers check `bridge.connected` before operations
- Return JSON with `{"error": "message"}` on failures
- Logging via `logger = logging.getLogger("ibkr-mcp")`

### Order Fixes
The `fix_order_attributes()` method addresses deprecated TWS attributes:
- Sets `eTradeOnly = False`
- Sets `firmQuoteOnly = False`
- Prevents errors 10268, 10269, 10270 on TWS 983+

### Data Formats
- Dates: `YYYYMMDD` format (e.g., "20251207")
- Times: ISO 8601 format
- All tool responses are JSON formatted with `indent=2`

## Important Notes for AI Assistants

### Before Making Changes
1. Always read the relevant section of `src/ibkr_mcp_server.py` first
2. Check test file for expected behaviors
3. Understand the async event flow for new features

### Common Modifications
- **Adding a new tool**:
  1. Add to `handle_list_tools()` with inputSchema
  2. Add handler in `handle_call_tool()`
  3. Add async method to `IBKRBridge` if needed

- **Modifying order handling**:
  1. Check `fix_order_attributes()` for compatibility
  2. Test with `transmit=False` first

### Market-Specific Settings
- **US Stocks/Options**: exchange="SMART", currency="USD"
- **SPX Index**: exchange="CBOE", currency="USD"
- **DAX Options**: exchange="EUREX", currency="EUR", trading_class="ODAX"

### Security Considerations
- Never commit `.env` files with credentials
- Use paper trading (port 7497) for testing
- `transmit=False` by default for order review in TWS
- API credentials are managed by TWS, not this server

## Configuration

### Claude Desktop Integration
Add to `claude_desktop_config.json`:
```json
{
  "mcpServers": {
    "ibkr": {
      "command": "python",
      "args": ["/path/to/ibkr-mcp-server/src/ibkr_mcp_server.py"]
    }
  }
}
```

### Environment Variables (optional)
See `.env.template`:
- `TWS_HOST`: TWS hostname (default: 127.0.0.1)
- `TWS_PAPER_PORT`: Paper trading port (default: 7497)
- `TWS_LIVE_PORT`: Live trading port (default: 7496)
- `MAX_RISK_PER_TRADE`: Risk limit (default: 0.02 = 2%)

## Troubleshooting

### Connection Issues
- Ensure TWS/IB Gateway is running
- Enable API in TWS: File > Global Configuration > API > Settings
- Check port matches (7497 for paper, 7496 for live)
- Add 127.0.0.1 to trusted IPs in TWS

### Market Data Issues
- Verify market data subscriptions in TWS
- Check market hours for the instrument
- For options, ensure correct expiry format (YYYYMMDD)

### Order Errors
- Error 10268/10269/10270: Fixed by `fix_order_attributes()`
- "Insufficient buying power": Check account balance
- "No security definition": Verify contract parameters

## Recent Changes

See git log for recent commits. Key updates include:
- Order transmit functionality
- Vertical spread (combo) orders
- DAX option chain fixes
- Currency configuration updates

## Contributing

1. Create feature branch from main
2. Add tests for new functionality
3. Ensure all tests pass
4. Update this CLAUDE.md if adding tools/features
5. Submit PR with clear description
