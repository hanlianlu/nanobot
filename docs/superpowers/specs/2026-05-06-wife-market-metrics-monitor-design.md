# Wife Market Metrics Monitor Design

**Date:** 2026-05-06
**Status:** Draft for user review
**Scope:** wife nanobot instance only; independent US equity metrics monitor

## 1. Purpose

Create a wife-focused nanobot skill that tracks user-selected US stocks and watchlists as a daily metrics archive. The system is an observation and explanation layer: it maintains price history, computes its own metrics, detects metric evolution and anomalies, and generates integrated visual cards for WeChat and Telegram.

This is not the existing finance skill and not the finvesto professional investment pipeline. It must not produce portfolio optimization, model predictions, execution guidance, or trading recommendations unless the user explicitly switches to the existing finance workflow.

## 2. Skill Placement and Discovery

The skill will live in the wife workspace:

```text
/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/
  SKILL.md
  scripts/
  references/
  assets/
```

Nanobot discovers workspace skills from `<workspace>/skills/{name}/SKILL.md` and builtin repo skills from `nanobot/skills/{name}/SKILL.md`. Workspace skills take precedence over builtin skills with the same name. The wife config already points `agents.defaults.workspace` to `~/.nanobot-wife/workspace`, so this location is discoverable after restarting `com.nanobot-wife.gateway`.

## 3. Boundaries

The new skill owns its trigger language, data, metrics, rendering, and update schedule.

It may:

- Read its own monitor data under the wife workspace.
- Read finvesto credentials or raw market cache as an optional, read-only source.
- Use live/EOD market providers through its own provider adapters.
- Send integrated PNG cards, optional HTML files, and concise chat summaries.

It must not:

- Trigger `invest_portfolio`, optimizer, alpha model, executor, professional report, or finvesto trading pipeline.
- Write into the finvesto repo or finvesto data directories.
- Present its metrics as finvesto model output.
- Give buy/sell or rebalance advice.

Routing rule:

- Use `market-metrics-monitor` for: "monitor", "指标趋势", "trendline", "trend table", "异常", "关键价位", "RSI/MACD演化", "这个组合最近有没有异常".
- Use existing `finance` only for: buy/sell, 调仓, portfolio optimization, model prediction, 操盘建议, investment report.
- For ambiguous prompts like "TSLA 怎么样", ask whether the user wants indicator monitoring or investment/portfolio advice.

## 4. Persistence Layout

The monitor owns this root:

```text
/Users/hanlianlyu/.nanobot-wife/workspace/data/market-metrics-monitor/
  registry/
    tickers.json
    watchlists.json
    providers.json
  us/
    prices/{TICKER}.parquet
    metrics/{TICKER}.parquet
    anomalies/{TICKER}.jsonl
    snapshots/{TICKER}.latest.json
    snapshots/{WATCHLIST}.latest.json
    cards/
      {TICKER}_monitor_{YYYY-MM-DD}.html
      {TICKER}_monitor_{YYYY-MM-DD}.png
      {WATCHLIST}_monitor_{YYYY-MM-DD}.html
      {WATCHLIST}_monitor_{YYYY-MM-DD}.png
  runs/
    update_runs.jsonl
    data_quality.jsonl
```

`registry/tickers.json` stores the monitored universe:

```json
{
  "AAPL": {
    "symbol": "AAPL",
    "market": "us",
    "name": "Apple Inc.",
    "active": true,
    "added_at": "2026-05-06T21:00:00+02:00",
    "source": "user"
  }
}
```

`registry/watchlists.json` stores user groupings:

```json
{
  "wife_core": {
    "market": "us",
    "symbols": ["AAPL", "MSFT", "NVDA"],
    "active": true,
    "created_at": "2026-05-06T21:00:00+02:00"
  }
}
```

`registry/providers.json` stores provider priority and credential references, not plaintext secrets:

```json
{
  "priority": ["alpaca", "stooq", "yfinance", "finvesto_raw_cache"],
  "credential_refs": {
    "alpaca": {
      "env_file": "/Users/hanlianlyu/Github/finvesto/.env",
      "read_only": true
    }
  }
}
```

## 5. Stored Tables

`prices/{TICKER}.parquet` stores raw daily market facts:

```text
date
symbol
open
high
low
close
adjusted_close
volume
currency
provider
provider_symbol
fetched_at
quality_flags
```

`metrics/{TICKER}.parquet` stores one computed row per trading day:

```text
date
symbol
close
return_1d
return_5d
return_20d
ma_5
ma_20
ma_50
ma_200
ma_stack_state
rsi_14
macd
macd_signal
macd_hist
volatility_20d
volume_ratio_20d
drawdown_from_52w_high
price_action_state
trend_slope_20d
trend_slope_60d
relative_strength_spy_20d
relative_strength_spy_60d
anomaly_score
quality_flags
computed_at
```

