# Local AI "TradingView" Assistant: Project Plan

*Prepared 2026-11-18. Technical planning notes, not financial advice. Prices, policies and model rankings change quickly, so re-verify before building.*

---

## 1. Goal

A local, privately run assistant that works like a "TradingView with a brain" for stocks and crypto:

- Ingests prices, news, social posts and SEC filings (including newly public companies).
- Estimates calibrated probabilities for well-defined events and notifies me when a probability is high enough (~70%+, to be tuned from backtests).
- Explains its reasoning in chat, with sources.
- Eventually places trades through a locked-down, rule-gated executor (paper trading first).

## 2. Hardware and what it implies

| Part | Spec | Notes |
|---|---|---|
| GPU | RTX 4070, 12 GB VRAM | Runs 7-14B LLMs at Q4; QLoRA fine-tuning of ~8B models is feasible; 30B+ is not practical |
| CPU | Ryzen 5 7600 (6c/12t) | Fine for pipelines and backtests; parallelize parameter sweeps |
| RAM | 32 GB DDR5 | Use Parquet + DuckDB/Polars lazily for big datasets |
| Storage | 2 TB NVMe | Store data as compressed Parquet; models take ~5-10 GB each |

## 3. Core design principle

**Do not train the LLM to "be" the trader.** Ollama serves models; it does not train them. LLMs also have no real edge at predicting prices.

- **LLM (via Ollama):** reads and explains: news, filings, chart descriptions, chat UI, tool calling. QLoRA fine-tune only for narrow tasks (sentiment tagging, filing extraction, strict JSON output).
- **Numeric models (LightGBM / logistic regression / time-series models):** produce the probabilities, calibrated and walk-forward validated.
- **Rules + risk gate (plain code):** the only thing that can approve an order.

```
Data sources → Ingestion (no LLM) → Parquet/DuckDB
                                        │
                  ┌─────────────────────┼──────────────────────┐
                  ▼                     ▼                      ▼
           LLM layer (Ollama)    Probability engine      Chart UI + chat
           sentiment, extraction  features → calibrated   (Lightweight Charts)
                  │               classifier
                  └───────┬──────────────┘
                          ▼
                 Signal (JSON proposal)
                          ▼
                 Risk gate (hard limits)  →  Alerts (Telegram/ntfy)
                          ▼
                 Executor (holds API key)  →  Broker / exchange (paper first)
```

## 4. Model recommendations (12 GB VRAM)

- **Default:** Qwen3 14B (Q4_K_M): strong tool calling and structured output, hybrid thinking mode.
- **Fast / high-volume jobs:** Qwen3 8B (fits in ~5-6 GB at Q4).
- **Multimodal alternative:** Gemma 4 12B (QAT Q4): handles chart images.
- **Reasoning-focused option:** Phi-4 14B (~9 GB at Q4).
- Run with Ollama (`ollama pull qwen3:14b`). Compare candidates on your own 200-item labeled eval set before committing; blog benchmarks vary in quality.

Fine-tuning path: train with Unsloth or Axolotl (QLoRA) → merge → export GGUF → load via an Ollama Modelfile. FinGPT's sentiment model was trained on a single RTX 3090 in ~17 hours, so this is realistic on a 4070 with an 8B model.

## 5. Data sources

### Live / ongoing
- **Prices:** yfinance / Stooq for hobby use; Polygon, Tiingo, Alpha Vantage if paid reliability is needed. Include delisted tickers to avoid survivorship bias.
- **Crypto:** Binance public data (data.binance.vision), Kraken, Coinbase APIs.
- **Filings:** SEC EDGAR free API (send a descriptive User-Agent, respect rate limits). S-1/F-1, 424B4, 10-K/10-Q, 8-K, Form 4, 13F/13D/G.
- **News:** RSS, NewsAPI, Finnhub; Reddit API (free tier for small personal use); StockTwits; Telegram/Discord channels.
- **X (Twitter):** read-only via the official pay-per-use API or a third-party data vendor. Browser scraping violates X's terms and is actively blocked. Cache results and set a spend cap. X is among the most expensive and noisiest sources, so add it only if backtests show it helps.

