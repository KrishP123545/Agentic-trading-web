# Agentic Trading Web

A local, privately run "TradingView with a brain" for stocks and crypto. It ingests prices, news, social posts and SEC filings, estimates calibrated probabilities for well-defined market events, explains its reasoning in chat with sources, and eventually places trades through a locked-down, rule-gated executor (paper trading first).

> **Status:** planning. No code yet. The full plan is in [local-ai-trading-assistant-roadmap.md](local-ai-trading-assistant-roadmap.md).
>
> Not financial advice. This is a personal research project.

## How it works

The LLM is not the trader. Each job goes to the tool that does it best:

| Layer | Tooling | Job |
|---|---|---|
| LLM | Ollama (Qwen3 14B / 8B) | Read and explain news and filings, run the chat UI, call tools. Fine-tuned only for narrow tasks such as sentiment tagging and filing extraction. |
| Probability engine | LightGBM + logistic regression baseline | Produce calibrated, walk-forward-validated event probabilities |
| Risk gate | Plain code | The only thing that can approve an order |
| Executor | Separate isolated process | Holds the broker API key and places validated orders only |

```
Data sources → Ingestion → Parquet/DuckDB
                               │
         ┌─────────────────────┼──────────────────────┐
         ▼                     ▼                      ▼
   LLM layer (Ollama)    Probability engine     Chart UI + chat
         └───────┬─────────────┘
                 ▼
        Signal (JSON proposal) → Risk gate → Alerts (Telegram/ntfy)
                                     ▼
                                 Executor → Broker / exchange (paper first)
```

## Planned stack

- **Models:** Ollama serving Qwen3 14B (default), Qwen3 8B (high-volume jobs), Gemma 4 12B (chart images); QLoRA fine-tuning with Unsloth or Axolotl
- **Data:** Python, DuckDB, Polars, Parquet
- **Sources:** yfinance/Stooq, Binance/Kraken/Coinbase, SEC EDGAR, RSS, Reddit, news APIs
- **Front end:** FastAPI + TradingView Lightweight Charts (Streamlit to start)
- **Brokers:** Alpaca (stocks) and Kraken or Coinbase (crypto), paper/testnet first

Target hardware: RTX 4070 (12 GB VRAM), Ryzen 5 7600, 32 GB RAM, 2 TB NVMe.

## Roadmap

| Phase | Weeks | Focus |
|---|---|---|
| 0 | 1 | Foundation: Ollama, Python, DuckDB; a 200-item hand-labeled eval set |
| 1 | 2-3 | Data layer: historical prices, EDGAR, news/RSS/Reddit ingestion |
| 2 | 4-7 | Probability engine: event definitions, features, calibration, walk-forward validation with costs |
| 3 | 6-9 | LLM layer: prompting, selective QLoRA, retrieval with citations, tool calls |
| 4 | 8-11 | Front end: charts, chat panel, probability and event overlays |
| 5 | 10-14 | Alerts and at least 2-3 months of paper trading |
| 6 | ongoing | Hardening: drift monitoring, retraining, prompt-injection filtering, executor isolation |

Gate for every phase: nothing advances unless it beats a naive baseline out of sample.

## Security rules

- The LLM never touches money or credentials. It writes proposals and nothing else.
- Bot API keys are trade-only, **never** withdrawal-enabled, and IP-allowlisted where supported.
- Secrets live in the OS keyring or a `chmod 600` `.env`, never in Git, prompts or logs.
- Hard limits on position size, exposure, daily loss, order rate and symbols. A tested kill switch.
- All external text is untrusted input.
