# Trading Bot Developer Guide

## Overview

This trading bot listens to 13 different websocket feeds and processes trading signals in real-time. This guide will help you understand how to modify, add, or remove features.

## Architecture Overview

The bot has three main parts:
1. **Websocket Listeners** - Collect data from various sources
2. **Signal Processors** - Analyze data and create trading signals
3. **Order Manager** - Execute trades on Binance

## Current Websocket Feeds

### 1. BWE News WebSocket
- **Purpose**: General crypto news and alerts
- **Location**: `src/feeds/` (generic handler)
- **Config**: `BWE_WS_URL`, `BWE_WS_ENABLED`
- **Default URL**: `wss://bwenews-api.bwe-ws.com/ws`

### 2. Tokyo WebSocket (Tree-of-Alpha)
- **Purpose**: Upbit listings and Elon Musk tweets
- **Location**: `src/feeds/upbit_via_tokyo.py`
- **Config**: `TOA_TOKYO_WS_URL`, `TOA_TOKYO_API_KEY`, `TOA_TOKYO_WS_ENABLED`
- **Default URL**: `ws://tokyo.treeofalpha.com:5124`
- **Special Features**: Handles Upbit listing detection and automated trading

### 3. Tree-of-Alpha News WebSocket
- **Purpose**: General crypto news
- **Location**: Generic handler
- **Config**: `TOA_NEWS_WS_URL`, `TOA_NEWS_API_KEY`, `TOA_NEWS_WS_ENABLED`
- **Default URL**: `wss://news.treeofalpha.com/ws`

### 4. Phoenix News WebSocket
- **Purpose**: High-priority crypto news
- **Location**: Generic handler
- **Config**: `PHOENIX_WS_URL`, `PHOENIX_API_KEY`, `PHOENIX_WS_ENABLED`
- **Default URL**: `wss://wss.phoenixnews.io`

### 5. Synoptic WebSocket
- **Purpose**: Economic data (NFP, unemployment rates)
- **Location**: `src/feeds/synoptic_listener.py`
- **Config**: `SYNOPTIC_WS_URL`, `SYNOPTIC_API_KEY`, `SYNOPTIC_WS_ENABLED`
- **Default URL**: `wss://synoptic.com/v1/ws/on-stream-post`
- **Special Features**: Processes NFP and unemployment data for macro trading

### 6. Twitter Monitor
- **Purpose**: Monitors specific Twitter accounts
- **Location**: `src/feeds/twitter_hunter.py`
- **Config**: `TWITTER_MONITOR_ENABLED`, `TWITTER_SCREEN_NAME`, `TWITTER_TARGET_PHRASE`
- **Default Target**: `@coinbaseassets` for roadmap updates

### 7-13. Additional Signal Channels
The bot also processes various other signal sources through the main signal processing system.

## How to Add a New Websocket Feed

### Step 1: Create Configuration
Add your websocket config to `src/core/config.py`:

```python
# In DataFeedsConfig class
your_feed: WebSocketFeedConfig = field(default_factory=lambda: WebSocketFeedConfig("", "Your Feed"))

# In from_env method
your_feed=WebSocketFeedConfig(
    url=os.getenv('YOUR_FEED_WS_URL', 'wss://your-websocket-url.com'),
    name="Your Feed Websocket",
    api_key=os.getenv('YOUR_FEED_API_KEY'),
    enabled=get_bool_env('YOUR_FEED_WS_ENABLED', True)
)
```

### Step 2: Create WebSocket Listener
Add to `src/core/background_tasks.py`:

```python
# In _start_websocket_listeners method
if self.config.data_feeds.your_feed.enabled:
    self.tasks["your_feed_listener"] = asyncio.create_task(
        self._your_feed_websocket_listener()
    )

# Add the listener method
async def _your_feed_websocket_listener(self):
    """Your Feed WebSocket listener."""
    ws_client = self.api_factory.get_websocket_feed_client('your_feed')
    
    while self.running:
        try:
            if not await ws_client.connect():
                await asyncio.sleep(5)
                continue
            
            logger.info("⚡ Your Feed WebSocket connected")
            
            async for message in ws_client.listen():
                signal_data = {"source": "YOUR_FEED", "raw_data": message}
                await self.signal_queue.put(signal_data)
                
        except Exception as e:
            logger.error(f"❌ Your Feed WebSocket error: {e}")
            await asyncio.sleep(5)
```

