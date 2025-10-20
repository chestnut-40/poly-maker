# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Poly-Maker is an automated market making bot for Polymarket prediction markets. The bot provides liquidity by maintaining orders on both sides of the order book with configurable parameters, position management, and risk controls.

## Package Manager

This project uses `pip` for Python dependencies and `npm` for the poly_merger Node.js component.

## Key Commands

### Installation
```bash
# Install Python dependencies
pip install -r requirements.txt

# Install Node.js dependencies for position merger
cd poly_merger && npm install && cd ..
```

### Running the Bot
```bash
# Start the market making bot (main process)
python main.py

# Update market data (run separately, preferably on different IP)
python update_markets.py

# Update account statistics
python update_stats.py
```

## Architecture

### Core Components

**main.py**: Application entry point that orchestrates the bot lifecycle:
- Initializes the `PolymarketClient` and loads initial data
- Spawns a background thread (`update_periodically`) that refreshes positions, orders, and market data every 5-30 seconds
- Maintains two persistent WebSocket connections: market updates (order book) and user updates (trades/orders)
- Cleans up stale pending trades after 15 seconds to prevent stuck states

**trading.py**: Main trading logic implementing market making strategy via `perform_trade(market)`:
- **Position Merging**: Automatically merges opposing positions (YES+NO) when both exceed `MIN_MERGE_SIZE` (20 USDC) to free capital
- **Order Management**: Places buy/sell orders based on order book depth, spread, and existing positions
- **Risk Controls**:
  - **Stop-loss**: Exits positions when PnL drops below threshold AND spread is tight enough
  - **Take-profit**: Sells positions when price reaches `avgPrice + take_profit_threshold%`
  - **Volatility filtering**: Blocks new positions when 3-hour volatility exceeds threshold
  - **Risk-off periods**: After stop-loss, pauses trading for configurable hours
- **Smart Order Updates**: Only cancels/recreates orders when price changes >0.5¢ or size changes >10%
- Uses per-market locks to prevent concurrent trading on the same market

**poly_data/polymarket_client.py**: `PolymarketClient` wrapper around py-clob-client:
- Authenticates with Polymarket API using private key and wallet address
- Creates/cancels orders via `create_order()` and `cancel_all_asset()`/`cancel_all_market()`
- Queries order books, positions, and balances
- Executes position merges by calling the external Node.js script via subprocess

**poly_data/websocket_handlers.py**: WebSocket connection management:
- `connect_market_websocket(tokens)`: Subscribes to order book updates for specified tokens, triggers `process_data()` → `perform_trade()`
- `connect_user_websocket()`: Authenticates with API credentials, processes trade fills and order updates via `process_user_data()`
- Both auto-reconnect on disconnection with 5-second delay

**poly_data/data_processing.py**: WebSocket message processing:
- `process_data()`: Maintains in-memory order books using SortedDict for efficient price level lookups
- `process_user_data()`: Updates local position/order tracking when trades execute or orders are placed/cancelled
- Tracks pending trades in `global_state.performing` to prevent concurrent modifications

**poly_data/data_utils.py**: State management for positions and orders:
- `update_positions()`: Fetches positions from API, intelligently updates local state (skips if trades pending)
- `update_orders()`: Syncs open orders from API, cancels if multiple orders exist for same token
- `set_position()`: Updates position size and weighted average price when trades execute
- `update_markets()`: Loads market configuration and trading parameters from Google Sheets

**poly_data/global_state.py**: Shared state across modules:
- `client`: PolymarketClient instance
- `df`: DataFrame of markets to trade (from Google Sheets "Selected Markets")
- `params`: Trading parameters by market type (from "Hyperparameters" sheet)
- `positions`: Dict of `{token_id: {'size': float, 'avgPrice': float}}`
- `orders`: Dict of `{token_id: {'buy': {...}, 'sell': {...}}}`
- `all_data`: In-memory order books from WebSocket
- `performing`: Tracks pending trades to prevent race conditions
- `REVERSE_TOKENS`: Maps each token to its opposite outcome

**poly_data/trading_utils.py**: Market making calculations:
- `get_best_bid_ask_deets()`: Analyzes order book to find best bid/ask, depth, and liquidity ratios
- `get_order_prices()`: Calculates optimal bid/ask prices based on order book and position
- `get_buy_sell_amount()`: Determines order sizes based on position limits and existing exposure

