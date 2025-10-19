# varon.fi
Advanced AI Crypto Trading

## 1. Executive Summary
This refined plan optimizes the design for a Python 3-based algorithmic trading platform focused on short-term, high-leverage scalping in cryptocurrency perpetual futures (perps). The primary DEX is Hyperliquid, selected for its on-chain order book, zero gas fees, support for up to 50x leverage on major pairs (e.g., BTC/USDT, ETH/USDT), and recent HIP-3 upgrade enabling user-deployed perps. Extensibility to other DEXes like dYdX or GMX is maintained via abstracted interfaces.

Optimizations include:
- Unified IPC: Exclusively gRPC for all interprocess communication, leveraging its efficiency, bidirectional streaming, and strong typing to reduce complexity from multiple protocols. gRPC supports low-latency streaming for realtime data and unary calls for signals, aligning with trading system needs for performance and reliability.
- Ambiguity Elimination: Precise library choices (e.g., Hyperliquid Python SDK v0.15.0+), data sources, schema definitions, and workflows based on industry standards from frameworks like Freqtrade and best practices in low-latency systems.
- Focus: High-frequency scalping strategies (e.g., EMA crossovers, order book imbalances, funding rate arbitrage) with built-in risk controls to mitigate liquidation risks in high-leverage environments.

The architecture draws from microservices patterns, ensuring modularity, scalability, and testability while prioritizing security and performance.