`anomalies/{TICKER}.jsonl` stores event-level records:

```json
{"date":"2026-05-06","symbol":"AAPL","type":"volume_spike","severity":"medium","metric":"volume_ratio_20d","value":2.4,"explanation":"成交量达到20日均量的2.4倍，说明市场关注突然升高。"}
```

`snapshots/*.latest.json` stores chat-optimized cache:

```json
{
  "symbol": "AAPL",
  "as_of": "2026-05-06",
  "state": "uptrend_pullback",
  "headline": "中期趋势仍向上，但短线动能回落",
  "key_metrics": [],
  "trendlines": [],
  "recent_anomalies": [],
  "metric_explanations": {},
  "quality_flags": []
}
```

All writes use a temp file followed by atomic rename. Upserts are keyed by `date + symbol`. A failed ticker update never blocks other tickers.

## 6. History Windows

When a ticker is first added, the system backfills at least two years of daily OHLCV. The monitor computes metrics for all available history, not just a short window.

Some metrics require warmup:

- MA200 needs about 200 trading days.
- 52-week drawdown needs about 252 trading days.
- Percentile and anomaly baselines improve with longer history.

Rows with insufficient warmup are retained and marked with `quality_flags`, such as `insufficient_history`. Chat defaults to 90D, 180D, or 260D views, but the stored metrics are not clipped to 260 trading days.

## 7. Metrics and Explanations

The monitor combines Livermore-style price-action state with modern indicators.

Price-action states:

- `natural_rally` / 自然回升
- `natural_reaction` / 自然回调
- `uptrend` / 上涨趋势
- `downtrend` / 下跌趋势
- `secondary_rally` / 次级回升
- `secondary_reaction` / 次级回调
- `range_bound` / 区间震荡
- `uncertain` / 状态不明确

Modern metrics:

- MA5/20/50/200: trend structure and support/resistance context.
- RSI-14: short-term overbought/oversold pressure; above 70 is hot, below 30 is cold.
- MACD: momentum acceleration and deceleration.
- 20D volatility: recent price swing intensity.
- Volume ratio 20D: whether participation is unusually high or low.
- Drawdown from 52W high: distance from recent long-term peak.
- Relative strength vs SPY: whether the ticker is outperforming the broad US market.

Every card or table must include explanations for the metrics it displays. The skill must prefer plain Chinese explanations for wife chat unless the user asks in English.

## 8. Trend and Anomaly Engine

The core value is metric evolution, not only current values. Each update computes:

- Current value.
- 20D and 60D trend direction.
- Trend slope and slope change.
- Crossovers and state transitions.
- Distance to key levels.
- Recent anomaly count and severity.

Anomaly types:

- `trend_shift`: price-action state changed.
- `volume_spike`: volume materially above baseline.
- `volatility_spike`: volatility materially above baseline.
- `momentum_divergence`: price and momentum disagree.
- `ma_cross`: important moving-average crossover.
- `rsi_extreme`: RSI entered or exited an extreme zone.
- `gap_move`: open gap or large one-day move.
- `stale_data`: expected update missing.
- `provider_gap`: provider has incomplete or suspicious rows.

Severity is `low`, `medium`, or `high`. Explanations must describe the metric, the observed change, and why it matters, without converting it into investment advice.

## 9. Data Sources

The monitor is watchlist-first and does not assume finvesto universe coverage. It supports any US tradable ticker that the configured providers can resolve, including symbols outside finvesto's local universe.

Provider order:

1. Monitor local cache.
2. EOD/live provider using credential references, such as Alpaca credentials read from finvesto `.env`.
3. Independent fallback providers, such as Stooq or yfinance-style adapters.
4. finvesto raw cache, read-only, only when it covers the requested ticker.

The monitor records the provider used per row. If providers disagree or a ticker cannot be resolved, it writes a data-quality event and surfaces the issue in the card.

## 10. Daily Update Mechanism

Use `launchd` for the primary update path, not nanobot cron.

Rationale:

- The update is deterministic data engineering work, not an LLM task.
- It should keep running even if nanobot gateway, model, or channel delivery is unavailable.
- `launchd` gives clearer process logs, exit status, and retry scheduling.
- macOS M4 Max has enough local capacity for a watchlist-first daily job.

Script:

```text
/Users/hanlianlyu/.nanobot-wife/workspace/skills/market-metrics-monitor/scripts/update_metrics.py
```

Schedule:

- Primary run after US market close, around 23:30 Stockholm time.
- Retry windows around 02:30 and 05:30 Stockholm time to handle provider delays and machine sleep.

Each run:

