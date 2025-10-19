# varon.fi - AI Trading Platform Design Document

## Executive Summary

varon.fi is a high-performance algorithmic trading platform designed for high-frequency cryptocurrency trading with machine learning strategies. The platform focuses on reliable data sources (Coinbase, major exchanges) and executes trades on Hyperliquid DEX for perpetual futures with up to 50x leverage.

**Core Philosophy**: Modular, ML-first architecture that abstracts data collection, model training, signal generation, and order execution to enable rapid iteration on diverse trading strategies.

## Platform Architecture

### Design Principles

1. **Modularity**: Four core modules with clear interfaces
2. **ML-First**: Built for machine learning strategy development and deployment
3. **High-Frequency**: Sub-100ms latency for scalping strategies
4. **Reliable Data**: Primary focus on Coinbase and major exchange data
5. **Risk Management**: Built-in position limits and circuit breakers

### Core Modules

#### 1. Data Collection Module
**Purpose**: Aggregate reliable market data from major exchanges

**Key Features**:
- **Primary Sources**: Coinbase WebSocket feeds, major exchange APIs
- **Data Types**: OHLCV, order book snapshots, funding rates, volume data
- **Storage**: PostgreSQL with TimescaleDB for time-series optimization
- **Distribution**: gRPC streaming for real-time data feeds

**Current Implementation**:
- `coinbase/` - Coinbase WebSocket data collector
- `dydx/` - dYdX integration (secondary)
- `postgres/` - Database schema and migrations

#### 2. ML Strategy Module
**Purpose**: Develop, train, and deploy machine learning trading strategies

**Key Features**:
- **Strategy Types**: DRQN, DQN, Rainbow DQN, and custom ML models
- **Training Pipeline**: Backtesting, validation, and model persistence
- **Signal Generation**: Real-time inference and signal broadcasting
- **Composability**: Modular strategy components and risk modules

**Current Implementation**:
- `poc/` - DRQN proof-of-concept with 500%+ outperformance
- `models/` - Enhanced ML models and training pipelines
- `trading-bot/` - Legacy trading bot implementations

#### 3. Signal Execution Module
**Purpose**: Execute trading signals on DEXes with risk management

**Key Features**:
- **Primary DEX**: Hyperliquid (on-chain order book, zero gas fees)
- **Order Types**: Market, limit, stop-loss orders
- **Risk Controls**: Position limits, leverage caps, circuit breakers
- **Account Management**: Balance monitoring, position tracking

**Current Implementation**:
- `python/` - gRPC protocol definitions
- `platform/` - Execution engine (in development)

#### 4. Infrastructure Module
**Purpose**: Platform orchestration, monitoring, and data persistence

**Key Features**:
- **Communication**: gRPC for inter-module communication
- **Database**: PostgreSQL with TimescaleDB for time-series data
- **Monitoring**: Logging, metrics, and performance tracking
- **Configuration**: TOML-based configuration management

**Current Implementation**:
- `postgres/` - Database migrations and schema
- `python/` - Shared utilities and gRPC stubs
- `web/` - Monitoring dashboard (basic)

## Data Strategy

### Primary Data Sources

1. **Coinbase**: Real-time WebSocket feeds for major crypto pairs
   - High-frequency tick data
   - Order book snapshots
   - Volume and liquidity metrics

2. **Major Exchanges**: Binance, Kraken, Gemini for cross-validation
   - Price discovery and arbitrage opportunities
   - Market depth analysis
   - Funding rate data

### Data Processing Pipeline

1. **Real-time Collection**: WebSocket streams with <50ms latency
2. **Aggregation**: OHLCV bars (1s, 1m, 5m intervals)
3. **Feature Engineering**: Technical indicators, market microstructure
4. **Storage**: TimescaleDB hypertables with automatic partitioning
5. **Distribution**: gRPC streaming to ML strategies

## ML Strategy Framework

### Current ML Models

#### DRQN (Deep Recurrent Q-Network)
- **Performance**: 500%+ outperformance over buy-and-hold
- **Features**: Action augmentation, LSTM-based sequential learning
- **Implementation**: Production-ready with CLI training/validation

#### DQN Variants
- **Double DQN**: Reduced overestimation bias
- **Rainbow DQN**: Multiple improvements combined
- **Noisy Networks**: Exploration without epsilon-greedy

### Strategy Development Workflow

1. **Data Preparation**: Historical data from reliable sources
2. **Feature Engineering**: Technical indicators, market features
3. **Model Training**: Backtesting with proper validation splits
4. **Strategy Validation**: Out-of-sample testing and cross-validation
5. **Live Deployment**: Real-time signal generation and execution

### Modular Strategy Components

- **Indicators**: EMA, RSI, MACD, custom ML features
- **Filters**: Volume thresholds, liquidity checks, market conditions
- **Risk Modules**: Position sizing, stop-loss, leverage management
- **Execution**: Order routing, slippage control, timing optimization