### Historical datasets for training / evaluation

**SEC filings**
- EDGAR-CORPUS: 10-K annual reports 1993-2020, unlabeled (Hugging Face `eloukas/edgar-corpus`; use EDGAR-CRAWLER for newer years).
- cleaned_sec_docs: 651,900 filings (10-K, 10-Q, 8-K, etc.) 2000-2026, parsed into sections; custom license (eddl-1.0), so read it first.
- SEC-EDGAR (datamule): 590 GB, ~8M filings, ~43B tokens. Use subsets only.
- financial-reports-sec: 10-Ks split into sections and sentences, useful for classification.

**Finance instruction tuning**
- FinGPT datasets/models (huggingface.co/FinGPT): FiQA, WealthAlpaca, sentiment LoRA recipes, benchmark.

**Crypto prices**
- Binance public data (spot + futures klines, trades).
- Kraken full historical OHLCVT up to Q3 2024.
- `rogerdehe/klines-binance` on Hugging Face (Parquet).
- Watch for survivorship bias: dead coins vanish from many free feeds.

**Sentiment**
- Bitcoin Tweets (16M, labels are noisy; check usage terms).
- DLT-Sentiment-News (23,301 crypto headlines, crowdsourced labels; cleaner for fine-tuning).
- Bitcoin Reddit Sentiment Dataset (posts + comments annotated with sentiment/emotion).
- LLM-Sentiment-Augmented Bitcoin Dataset (news + Reddit aligned with daily price moves).

## 6. Time-series models: stay skeptical

- **Kronos** (Tsinghua, AAAI 2026, MIT license): trained specifically on candlestick data; most promising open option to test.
- **Chronos-2, TimesFM 2.5:** general-purpose; zero-shot gains on equities are small. One 2026 study found TimesFM's best result was a 21-day MAPE of 2.87% vs 3.06% for the naive baseline.
- Use them as baselines or extra features. Always test against naive baselines with fees and slippage.

## 7. Probability engine rules

1. **Define events precisely** (e.g. triple-barrier: +3% before -2% within 5 trading days).
2. **Features:** price/volume indicators, volatility, volume spikes, market and sector context, plus LLM-derived features (sentiment score, event type, source reliability).
3. **Model:** LightGBM with logistic regression baseline.
4. **Calibrate** (isotonic or Platt) and check a reliability diagram: when the model says 70%, does it happen ~70% of the time?
5. **Validate walk-forward** (never shuffle time series; avoid look-ahead leakage; timestamp every data source at publication time).
6. **Include costs** and check expected value, not just hit rate.
7. **Choose the alert threshold from backtests**, not by fiat. Expect honest models to land at modest accuracy; a 70%+ alert should be rare. If it fires constantly, suspect leakage or overfitting.
8. **Log every alert and outcome** to monitor live calibration.

## 8. Newly public companies (IPO) module

- Pull S-1/F-1, 424B4, 10-Q/10-K, 8-K, Form 4 from EDGAR.
- LLM extracts structured fields (revenue growth, margins, cash burn/runway, debt, customer concentration, share structure, risk-factor red flags) into a strict JSON schema. Validate numbers against XBRL data instead of trusting LLM arithmetic. Chunk long filings to fit context.
- Track dated events: quiet-period end (~25 days), first earnings, lock-up expiry (typically 90-180 days), insider and early-investor selling.
- Pool many IPOs across years (including failed and delisted) for a cross-sectional model; per-company history is too short.
- Beware thin liquidity, wide spreads, and retail hype.

## 9. Roadmap

**Phase 0: Foundation (week 1)**
- Install Ollama, pull Qwen3 14B and an 8B model; set up Python, DuckDB, Parquet.
- Build a 200-item hand-labeled eval set (headlines + filing snippets).

**Phase 1: Data layer (weeks 2-3)**
- Bulk-load historical prices (stocks + crypto), schedule incremental updates.
- Add EDGAR ingestion and news/RSS/Reddit feeds; store publication timestamps.

