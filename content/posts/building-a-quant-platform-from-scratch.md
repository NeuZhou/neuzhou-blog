---
title: "Building a Quantitative Finance Platform from Scratch with Python"
date: 2025-03-16T20:00:00+08:00
draft: false
tags: ["python", "quantitative-finance", "backtesting", "machine-learning", "fintech", "open-source"]
categories: ["Engineering", "Finance"]
description: "How we built FinClaw — a zero-dependency quantitative finance platform in Python with backtesting, risk analytics, ML integration, and options pricing. Architecture decisions, lessons learned, and code examples."
keywords: ["quantitative finance python", "backtesting framework", "risk analytics", "options pricing python", "numpy technical analysis"]
slug: "building-a-quant-platform-from-scratch"
---

Most quantitative finance platforms fall into two camps: bloated enterprise monsters that need a dedicated ops team, or toy scripts that break the moment you feed them real market data. We wanted neither. So we built **FinClaw** — a lightweight, zero-heavy-dependency Python platform for quantitative finance that actually works.

This post covers why we built it, the architecture decisions that shaped it, key features, and the hard lessons we learned along the way.

## Why Build Another Quant Platform?

The short answer: everything else was either too heavy or too fragile.

We evaluated the usual suspects — Zipline, Backtrader, QuantLib bindings, various commercial platforms. They all had the same problem: **dependency hell**. Installing QuantLib on a fresh machine is a multi-hour adventure. Zipline pins you to ancient pandas versions. Commercial platforms lock you into their ecosystems.

We had specific requirements:

1. **Portable** — should work on any machine with Python 3.10+ and numpy
2. **Fast enough** — backtesting thousands of strategies shouldn't require a cluster
3. **Transparent** — no magic. Every calculation should be inspectable
4. **Extensible** — easy to plug in new data sources, indicators, or models
5. **Production-ready** — not just a research toy, but something you can run daily

FinClaw was born from these constraints.

## Architecture: The Zero Heavy Deps Decision

The most controversial decision was **numpy-only technical analysis**. No TA-Lib, no pandas (for core computations), no scipy for basic stats. Just numpy.

Why? Three reasons:

### 1. Installation Friction Kills Adoption

TA-Lib requires compiling C libraries. On Windows, this means downloading pre-built wheels from unofficial sources. On ARM Macs, it's a coin flip. We wanted `pip install finclaw` to just work, everywhere.

### 2. Numpy Is Enough

Most technical indicators are moving averages, standard deviations, and simple arithmetic on arrays. You don't need a 200MB library for that:

```python
import numpy as np

def sma(prices: np.ndarray, period: int) -> np.ndarray:
    """Simple Moving Average — pure numpy, no dependencies."""
    if len(prices) < period:
        return np.full_like(prices, np.nan)
    kernel = np.ones(period) / period
    result = np.full_like(prices, np.nan, dtype=np.float64)
    result[period - 1:] = np.convolve(prices, kernel, mode='valid')
    return result

def ema(prices: np.ndarray, period: int) -> np.ndarray:
    """Exponential Moving Average — vectorized numpy."""
    alpha = 2.0 / (period + 1)
    result = np.empty_like(prices, dtype=np.float64)
    result[0] = prices[0]
    for i in range(1, len(prices)):
        result[i] = alpha * prices[i] + (1 - alpha) * result[i - 1]
    return result

def bollinger_bands(prices: np.ndarray, period: int = 20, num_std: float = 2.0):
    """Bollinger Bands from scratch."""
    middle = sma(prices, period)
    rolling_std = np.array([
        np.std(prices[max(0, i - period + 1):i + 1])
        if i >= period - 1 else np.nan
        for i in range(len(prices))
    ])
    upper = middle + num_std * rolling_std
    lower = middle - num_std * rolling_std
    return upper, middle, lower
```

Yes, the EMA loop isn't fully vectorized. In practice, it processes 10 years of daily data in under a millisecond. Premature optimization is the root of all evil — and in quant finance, the bottleneck is never the indicator calculation.

### 3. Transparency

When something goes wrong in a backtest (and it will), you need to trace every number. With numpy-only code, you can step through any calculation in a debugger. With TA-Lib, you're staring at a C library hoping the documentation is accurate.

