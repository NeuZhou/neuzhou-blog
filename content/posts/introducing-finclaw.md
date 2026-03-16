---
title: "FinClaw: AI-Powered Financial Intelligence for the Agentic Era"
date: 2026-03-16
tags: ["fintech", "ai", "trading", "open-source"]
categories: ["Announcements"]
summary: "FinClaw is an AI finance assistant covering US stocks, BIST, crypto, and forex — with 8 master strategies, portfolio tracking, technical analysis, and real-time market intelligence."
ShowToc: true
TocOpen: true
---

What if your AI agent could analyze markets, run technical indicators, track your portfolio, and brief you every morning — all without a single API key?

**FinClaw** does exactly that. It's an AI-powered finance assistant that turns your agent into a capable market analyst covering US stocks, Turkish equities (BIST), crypto, and forex.

## What Is FinClaw?

FinClaw is an [OpenClaw](https://github.com/nicepkg/openclaw) skill that gives AI agents deep financial capabilities:

- **Real-time quotes** for stocks, crypto, and forex
- **Candlestick and line charts** generated locally with matplotlib
- **Technical analysis** — SMA, EMA, RSI, MACD, Bollinger Bands with buy/sell signals
- **Portfolio tracking** with P&L calculation
- **Price alerts** with configurable conditions
- **Watchlists** for organized market monitoring
- **Morning/evening briefings** with market summaries
- **Macro economics dashboard** with Fed rate, CPI, unemployment data
- **Sentiment analysis** from news sources
- **Deep research** reports on individual securities

All computation runs locally. Core features work with zero API keys — just install and go.

## 3 Markets, One Interface

| Market | Symbols | Data Source |
|--------|---------|-------------|
| **US Stocks** | AAPL, MSFT, TSLA... | Finnhub + yfinance |
| **BIST (Turkey)** | THYAO.IS, GARAN.IS... | yfinance (.IS suffix) |
| **Crypto** | BTC, ETH, SOL... | Binance API |
| **Forex** | USD/TRY, EUR/USD... | ExchangeRate API |

Symbol detection is automatic. Type `AAPL` and FinClaw knows it's a US stock. Type `BTC` and it queries Binance. Type `THYAO.IS` and it pulls BIST data. No configuration, no mode switching.

## 8 Master Strategies

FinClaw doesn't just show you numbers — it helps you think about them. The research module combines 8 analytical strategies:

| # | Strategy | What It Does |
|---|----------|-------------|
| 1 | **Trend Following** | SMA/EMA crossovers, ADX strength, trend persistence |
| 2 | **Mean Reversion** | Bollinger Band extremes, RSI overbought/oversold, z-score |
| 3 | **Momentum** | Rate of change, relative strength, momentum divergences |
| 4 | **Volume Analysis** | Volume-price confirmation, OBV, accumulation/distribution |
| 5 | **Support/Resistance** | Key price levels, breakout detection, pivot points |
| 6 | **Volatility** | ATR analysis, volatility regime detection, squeeze setups |
| 7 | **Macro Context** | Interest rates, yield curve, sector rotation signals |
| 8 | **Sentiment** | News sentiment scoring, social media buzz, fear/greed indicators |

Each strategy produces independent signals. The research module synthesizes them into a coherent view — agreeing signals strengthen conviction, conflicting signals flag uncertainty.

## The Verified Alpha: Why Agent-Driven Analysis Works

Here's the insight that makes FinClaw different from a Bloomberg terminal or a TradingView chart:

**AI agents can chain analysis steps that humans typically do sequentially.**

When you ask FinClaw to research a stock, it doesn't just show you one chart. It:

1. Pulls the latest price and volume data
2. Runs all technical indicators simultaneously
3. Checks the macro environment (rates, inflation, sector performance)
4. Analyzes recent news sentiment
5. Cross-references with your existing portfolio exposure
6. Synthesizes everything into a narrative with actionable signals

A human analyst does this too — but it takes 30-60 minutes per security. FinClaw does it in seconds, and it does it consistently every time without fatigue or cognitive bias.

The agent doesn't have "conviction" or "gut feelings." It has data, indicators, and systematic analysis. That's not always better than human judgment — but it's always **reproducible**.

## Daily Workflow

Here's what a typical day with FinClaw looks like:

**Morning:**
```
You: "Morning briefing"
FinClaw: [Generates market summary — overnight movers, pre-market action,
          key events, portfolio impact, alert status]
```

**During the day:**
```
You: "What's happening with NVDA?"
FinClaw: [Real-time quote, today's movement, technical signals,
          recent news, your portfolio position if any]
```

**Research:**
```
You: "Deep research on MSFT"
FinClaw: [Full 8-strategy analysis, chart with indicators,
          news sentiment, macro context, risk assessment]
```

**End of day:**
```
You: "Close briefing"
FinClaw: [Day's P&L, notable movers, alert triggers,
          tomorrow's earnings calendar]
```

## Portfolio Tracking That Actually Works

FinClaw tracks your positions with full cost basis accounting:

```bash
# Add a position
"Add 10 shares of AAPL at $185"

# Record a sale
"Sell 5 AAPL at $195"

# Check P&L
"Show my portfolio P&L"
```

It calculates realized and unrealized gains, tracks your average cost basis across multiple buys, and factors in fees. No spreadsheet needed.

**Price alerts** trigger when conditions are met:
- Price above/below a target
- Percentage change threshold
- Volume spike detection

Set them conversationally and FinClaw checks them on its heartbeat cycle.

## Security Considerations: Why We Also Built ClawGuard

FinClaw handles financial data — portfolio positions, trading patterns, market research. This is sensitive information that should never leave your machine.

That's one of the reasons we built [ClawGuard](/posts/introducing-clawguard/). When FinClaw and ClawGuard run together:

- **PII sanitization** prevents account numbers or portfolio details from leaking into logs
- **Exfiltration detection** blocks any attempt to send financial data to unauthorized endpoints
- **Intent-action mismatch** catches if a compromised prompt tries to use FinClaw's market access for unintended purposes

Financial tools demand financial-grade security. We take that seriously.

## Data Sources — No Vendor Lock-in

| Source | What It Provides | API Key |
|--------|-----------------|---------|
| **yfinance** | US + BIST quotes, charts, fundamentals | Not needed |
| **Binance** | Crypto prices, order books | Not needed |
| **ExchangeRate API** | Forex rates, conversion | Not needed |
| **Finnhub** | News, earnings calendar | Optional |
| **FRED** | Macro economics data | Optional |
| **Alpha Vantage** | Sentiment analysis | Optional |

Core functionality — quotes, charts, technical analysis, portfolio, alerts — works with **zero API keys**. Premium data sources (news, macro, sentiment) are additive.

## Get Started

FinClaw installs as an OpenClaw skill:

```bash
# Install OpenClaw if you haven't
npm install -g openclaw

# FinClaw is available in the skill marketplace
# Or install from the workspace
python3 scripts/setup.py
```

The setup script creates a Python virtual environment and installs all dependencies (yfinance, matplotlib, mplfinance, pandas, numpy).

## What's Next

We're actively developing:

- **Backtesting engine** — test strategies against historical data
- **Multi-portfolio support** — separate personal and research portfolios
- **Options chain analysis** — IV, Greeks, unusual activity detection
- **Automated reporting** — weekly PDF reports delivered to your inbox
- **BIST depth** — order book data for Turkish equities

## Try It

FinClaw is open source and free. If you're an investor, trader, or just someone who likes to stay informed about markets, give it a spin.

⭐ **Star the repo**: [github.com/nicepkg/openclaw](https://github.com/nicepkg/openclaw)

📈 **Get started**: Install OpenClaw, add the FinClaw skill, and ask for a morning briefing.

💬 **Join the community**: Share your strategies, request features, contribute improvements.

---

*FinClaw is part of the [OpenClaw](https://github.com/nicepkg/openclaw) ecosystem — because the best financial advisor is one that never sleeps, never panics, and never charges 2-and-20.*