## MVP Implementation

### Phase 1: Data Foundation
- **Data Sources**: Coinbase WebSocket integration
- **Database**: TimescaleDB setup with proper schema
- **Storage**: Historical data collection and management

### Phase 2: ML Strategy Development
- **Model Training**: DRQN implementation with backtesting
- **Validation**: Out-of-sample testing and performance metrics
- **Signal Generation**: Real-time inference pipeline

### Phase 3: Execution Integration
- **DEX Integration**: Hyperliquid SDK integration
- **Order Management**: Risk controls and position management
- **Monitoring**: Real-time performance tracking

### Phase 4: Platform Orchestration
- **gRPC Communication**: Inter-module communication
- **Configuration**: TOML-based module configuration
- **Monitoring**: Logging, metrics, and alerting

## Technical Specifications

### Performance Requirements
- **Latency**: <100ms end-to-end for signal generation
- **Throughput**: High-frequency data streaming (1s intervals)
- **Scalability**: Horizontal scaling via multiple strategy instances
- **Reliability**: 99.9% uptime with automatic reconnection

### Database Schema
```sql
-- Market data hypertable
CREATE TABLE market_data (
    timestamp TIMESTAMPTZ,
    symbol VARCHAR(20),
    open NUMERIC(18,8),
    high NUMERIC(18,8),
    low NUMERIC(18,8),
    close NUMERIC(18,8),
    volume NUMERIC(18,8)
);

-- Strategy execution tracking
CREATE TABLE trades (
    id SERIAL PRIMARY KEY,
    strategy_id INT,
    symbol VARCHAR(20),
    side ENUM('buy','sell'),
    leverage INT,
    size NUMERIC(18,8),
    entry_price NUMERIC(18,8),
    exit_price NUMERIC(18,8),
    pnl NUMERIC(18,8),
    timestamp TIMESTAMPTZ
);
```

### gRPC Communication
- **Data Streaming**: Server-side streaming for real-time feeds
- **Signal Execution**: Unary RPC for discrete trading signals
- **Status Updates**: Bidirectional streaming for execution status

## Risk Management

### Position Limits
- **Maximum Exposure**: 20% of portfolio per strategy
- **Leverage Caps**: Configurable maximum leverage (up to 50x)
- **Stop Loss**: Mandatory stop-loss enforcement
- **Daily Loss Limits**: Circuit breakers for excessive losses

### Safety Features
- **Kill Switch**: Emergency halt functionality
- **Input Validation**: All gRPC calls validated
- **Audit Trails**: Complete logging of all trading actions
- **API Key Management**: Secure credential storage

## Development Roadmap

### Immediate Priorities (Weeks 1-4)
1. **Data Pipeline**: Complete Coinbase integration
2. **ML Models**: Enhance DRQN implementation
3. **Database**: TimescaleDB optimization
4. **Testing**: Comprehensive backtesting framework

### Short-term Goals (Weeks 5-8)
1. **Hyperliquid Integration**: DEX execution module
2. **Strategy Framework**: Modular component system
3. **Risk Management**: Position and exposure controls
4. **Monitoring**: Real-time performance tracking

### Long-term Vision (Weeks 9-12)
1. **Multi-DEX Support**: dYdX, GMX integration
2. **Advanced ML**: Ensemble methods, reinforcement learning
3. **Platform Orchestration**: Full gRPC communication
4. **Production Deployment**: Scalable, reliable trading system

## Project Structure

```
varon-fi/
├── .github/              # Design documents and project management
├── python/               # Shared Python package with gRPC protocol
├── postgres/             # Database migrations and TimescaleDB setup
├── coinbase/             # Coinbase WebSocket data collector
├── dydx/                 # dYdX API integration
├── poc/                  # DRQN proof-of-concept implementation
├── models/               # Enhanced ML models and training pipelines
├── trading-bot/          # Legacy trading bot implementations
├── platform/             # Execution engine and orchestration
└── web/                  # Monitoring dashboard
```

## Success Metrics

### Technical Performance
- **Latency**: <100ms end-to-end signal generation
- **Accuracy**: >80% signal accuracy in backtesting
- **Uptime**: 99.9% system availability
- **Scalability**: Support for 10+ concurrent strategies

### Trading Performance
- **Returns**: Outperformance over buy-and-hold baselines
- **Risk-Adjusted Returns**: Sharpe ratio >1.5
- **Drawdown**: Maximum drawdown <10%
- **Consistency**: Positive returns across market conditions

## Risk Disclaimer

This platform is designed for high-leverage trading which carries significant risk of loss. Users should:

- Understand the risks of high-leverage trading
- Start with small position sizes
- Test strategies thoroughly in backtesting
- Monitor positions continuously
- Have appropriate risk management in place

---

*This design document serves as the authoritative guide for varon.fi platform development and should be updated as the platform evolves.*