1. Read active tickers from `registry/watchlists.json` and `registry/tickers.json`.
2. Backfill new tickers with at least two years of daily OHLCV.
3. Re-fetch the latest ten trading days for existing tickers to catch corrections, splits, and provider restatements.
4. Upsert prices.
5. Recompute metrics over all available history.
6. Append anomaly events.
7. Write ticker and watchlist latest snapshots.
8. Generate integrated HTML and PNG cards only for requested/on-demand output, or optionally for active watchlists after EOD if configured.
9. Append `runs/update_runs.jsonl` and `runs/data_quality.jsonl`.

Nanobot heartbeat or cron can be used for notification only, such as "daily update failed" or "data is stale", not for the primary update.

## 11. Chat Workflows

Add or update monitoring:

```text
User: 监控 AAPL, MSFT, NVDA，叫 wife_core
Bot: creates/updates watchlist, backfills data, reports whether each ticker is ready.
```

Single ticker check:

```text
User: 看看 AAPL 最近指标趋势
Bot: reads latest snapshot, renders one integrated PNG card, sends 3-5 line summary.
```

Watchlist check:

```text
User: wife_core 最近有没有异常
Bot: renders one watchlist monitor PNG with row-level trends, top anomalies, and quality notes.
```

Ambiguous query:

```text
User: TSLA 怎么样
Bot: asks whether the user wants indicator monitoring or investment/portfolio advice.
```

The default chat answer is short. The card carries the dense information.

## 12. Integrated Rendering

Default output is a single integrated PNG card because WeChat and Telegram do not support native embedded HTML5 cards in the current nanobot channel implementation.

Rendering pipeline:

```text
snapshot + metrics history
  -> HTML5 card source
  -> browser render / screenshot
  -> PNG attachment
  -> optional HTML file attachment
```

Single ticker card:

- Header: symbol, company name, as-of date, source, quality flags.
- Price panel: close price with MA5/20/50/200.
- State panel: price-action state, trend summary, key levels.
- Metrics panel: RSI, MACD, volatility, volume ratio, drawdown, relative strength.
- Anomaly timeline: recent events by severity.
- Explanation panel: concise definitions for displayed metrics.

Watchlist card:

- Header: watchlist, as-of date, coverage count, failed/stale count.
- Main table: one ticker per row with state, 20D trend, 60D trend, RSI, volatility, volume, anomaly count.
- Sparklines: compact 60D price trend per ticker.
- Top anomalies: highest-severity recent events.
- Quality notes: stale or partial data.

Optional artifacts:

- `.html`: interactive local/static version, attached as a file if requested.
- `.pdf`: archival report if requested.
- `.csv`: raw metrics export if requested.

If a future web-hosting layer is added, chat can send a PNG preview plus an authenticated URL to the interactive HTML card. That is a separate integration and not part of the first implementation.

## 13. Channel Reality

Current nanobot channel support:

- WeChat supports text, image, voice, file, and video. `.html` can be sent as a file, not embedded as an interactive card.
- Telegram supports photos/documents and a limited HTML parse mode for message text. That HTML parse mode is not HTML5. Telegram inline keyboard buttons are available only when enabled in config.

Therefore the stable first-version UX is:

1. One PNG monitor card.
2. One concise text summary.
3. Optional HTML file attachment for users who want interactive local viewing.

## 14. Error Handling

Data failures are visible:

- If today's update failed but yesterday's snapshot exists, serve yesterday's card with `STALE`.
- If a provider cannot resolve a ticker, mark that ticker as failed and keep the rest of the watchlist usable.
- If volume is missing, compute price-only metrics and flag `missing_volume`.
- If split or adjustment looks suspicious, flag `split_suspected` and avoid overconfident anomaly explanations.
- If no history exists yet, tell the user that backfill must run before trend analysis is meaningful.

The skill must not fabricate successful updates, provider coverage, or metric values.

## 15. Testing and Verification

Implementation verification should cover:

- Skill discovery under wife workspace.
- Registry add/remove/list operations.
- New ticker two-year backfill.
- Existing ticker latest-ten-days refresh and upsert.
- Parquet schema compatibility.
- Metrics warmup flags for short histories.
- Anomaly detection fixtures for volume spike, MA cross, stale data, and trend shift.
- Integrated card rendering to PNG.
- WeChat/Telegram delivery path using PNG plus concise text.
- Isolation checks proving the new monitor does not call finvesto optimizer, alpha prediction, executor, or report tools.

## 16. Implementation Assumptions

- First market is US only.
- First deployment is wife instance only.
- The primary scheduler is `launchd`.
- The monitor owns its own metrics definitions.
- finvesto may be used only for read-only raw data or credential references.
- The default chat artifact is one integrated PNG card.