## 2. Project Scope and Assumptions
- **In-Scope**: Realtime data aggregation from specified sources; composable strategy framework with backtesting and live execution; order management on Hyperliquid and extensible DEXes; risk management (e.g., position limits, stop-loss enforcement).
- **Out-of-Scope**: Graphical UI; regulatory compliance beyond basic logging; infrastructure orchestration (e.g., Kubernetes); advanced ML integration.
- **Assumptions**: Hyperliquid API access via Python SDK; secure management of API private keys (e.g., via environment variables or vaults); platform deployment on a single machine or cloud VM with low-latency network; users accept high-leverage risks (e.g., potential liquidation at 50x).
- **Constraints**: Adhere to API rate limits (e.g., Hyperliquid's 100 requests/second); no runtime package installations; target sub-100ms end-to-end latency for scalping.

## 3. Architecture Principles
- **Modularity**: Three core modules (data, strategy, execution) as independent Python processes, each with defined gRPC interfaces for loose coupling.
- **Performance and Scalability**: Asynchronous Python via asyncio; gRPC for efficient protobuf-based communication; horizontal scaling by running multiple instances (e.g., per strategy).
- **IPC Details**: gRPC exclusively:
  - Server-side streaming for realtime data feeds (e.g., data module streams aggregated ticks to strategies).
  - Unary RPC for discrete events (e.g., strategy sends trade signals to execution).
  - Bidirectional streaming for interactive needs (e.g., execution status updates).
  - No gRPC-web needed, as this is backend-only; use standard gRPC with Python's grpcio library.
- **Configuration**: Single TOML file per module, loaded at startup with validation; no hot-reload to avoid runtime errors—restart processes for changes.
- **Database**: PostgreSQL with TimescaleDB extension for time-series optimization, enabling fast queries on large datasets (e.g., continuous aggregation for OHLCV bars).
- **Security and Reliability**: API keys stored in environment variables; exponential backoff for API retries; circuit breakers to halt trading on excessive losses (>5% daily drawdown); structured logging with logging library.
- **Best Practices**: Separation of concerns; mandatory backtesting before live deployment; risk diversification (e.g., limit to 5 concurrent positions); use TA-Lib for indicators to ensure accuracy.

## 4. Core Modules
### 4.1 Data Acquisition and Aggregation
- **Purpose**: Collect, process, and distribute realtime market data for scalping, ensuring <50ms processing latency.
- **Key Features**:
  - Sources: Hyperliquid API/SDK (primary for perps mids, order books, funding rates); CoinAPI or Amberdata (secondary for cross-market prices and historical backfills); no more than 3 sources to limit complexity.
  - Aggregation: Use pandas for OHLCV bars (1s-1m intervals) and TA-Lib for initial indicators (e.g., EMA); validate data for gaps or anomalies.
  - Storage: Insert into PostgreSQL/TimescaleDB hypertables; retain 7 days of tick data, archive older.
  - Distribution: gRPC server-side streaming endpoint for subscribers (e.g., strategies request streams by symbol).
- **Workflow**:
  1. Async fetch via aiohttp and Hyperliquid SDK.
  2. Aggregate in-memory, then batch-insert to DB every 10s.
  3. Stream updates to connected clients.
- **Best Practices**: Timestamp synchronization using UTC; handle disconnections with 5s reconnect intervals; data validation to discard outliers (>5% deviation).

### 4.2 Strategy Development and Execution
- **Purpose**: Build and run composable scalping strategies, generating precise signals with risk parameters.
- **Key Features**:
  - Composability: Implement Strategy Pattern—a base Strategy class with pluggable components:
    - Indicators: TA-Lib wrappers (e.g., EMA(5,15), RSI(14)).
    - Filters: Volume threshold (>1000 units), liquidity check (order book depth >10x position size).
    - Risk Modules: Fixed position sizing (0.5% of portfolio), max leverage (50x), mandatory stop-loss (1% below entry).
  - Configuration: TOML defines assembly (e.g., "ema_crossover + volume_filter + stop_loss").
  - Testing: Backtest mode queries historical DB data; paper trading simulates execution without real orders.
  - Execution: Process streamed data, generate signals (e.g., JSON: {"symbol": "BTC/USDT", "side": "buy", "leverage": 50, "size": 0.005, "stop_loss": 0.99*entry}).
- **Workflow**:
  1. Connect to data module via gRPC stream.
  2. Apply components sequentially to input data.
  3. If signal valid, send via unary gRPC to execution module.
- **Best Practices**: Limit to 3-5 components per strategy to prevent overfitting; out-of-sample testing on 20% holdout data; log all signals for audits.

### 4.3 Order Execution and Management
- **Purpose**: Execute signals on DEXes, managing positions with minimal slippage.
- **Key Features**:
  - Order Handling: Use Hyperliquid SDK for market/limit/stop orders; track via API polling (every 1s).
  - Account Management: Monitor balance, positions, funding (every 60s); support one wallet per DEX.
  - Liquidity: Pre-check order book depth via API; reject if insufficient (>2x size at <0.5% slippage).
  - Extensibility: Abstract base class (e.g., DexExecutor) with Hyperliquid implementation; add adapters for dYdX/GMX.
- **Workflow**:
  1. gRPC server listens for unary signal calls.
  2. Validate against risk limits (e.g., total exposure <20% portfolio).
  3. Place order, monitor until filled/canceled.
  4. Log to DB; send status via bidirectional gRPC stream if requested.
- **Best Practices**: Use limit orders for scalping to control entry; enforce kill switch on >10% loss; audit trails for all actions.

## 5. Database and Configuration Structure
- **PostgreSQL/TimescaleDB Schemas** (Precise Definitions):
  - market_data (hypertable): timestamp TIMESTAMPTZ (partition key), symbol VARCHAR(20), open NUMERIC(18,8), high NUMERIC(18,8), low NUMERIC(18,8), close NUMERIC(18,8), volume NUMERIC(18,8); index on (symbol, timestamp DESC).
  - trades: id SERIAL PRIMARY KEY, strategy_id INT, symbol VARCHAR(20), side ENUM('buy','sell'), leverage INT, size NUMERIC(18,8), entry_price NUMERIC(18,8), exit_price NUMERIC(18,8), pnl NUMERIC(18,8), timestamp TIMESTAMPTZ.
  - logs: id SERIAL PRIMARY KEY, level ENUM('info','warn','error'), message TEXT, timestamp TIMESTAMPTZ.
- **TOML Configuration Example Structure** (One File Per Module):
  - data.toml: db_url="postgresql://user:pass@host:5432/db", sources=["hyperliquid","coinapi"], api_keys={"hyperliquid":"env:HL_KEY"}, grpc_port=50051.
  - strategy.toml: components=["ema:5,15","volume:1000","stop:0.01"], max_leverage=50, portfolio_size=10000.
  - execution.toml: dex="hyperliquid", wallet_private="env:WALLET_KEY", max_exposure=0.2.

## 6. Implementation Roadmap
1. **Setup (Weeks 1-2)**: Project skeleton; install dependencies (grpcio, pandas, ta-lib, psycopg2, hyperliquid-python-sdk); define gRPC protos (e.g., DataStream, TradeSignal).
2. **Data Module (Weeks 3-4)**: Implement fetching/aggregation; TimescaleDB setup; gRPC streaming server.
3. **Strategy Module (Weeks 5-6)**: Strategy class with components; backtesting logic; gRPC client for data, server for signals.
4. **Execution Module (Weeks 7-8)**: SDK integration; order logic; gRPC server for signals.
5. **Integration/Testing (Weeks 9-10)**: End-to-end tests with simulated data; performance benchmarks.
6. **Optimization/Deployment (Weeks 11-12)**: Latency tuning; security review; Docker packaging.

## 7. Risks and Mitigations
- **Latency**: Monitor with logging; mitigate via async and gRPC optimizations.
- **API Failures**: Exponential backoff (1s base, max 30s); fallback to secondary sources.
- **Liquidation**: Enforce stops and leverage caps; simulate in backtests.
- **Security**: Key vaulting; input validation on gRPC calls.
- **Overfitting**: Mandatory holdout testing; limit strategy complexity.

## 8. Future Enhancements
- Add dYdX/GMX adapters.
- Integrate basic ML (e.g., scikit-learn for signal weighting).
- Monitoring via Prometheus.
- HIP-3 support for custom perps if staking thresholds met.