**poly_merger/**: Node.js module for on-chain position merging:
- Calls Polymarket smart contracts to merge opposite positions (YES+NO → USDC)
- Invoked via subprocess from Python when merge conditions are met
- Requires private key in .env

**update_markets.py**: Market data collection script (runs independently):
- Fetches all available Polymarket markets via API
- Calculates volatility metrics (1h, 3h, 6h, 12h, 24h, 7d, 30d)
- Computes maker rewards per $100 liquidity
- Updates Google Sheets with three worksheets:
  - "All Markets": All markets sorted by maker rewards
  - "Volatility Markets": Low-volatility markets (<20 volatility_sum)
  - "Full Markets": Complete market data
- Runs continuously with 1-hour refresh cycle

**poly_stats/account_stats.py**: Portfolio tracking and statistics

### Google Sheets Integration

The bot is entirely configured via Google Sheets:

1. **Selected Markets**: Markets the bot trades (copied from "Volatility Markets")
2. **Hyperparameters**: Trading parameters by `param_type`:
   - `trade_size`: Base order size in USDC
   - `max_size`: Maximum position size per outcome
   - `min_size`: Minimum order size threshold
   - `max_spread`: Maximum spread to quote
   - `stop_loss_threshold`: PnL % to trigger exit
   - `take_profit_threshold`: PnL % to take profit
   - `spread_threshold`: Max spread for stop-loss execution
   - `volatility_threshold`: Max 3-hour volatility to allow new positions
   - `sleep_period`: Hours to pause after stop-loss
3. **All Markets**: Database of all available markets
4. **Volatility Markets**: Filtered markets with low volatility

Access requires Google Service Account credentials (JSON file) with edit permissions on the sheet.

### Trading Flow

1. **Initialization**: Load markets, positions, and orders from API and Google Sheets
2. **WebSocket Updates**: Receive order book changes → trigger `perform_trade(market)`
3. **Position Merging**: If holding both YES and NO >= 20 USDC, merge to recover collateral
4. **For each outcome (YES/NO)**:
   - Fetch order book depth and current position
   - Calculate optimal bid/ask prices based on order book
   - **If position < max_size and buy_amount > min_size**:
     - Check if in risk-off period (skip if yes)
     - Check volatility and price deviation (cancel orders if too high)
     - Place/update buy order if: better price available, position+orders < 95% of max_size, or existing order too large
   - **If position > 0 (holding)**:
     - Calculate take-profit price: `avgPrice + (avgPrice * take_profit_threshold%)`
     - Place/update sell order if: price deviation >2%, or order size < 97% of position
     - **Stop-loss check**: If PnL < threshold AND spread <= threshold, sell at market and enter risk-off period
4. **Order Optimization**: Only cancel/recreate orders when meaningful changes occur (>0.5¢ price or >10% size)

### Data Flow

```
Google Sheets (config) → update_markets.py → Google Sheets (market data)
                                              ↓
                                        main.py (loads config)
                                              ↓
WebSocket (market) → process_data → perform_trade → PolymarketClient → Polymarket API
WebSocket (user) → process_user_data → set_position/set_order
                                              ↓
                                    global_state (positions/orders)
```

### State Management

The bot maintains local state for performance and uses several strategies to keep it synchronized:

- **Positions**: Updated every 5 seconds from API (unless trades pending), immediately on WebSocket trade fills
- **Orders**: Updated every 5 seconds from API, immediately on WebSocket order events
- **Pending Trades**: Tracked in `global_state.performing` with timestamps, auto-cleaned after 15 seconds
- **Order Books**: Real-time via WebSocket, stored as SortedDict for efficient lookups

### Risk Management

Multiple layers of protection:

1. **Position Limits**: `max_size` per outcome (default ~trade_size), absolute cap of 250 USDC
2. **Volatility Filtering**: Block new positions when 3-hour volatility exceeds threshold
3. **Price Deviation**: Cancel orders if price moves >5¢ from reference
4. **Stop-Loss**: Exit when PnL < threshold (e.g., -2%) and spread is tight
5. **Risk-Off Periods**: After stop-loss, pause trading for N hours
6. **Reverse Position Check**: Don't buy YES if holding significant NO position
7. **Liquidity Ratio**: Skip buy orders if bid_sum/ask_sum < 0 (more sellers than buyers)
8. **Price Range**: Only place orders between 0.1 and 0.9 to avoid extreme positions

### Constants

**poly_data/CONSTANTS.py**:
- `MIN_MERGE_SIZE = 20`: Minimum USDC value to trigger position merging

## Environment Variables

Required in `.env`:

- `PK`: Private key for Polymarket wallet
- `BROWSER_ADDRESS`: Wallet address (must have done at least one trade via UI for proper permissions)
- `SPREADSHEET_URL`: Google Sheets URL for configuration

The Google Service Account JSON credentials file must be in the main directory.

## Important Implementation Details

- **Concurrency**: Uses asyncio with per-market locks in `perform_trade()` to prevent race conditions
- **Order Book Storage**: SortedDict for O(log n) price level access instead of O(n) lists
- **Memory Management**: Explicit `gc.collect()` calls to prevent memory leaks in long-running process
- **Stale Trade Cleanup**: Background thread removes pending trades after 15 seconds to recover from failures
- **Smart Order Updates**: Avoids excessive API calls by only updating orders when changes are meaningful
- **Negative Risk Markets**: Special handling via `neg_risk` parameter (uses different smart contract)
- **Position Merging**: Calls external Node.js script because Polymarket's merge contracts use complex EVM signatures

## Common Development Patterns

### Adding a New Trading Parameter

1. Add column to Google Sheets "Hyperparameters" worksheet
2. Update `poly_data/utils.py` `get_sheet_df()` to parse the parameter
3. Access via `global_state.params[param_type]['new_parameter']` in `trading.py`

### Modifying Trading Logic

All market making logic is in `trading.py` `perform_trade()`. Key sections:
- Lines 168-186: Position merging
- Lines 346-430: Buy order logic (including risk checks)
- Lines 431-463: Sell order logic (take-profit)
- Lines 321-344: Stop-loss logic

### Testing Order Book Calculations

Use `poly_data/trading_utils.py` functions with test data:
```python
from poly_data.trading_utils import get_best_bid_ask_deets
deets = get_best_bid_ask_deets(market, 'token1', min_size=100, percentage=0.1)
```

### Debugging WebSocket Issues

WebSocket handlers log connection events and auto-reconnect. Check:
- `connect_market_websocket()` and `connect_user_websocket()` in `poly_data/websocket_handlers.py`
- `process_data()` and `process_user_data()` in `poly_data/data_processing.py`
- Enable print statements in these functions to trace message flow

## Project Structure

```
poly-maker/
├── main.py                 # Entry point, WebSocket orchestration
├── trading.py              # Core market making logic
├── update_markets.py       # Market data collection (separate process)
├── update_stats.py         # Account statistics
├── poly_data/              # Core trading engine
│   ├── polymarket_client.py       # API client wrapper
│   ├── websocket_handlers.py      # WebSocket connections
│   ├── data_processing.py         # Message processing
│   ├── data_utils.py              # State management
│   ├── trading_utils.py           # Market making calculations
│   ├── global_state.py            # Shared state
│   ├── utils.py                   # Google Sheets integration
│   ├── abis.py                    # Smart contract ABIs
│   └── CONSTANTS.py               # Configuration constants
├── poly_merger/            # Position merging utility (Node.js)
│   ├── merge.js                   # Smart contract interaction
│   └── package.json
├── poly_stats/             # Portfolio tracking
│   └── account_stats.py
├── poly_utils/             # Shared utilities
│   └── google_utils.py
├── data_updater/           # Market data fetching (separate module)
│   ├── find_markets.py
│   ├── trading_utils.py
│   └── google_utils.py
└── requirements.txt        # Python dependencies
```

## Safety Notes

- This bot trades real money on live markets
- Always test with small `trade_size` values first
- The bot will accumulate positions over time if not profitable
- Monitor the Google Sheets position tracking regularly
- `update_markets.py` should run on a different IP address than `main.py` to avoid rate limiting
- Wallet must have USDC balance and existing Polymarket permissions (do at least one manual trade first)
- Never commit `.env` file or Google Service Account credentials