## The Module Architecture

FinClaw is organized into focused modules:

```
finclaw/
├── data/           # Market data fetching and caching
├── indicators/     # Technical analysis (numpy-only)
├── backtest/       # Strategy backtesting engine
├── risk/           # Risk analytics and portfolio metrics
├── ml/             # Machine learning integration
├── options/        # Options pricing models
├── portfolio/      # Portfolio tracking and alerts
└── utils/          # Shared utilities
```

Each module is independently importable. Want just the indicators? `from finclaw.indicators import rsi, macd`. Need backtesting? Import the backtest engine. No God objects, no mandatory initialization ceremony.

## Key Features

### Backtesting Engine

The backtesting engine uses an event-driven architecture with a simple but powerful API:

```python
from finclaw.backtest import Strategy, Backtest

class MeanReversionStrategy(Strategy):
    def __init__(self, lookback: int = 20, entry_z: float = -2.0, exit_z: float = 0.0):
        self.lookback = lookback
        self.entry_z = entry_z
        self.exit_z = exit_z

    def on_bar(self, context):
        prices = context.history('close', self.lookback)
        if len(prices) < self.lookback:
            return

        z_score = (prices[-1] - np.mean(prices)) / np.std(prices)

        if z_score <= self.entry_z and not context.position:
            context.buy(size=context.portfolio_value * 0.1 / prices[-1])
        elif z_score >= self.exit_z and context.position:
            context.sell(size=context.position.size)

# Run backtest
bt = Backtest(
    strategy=MeanReversionStrategy(),
    data=market_data,
    initial_capital=100_000,
    commission=0.001,
    slippage=0.0005
)
results = bt.run()
print(results.summary())
```

Key design decisions:
- **Commission and slippage are mandatory parameters** — no "frictionless" backtests that look amazing but fall apart in production
- **No look-ahead bias by construction** — the `context.history()` method only returns data up to the current bar
- **Position sizing is explicit** — no magic "invest 100%" defaults

### Risk Analytics

Risk management isn't an afterthought — it's a core module:

```python
from finclaw.risk import RiskAnalyzer

analyzer = RiskAnalyzer(returns=portfolio_returns)

metrics = analyzer.full_report()
# Returns: Sharpe, Sortino, Max Drawdown, VaR (95/99),
# CVaR, Calmar ratio, rolling beta, tail risk metrics
```

We compute Value at Risk using three methods — historical, parametric, and Monte Carlo — and report all three. If they disagree significantly, that's a signal your return distribution has fat tails (it always does).

The drawdown analysis deserves special mention:

```python
def max_drawdown_analysis(equity_curve: np.ndarray) -> dict:
    """Compute drawdown metrics with recovery analysis."""
    peak = np.maximum.accumulate(equity_curve)
    drawdown = (equity_curve - peak) / peak

    max_dd = np.min(drawdown)
    max_dd_end = np.argmin(drawdown)
    max_dd_start = np.argmax(equity_curve[:max_dd_end + 1])

    # Recovery time
    recovery = np.where(equity_curve[max_dd_end:] >= equity_curve[max_dd_start])[0]
    recovery_time = recovery[0] if len(recovery) > 0 else None

    return {
        'max_drawdown': max_dd,
        'drawdown_start': max_dd_start,
        'drawdown_end': max_dd_end,
        'recovery_bars': recovery_time,
        'underwater_periods': np.sum(drawdown < -0.05),  # days below -5%
    }
```

### Machine Learning Integration

The ML module provides a clean interface for integrating predictive models without pulling in sklearn as a hard dependency:

```python
from finclaw.ml import FeatureEngine, WalkForwardValidator

# Feature engineering with built-in lag safety
engine = FeatureEngine()
engine.add_feature('momentum_20', lambda df: df['close'].pct_change(20))
engine.add_feature('vol_ratio', lambda df: df['volume'] / df['volume'].rolling(50).mean())
engine.add_feature('rsi_14', lambda df: rsi(df['close'].values, 14))

features = engine.build(data, min_lag=1)  # Enforces 1-bar lag to prevent leakage

# Walk-forward validation (not random split!)
validator = WalkForwardValidator(
    train_size=252,    # 1 year training
    test_size=63,      # 1 quarter testing
    step_size=21,      # Re-train monthly
)

for train_idx, test_idx in validator.split(features):
    # Train your model (sklearn, xgboost, whatever you prefer)
    model.fit(features[train_idx], labels[train_idx])
    predictions = model.predict(features[test_idx])
```