**Phase 2: Probability engine (weeks 4-7)**
- Event definitions, features, LightGBM + calibration, walk-forward validation with costs.
- Test Kronos/Chronos as baselines or features.
- **Gate:** nothing advances unless it beats a naive baseline out of sample.

**Phase 3: LLM layer (weeks 6-9)**
- Prompt-engineer first; QLoRA fine-tune only where prompting falls short (sentiment, extraction, JSON output).
- Add retrieval over stored filings/news with citations.
- Add tools: `get_price`, `get_indicators`, `get_filing`, `get_probability`.

**Phase 4: "TradingView" front end (weeks 8-11)**
- Local web app (FastAPI + TradingView's open-source Lightweight Charts, or Streamlit to start).
- Chat panel using tools + retrieval; overlays for probabilities, news markers, earnings and lock-up dates.
- Optional vision: render charts to images for a multimodal model.

**Phase 5: Alerts and paper trading (weeks 10-14)**
- Telegram/ntfy alerts with probability, reasoning, sources, levels.
- Paper trade for at least 2-3 months.

**Phase 6: Hardening (ongoing)**
- Monitoring for data gaps and drift, scheduled retraining, prompt-injection filtering on all external text, executor isolation.

## 10. Securing the trade function

**Principle: the LLM never touches money.** It proposes; deterministic code decides and executes.

### Separation
- LLM process: no credentials, no network route to the broker, cannot import executor code. It only writes proposals to a queue/file.
- Risk gate: validates every proposal against fixed rules.
- Executor: small, separate process (own user/container/VM) holding the API key; places only validated orders.

### API key hygiene
- Trade + read permissions only; **never enable withdrawals/transfers** on a bot key.
- IP allowlist where supported; dedicated sub-account with only risk capital.
- Secrets in OS keyring / secrets manager / strict-permission `.env` (chmod 600); never in Git (add `.env` to `.gitignore`, run gitleaks), prompts, or logs.
- Rotate keys periodically and on any suspicion; enable 2FA.

### Risk gate rules (hard-coded)
- Max position size (e.g. 2-5% per trade) and max total exposure.
- Max daily/weekly loss, then halt until manual reset.
- Max orders per hour (catches runaway loops).
- Symbol allowlist; limit orders with max slippage; mandatory stop-loss.
- Sanity checks: reject stale data, prices far from market, unusually large sizes.
- No leverage/margin initially.

### Operations
- Kill switch (cancel open orders, stop executor); test it before you need it.
- Idempotent orders with unique client IDs; reconcile positions with the broker every cycle and halt on mismatch.
- Full audit log of proposals, gate decisions, orders, fills.
- Alerts on rejections, API errors, odd fills, kill-switch events.
- Human approval (Telegram approve/reject buttons) at first.
- Pin dependency versions; be wary of unvetted trading libraries; no open ports (use localhost or a VPN such as Tailscale).

### Rollout
1. Paper trading / testnet for several months.
2. Live with a tiny, affordable-to-lose size and manual approval.
3. Scale up only if live results match the backtest and calibration holds.

## 11. Getting a trading API

| Platform | Official API | Notes |
|---|---|---|
| Alpaca | Yes | API-first, free paper trading, SDKs in Python/JS/Go/C#; US-focused, check country eligibility |
| Interactive Brokers | Yes (TWS API, Web API, others) | Multi-asset, global; more complex setup |
| Tradier | Yes | Options-focused |
| Schwab | Yes | TD Ameritrade successor; OAuth/approval friction |
| Robinhood | No official public API for account access (as of 2026 comparisons) | Unofficial clients risk terms violations and lockouts; avoid |
| Coinbase / Kraken | Yes | REST + WebSocket; permission-scoped keys |
| Binance | Yes | Binance.com not available to US residents (Binance.US instead) |

Steps: open and verify an account → generate API keys or authorize via OAuth → start in the paper/testnet environment → create a separate trade-only key for live use after paper testing succeeds. Check market-data costs and the broker's terms on automated trading.

**Suggested start:** US → Alpaca (stocks) + Kraken or Coinbase (crypto). Outside the US → IBKR for stocks. *(Fill in: country = ____, starting asset class = ____.)*

## 12. Social media notes (X)

- A dedicated bot account is fine, but automated accounts must be labeled as bots in the profile; no impersonation, no bulk follow/like, no unsolicited automated replies.
- X's API is pay-per-use (legacy Free/Basic/Pro tiers discontinued or being phased out). Posts cost about $0.015, or $0.20 if they include a link; reads add up quickly, so cache and cap spend.
- Sharing X content with third parties is restricted (generally Post IDs/User IDs only).
- Treat all social/web text as untrusted input (prompt injection). The LLM outputs structured signals; it never executes actions based on instructions found in content.
- Prefer cheaper, more stable sources first (SEC EDGAR, Reddit, RSS, news APIs).

## 13. Open questions / next steps

- [ ] Country and broker eligibility (determines Alpaca vs IBKR, Binance vs Binance.US).
- [ ] Stocks first or crypto first? Holding period (intraday, swing, longer)?
- [ ] Build Phase 0: project skeleton, EDGAR downloader, eval-set template.
- [ ] Decide whether paid X reads are worth it after testing free sources.

## 14. Sources

- FinGPT: https://github.com/ai4finance-foundation/fingpt
- EDGAR-CORPUS: https://huggingface.co/datasets/eloukas/edgar-corpus and https://arxiv.org/pdf/2109.14394
- SEC-EDGAR (datamule): https://huggingface.co/datasets/kapilrao/SEC-EDGAR
- cleaned_sec_docs: https://huggingface.co/datasets/chiheemwong/cleaned_sec_docs
- financial-reports-sec: https://huggingface.co/datasets/JanosAudran/financial-reports-sec
- Binance public data guide: https://dev.to/pi19404/a-technical-guide-to-downloading-and-managing-binance-historical-crypto-market-data-d82
- Kraken historical data: https://concretumgroup.com/how-to-get-free-full-crypto-intraday-data-2013-2025-from-kraken/
- Binance klines (Hugging Face): https://huggingface.co/datasets/rogerdehe/klines-binance
- Bitcoin Tweets sentiment: https://huggingface.co/datasets/ckandemir/bitcoin_tweets_sentiment_kaggle
- DLT-Corpus / DLT-Sentiment-News: https://arxiv.org/html/2602.22045v1
- Bitcoin Reddit Sentiment Dataset: https://mx.nthu.edu.tw/~chungchichen/FinNLP2022_IJCAI/8.pdf
- LLM-Sentiment-Augmented Bitcoin Dataset: https://huggingface.co/datasets/danilocorsi/LLMs-Sentiment-Augmented-Bitcoin-Dataset
- Kronos overview: https://blog.pebblous.ai/report/kronos-financial-foundation-model/en/
- Classical vs foundation models for stock forecasting: https://theses.liacs.nl/pdf/2025-2026-KourntidisDDimitrios.pdf
- Local LLMs for 12 GB VRAM: https://atomic.chat/blog/guides/best-local-llm-12gb, https://bestllmfor.com/reviews/qwen3-14b-review/, https://fast.io/resources/best-local-ai-models-2026/, https://localaimaster.com/vram/best-llm-12gb-vram
- X API pricing and rules: https://www.xpoz.ai/blog/guides/understanding-twitter-api-pricing-tiers-and-alternatives/, https://opentweet.io/how-to/x-api-rules-for-ai-agents-2026, https://posteverywhere.ai/blog/x-twitter-api-pricing
- Broker APIs: https://aifinhub-monorepo.pages.dev/articles/choosing-a-broker-api-2026/, https://aifinhub.pages.dev/articles/broker-apis-compared-2026/, https://investingintheweb.com/brokers/best-api-brokers, https://walnutinvest.com/resources/best-brokers-for-ai-trading