### Step 3: Create Signal Handler (Optional)
If you need custom processing, create a file in `src/feeds/your_feed_handler.py`:

```python
import asyncio
import json
import logging

logger = logging.getLogger(__name__)

async def handle_your_feed_message(message):
    """Process messages from your feed"""
    try:
        # Parse your message format
        data = json.loads(message)
        
        # Extract trading signals
        if "buy_signal" in data:
            # Create trading signal
            from ..core.lowcapbot import place_futures_order
            
            result = await place_futures_order(
                symbol=data["symbol"],
                side="BUY",
                position_size_usd=data["amount"],
                slippage_percent=2.0
            )
            
            if result:
                logger.info(f"✅ Order placed from your feed: {data['symbol']}")
                
    except Exception as e:
        logger.error(f"Error processing your feed message: {e}")
```

### Step 4: Update Environment Variables
Add to your `.env` file:

```bash
YOUR_FEED_WS_URL=wss://your-websocket-url.com
YOUR_FEED_API_KEY=your_api_key_here
YOUR_FEED_WS_ENABLED=true
```

## How to Modify Existing Feeds

### Changing Signal Processing
1. Find the relevant handler in `src/feeds/`
2. Modify the message processing logic
3. Update trading parameters if needed

### Example: Modify Twitter Monitor
Edit `src/feeds/twitter_hunter.py`:

```python
# Change the target phrase
TARGET_PHRASE = "Your new target phrase"

# Modify the coin extraction logic
def extract_coin_symbol(text):
    # Your custom extraction logic
    pass
```

### Changing Position Sizes
Edit `src/core/config.py` in `PositionSizesConfig`:

```python
@dataclass
class PositionSizesConfig:
    your_signal: float = 50000  # $50k position size
```

Add to `.env`:
```bash
POSITION_SIZE_YOUR_SIGNAL=50000
```

## How to Remove a Websocket Feed

### Step 1: Disable in Configuration
Set in `.env`:
```bash
YOUR_FEED_WS_ENABLED=false
```

### Step 2: Remove from Background Tasks
Comment out or remove from `src/core/background_tasks.py`:

```python
# Remove or comment out
# if self.config.data_feeds.your_feed.enabled:
#     self.tasks["your_feed_listener"] = asyncio.create_task(
#         self._your_feed_websocket_listener()
#     )
```

### Step 3: Remove Handler Files
Delete the handler file if it exists:
```bash
rm src/feeds/your_feed_handler.py
```

## Signal Processing System

### How Signals Flow
1. **WebSocket** receives message
2. **Background Task** puts signal in queue
3. **Signal Worker** processes the signal
4. **Signal Strategy** analyzes the signal
5. **Order Manager** executes trades

### Signal Data Format
All signals follow this format:
```python
{
    "source": "WEBSOCKET_NAME",
    "raw_data": original_message,
    "timestamp": datetime.now(),
    "processed": False
}
```

### Adding Custom Signal Processing
Create a new strategy in `src/signals/signal_strategies.py`:

```python
class YourSignalStrategy(SignalStrategy):
    async def can_process(self, signal_data: Dict) -> bool:
        return signal_data.get("source") == "YOUR_FEED"
    
    async def process(self, signal_data: Dict) -> Optional[List[Dict]]:
        # Your processing logic here
        return [{
            "pair": "BTCUSDT",
            "side": "BUY",
            "position_size_usd": 1000,
            "slippage_percent": 2.0
        }]
```

## Trading System

### Position Sizes
All position sizes are configured in `.env`:
```bash
POSITION_SIZE_DEFAULT=14567
POSITION_SIZE_BTCUSDT=250000
POSITION_SIZE_ETHUSDT=350000
```