The `min_lag` parameter in `FeatureEngine.build()` is critical — it automatically shifts all features to prevent look-ahead bias. This single parameter prevents the most common ML-in-finance mistake.

### Options Pricing

The options module implements Black-Scholes, binomial trees, and Monte Carlo pricing:

```python
from finclaw.options import black_scholes, implied_volatility, greeks

# Price a European call
price = black_scholes(
    S=150.0,      # Current price
    K=155.0,      # Strike
    T=0.25,       # Time to expiry (years)
    r=0.05,       # Risk-free rate
    sigma=0.20,   # Volatility
    option_type='call'
)

# Compute all Greeks
g = greeks(S=150.0, K=155.0, T=0.25, r=0.05, sigma=0.20)
# g.delta, g.gamma, g.theta, g.vega, g.rho

# Implied volatility via Newton-Raphson
iv = implied_volatility(
    market_price=3.50, S=150.0, K=155.0, T=0.25, r=0.05
)
```

All implemented in pure numpy. The Monte Carlo pricer uses vectorized path generation — 100,000 paths for an Asian option in about 200ms.

## Lessons Learned the Hard Way

### Overfitting Detection

The biggest trap in quantitative finance is overfitting. A strategy that returns 500% in backtesting and loses money in production is worse than useless — it's expensive.

We built several overfitting guards into the platform:

1. **Out-of-sample testing is mandatory** — the backtest engine reserves the last 20% of data by default and warns if in-sample and out-of-sample Sharpe ratios diverge by more than 50%
2. **Parameter sensitivity analysis** — automatically perturb strategy parameters ±20% and check if performance degrades gracefully or falls off a cliff
3. **Trade count sanity checks** — a strategy that generates only 5 trades in 10 years of data is not statistically significant, no matter what the returns say

### Survivorship Bias

Early on, we tested a momentum strategy that looked incredible — until we realized our dataset only contained stocks that still existed. Dead companies (delisted, bankrupt) weren't in the data, which meant our momentum strategy was picking winners from a pre-filtered list.

The fix: FinClaw's data module explicitly supports point-in-time datasets and warns when survivorship bias might be present:

```python
from finclaw.data import DataLoader

loader = DataLoader(survivorship_free=True)
# Includes delisted securities, marks their removal date
# Warns if your universe changes size by >10% during backtest period
```

### Slippage Matters More Than You Think

A strategy that trades frequently with tight spreads can look profitable at zero slippage and hemorrhage money at 5bps. We learned to always model slippage as a function of volume:

```python
# Volume-dependent slippage model
def realistic_slippage(order_size, avg_volume, spread_bps=5):
    """Slippage increases non-linearly with order size relative to volume."""
    participation_rate = order_size / avg_volume
    base_slippage = spread_bps / 10_000
    impact = base_slippage * (1 + 10 * participation_rate ** 2)
    return impact
```

## Performance: Is Numpy Enough?

For daily data — absolutely. Backtesting 10 years of daily data for a single strategy takes under 1 second. Running a parameter sweep over 1,000 combinations takes about 15 minutes with multiprocessing.

For tick data or HFT simulations, you'd want to drop to Rust or C++. But FinClaw isn't targeting HFT — it's for strategies operating on daily or hourly timeframes, which covers 95% of systematic trading use cases.

## What's Next

FinClaw is open-source and actively developed. The roadmap includes:

- **Live trading connectors** for major brokers (Interactive Brokers, Alpaca)
- **Multi-asset support** — currently equities-focused, adding futures and crypto
- **Regime detection** — automatic identification of market regimes for adaptive strategies
- **Portfolio optimization** — mean-variance, risk parity, and Black-Litterman models

If you're building quantitative strategies in Python and tired of dependency hell, give FinClaw a try. The entire platform installs in seconds and the only hard dependency is numpy.

---

*FinClaw is part of the OpenClaw ecosystem. Check out the [GitHub repository](https://github.com/openclawai/finclaw) for documentation and examples.*