### Order Execution
The main trading function is in `src/core/lowcapbot.py`:

```python
await place_futures_order(
    symbol="BTCUSDT",
    side="BUY",
    position_size_usd=250000,
    slippage_percent=2.0,
    max_retry_slippage=10.0,
    retry_backoff=1.5
)
```

### Margin Management
The bot automatically:
- Calculates required margin
- Adjusts leverage (up to 125x)
- Reduces position size if needed
- Retries with smaller sizes

## Debugging and Monitoring

### Log Levels
- **INFO**: Normal operations
- **WARNING**: Potential issues
- **ERROR**: Failed operations
- **DEBUG**: Detailed information

### Key Log Messages
- `⚡ WebSocket connected`: Feed connected
- `✅ Order placed`: Successful trade
- `❌ Order failed`: Failed trade
- `🔧 RETRY`: Retrying with adjusted parameters

### Monitoring WebSocket Status
Check logs for connection status:
```bash
journalctl -u trading-bot -f | grep "WebSocket"
```

## Testing New Features

### Test Mode
Set in `.env`:
```bash
BINANCE_TESTNET=true
```

### Mock Signals
Use the API endpoint to test signals:
```bash
curl -X POST http://localhost:8000/api/test-signal \
  -H "Content-Type: application/json" \
  -d '{"source": "TEST", "symbol": "BTCUSDT", "side": "BUY"}'
```

### Dry Run Mode
Disable actual trading:
```bash
TRADING_ENABLED=false
```

## Common Issues and Solutions

### WebSocket Connection Fails
1. Check API keys in `.env`
2. Verify WebSocket URL is correct
3. Check network connectivity
4. Review rate limits

### Signal Not Processing
1. Check if WebSocket is enabled
2. Verify signal format
3. Check processing logic
4. Review queue status

### Orders Not Executing
1. Check Binance API credentials
2. Verify margin availability
3. Check position size limits
4. Review leverage settings

## File Structure

```
refactored-version/
├── src/
│   ├── core/
│   │   ├── background_tasks.py     # WebSocket management
│   │   ├── config.py               # Configuration
│   │   ├── lowcapbot.py           # Trading logic
│   │   └── ...
│   ├── feeds/
│   │   ├── twitter_hunter.py      # Twitter monitoring
│   │   ├── synoptic_listener.py   # Economic data
│   │   ├── upbit_via_tokyo.py     # Upbit listings
│   │   └── ...
│   ├── signals/
│   │   ├── signal_strategies.py   # Signal processing
│   │   └── ...
│   └── ...
├── main.py                        # Application entry point
├── .env                          # Environment variables
└── README.md                     # This guide
```

## Quick Reference

### Start the Bot
```bash
source venv/bin/activate
python3 -m uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```

### View Logs
```bash
journalctl -u trading-bot -f
```

### Test Configuration
```bash
python3 -c "from src.core.config import get_config; print(get_config())"
```

### Check WebSocket Status
```bash
curl http://localhost:8000/api/status
```

### Log Management

The bot automatically cleans up old logs every 24 hours to prevent disk space issues.

**Configuration (.env):**
```bash
LOG_CLEANUP_ENABLED=true           # Enable/disable log cleanup
LOG_CLEANUP_INTERVAL_HOURS=24      # How often to run cleanup (hours)
LOG_CLEANUP_KEEP_DAYS=7            # Keep logs for X days
LOG_CLEANUP_MAX_SIZE_MB=1000       # Max systemd log size (MB)
LOG_CLEANUP_SYSTEM_LOGS=true       # Clean systemd journal logs
LOG_CLEANUP_APPLICATION_LOGS=true  # Clean application log files
```

**Manual cleanup:**
```bash
# Clean systemd logs manually
sudo journalctl --vacuum-time=7d

# Check disk usage
df -h
du -sh /var/log/*
```

---

This guide covers the basics of working with the trading bot. For specific implementation details, refer to the code comments and existing examples in the codebase. 
