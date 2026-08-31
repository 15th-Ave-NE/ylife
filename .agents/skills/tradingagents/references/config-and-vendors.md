# Config keys, env-var overrides, and data-vendor chains — full detail

Source of truth for all of this is `tradingagents/default_config.py`. Read
that file directly if something here looks like it might have drifted —
this reference is a snapshot, not a live mirror.

## `DEFAULT_CONFIG` keys

| Key | Default | Notes |
|---|---|---|
| `project_dir` | repo root | Computed, not overridable via env. |
| `results_dir` | `~/.tradingagents/logs` | Env: `TRADINGAGENTS_RESULTS_DIR` (plain `getenv`, not in `_ENV_OVERRIDES`). |
| `data_cache_dir` | `~/.tradingagents/cache` | Env: `TRADINGAGENTS_CACHE_DIR`. |
| `memory_log_path` | `~/.tradingagents/memory/trading_memory.md` | Env: `TRADINGAGENTS_MEMORY_LOG_PATH`. |
| `memory_log_max_entries` | `None` | Caps *resolved* memory-log entries; oldest pruned past the cap. Pending entries are never pruned. `None` disables rotation. |
| `llm_provider` | `"openai"` | |
| `deep_think_llm` | `"gpt-5.5"` | |
| `quick_think_llm` | `"gpt-5.4-mini"` | |
| `backend_url` | `None` | `None` = each provider's own default endpoint. Deliberately not left pointing at one provider's URL — a stale OpenAI `/v1` URL was previously forwarded to Gemini and produced malformed requests. The CLI sets this per provider when a user picks one. |
| `google_thinking_level` | `None` | `"high"`, `"minimal"`, etc. `None` = provider default. |
| `openai_reasoning_effort` | `None` | `"low"`/`"medium"`/`"high"`. |
| `anthropic_effort` | `None` | `"low"`/`"medium"`/`"high"`. |
| `temperature` | `None` | Forwarded to every provider when set; `None` leaves each at its own default. Reasoning models largely ignore it, and no setting makes output bit-identical across runs. |
| `llm_max_retries` | `None` | `None` = provider/SDK default (usually 2). Raise to ride out bursty 429 throttling instead of aborting a run. |
| `checkpoint_enabled` | `False` | LangGraph saves state after each node so a crashed run can resume. |
| `output_language` | `"English"` | Internal agent debate always stays English for reasoning quality; only reports/decision translate. |
| `max_debate_rounds` | `1` | |
| `max_risk_discuss_rounds` | `1` | |
| `max_recur_limit` | `100` | |
| `news_article_limit` | `20` | Per-ticker cap. |
| `global_news_article_limit` | `10` | Macro/global cap. |
| `global_news_lookback_days` | `7` | |
| `global_news_queries` | 5 macro search strings | Fed rates/inflation, S&P earnings/GDP, geopolitical risk, central bank policy, oil/commodities. Extend or replace to broaden coverage. |
| `data_vendors` | see below | Category-level vendor chains. |
| `tool_vendors` | `{}` | Per-tool override; takes precedence over the category default. |
| `benchmark_ticker` | `None` | When set, overrides `benchmark_map` for *all* tickers. |
| `benchmark_map` | see below | Per-exchange-suffix alpha benchmark. |

`benchmark_map`: `.NS`→`^NSEI`, `.BO`→`^BSESN`, `.T`→`^N225`, `.HK`→`^HSI`,
`.L`→`^FTSE`, `.TO`→`^GSPTSE`, `.AX`→`^AXJO`, `.SS`→`000001.SS`,
`.SZ`→`399001.SZ`, `""` (no suffix, i.e. US)→`SPY`.

## `TRADINGAGENTS_*` environment overrides

From `_ENV_OVERRIDES` — settable for non-interactive runs; the CLI's
interactive prompt for the matching setting is skipped when its env var is
already set:

```
TRADINGAGENTS_LLM_PROVIDER          → llm_provider
TRADINGAGENTS_DEEP_THINK_LLM        → deep_think_llm
TRADINGAGENTS_QUICK_THINK_LLM       → quick_think_llm
TRADINGAGENTS_LLM_BACKEND_URL       → backend_url
TRADINGAGENTS_OUTPUT_LANGUAGE       → output_language
TRADINGAGENTS_MAX_DEBATE_ROUNDS     → max_debate_rounds
TRADINGAGENTS_MAX_RISK_ROUNDS       → max_risk_discuss_rounds
TRADINGAGENTS_CHECKPOINT_ENABLED    → checkpoint_enabled
TRADINGAGENTS_BENCHMARK_TICKER      → benchmark_ticker
TRADINGAGENTS_TEMPERATURE           → temperature
TRADINGAGENTS_LLM_MAX_RETRIES       → llm_max_retries
TRADINGAGENTS_GOOGLE_THINKING_LEVEL   → google_thinking_level
TRADINGAGENTS_OPENAI_REASONING_EFFORT → openai_reasoning_effort
TRADINGAGENTS_ANTHROPIC_EFFORT        → anthropic_effort
```

(`TRADINGAGENTS_RESULTS_DIR` / `_CACHE_DIR` / `_MEMORY_LOG_PATH` also work
but go through plain `os.getenv` calls, not this table.)

**Coercion follows the type of the existing default** (bool/int/float/str).
An invalid value — a misspelled boolean like `treu`, a non-numeric int —
**raises `ValueError` at startup** rather than silently falling back to the
default. This is deliberate: a misconfigured unattended run should fail
loudly, not quietly run with the wrong setting.

To add a new env-overridable config key: add one row to `_ENV_OVERRIDES`.
No entry-point script changes are required — coercion is driven entirely by
the type of the key's existing default.

## Data vendor chains

`data_vendors` (category-level, applies to all tools in that category unless
`tool_vendors` overrides a specific one):

```
core_stock_apis:      "a_stock,yfinance"       # also: alpha_vantage
technical_indicators: "a_stock,yfinance"       # also: alpha_vantage
fundamental_data:      "a_stock,yfinance"       # also: alpha_vantage
news_data:             "a_stock,yfinance"       # also: alpha_vantage
signal_data:           "a_stock"                # A-share only, free direct sources
earnings_data:         "yfinance,a_stock"       # note: reversed order, see below
earnings_commentary:   "alpha_vantage"          # needs ALPHA_VANTAGE_API_KEY
macro_data:            "fred"                   # needs FRED_API_KEY
prediction_markets:    "polymarket"             # keyless
```

**A configured chain is the whole chain.** Registering a vendor in
`VENDOR_METHODS` is not sufficient by itself — an unlisted vendor is never
tried, full stop. Order encodes real failure-mode reasoning, not just a
preference ranking:

- **Why `a_stock` leads the four core chains.** It's the only vendor that
  can *decline from the symbol alone*: anything that isn't a 6-digit A-share
  code is refused by string inspection, with no network call, so a US
  ticker falls straight through to `yfinance`. `yfinance` cannot do the
  reverse — asked for `600519` it does not raise, it *succeeds* with "No
  news found for 600519" — and because the router stops at the first
  success, an A-share ticker would silently get an empty answer from the
  wrong vendor if `yfinance` were first. Putting the self-declining vendor
  first removes that failure mode entirely.
- **Why `earnings_data` reverses that order** to `"yfinance,a_stock"`.
  Yahoo publishes a real consensus-revision history (7/30/60/90-day trend
  plus up/down analyst counts) for the venues it covers, A-shares in Yahoo
  form included; 同花顺 (via `a_stock`) offers only a current snapshot with
  no history behind it — so `yfinance` is the better answer when it has one.
  The hazard that puts `a_stock` first elsewhere doesn't apply here: this
  vendor refuses a bare 6-digit code by string inspection with no network
  call, and **raises** on an unknown symbol rather than answering emptily,
  so it can't silently win the chain by going second. Add `alpha_vantage` to
  this list for real announcement dates and release timing (needs an API
  key).
- `earnings_commentary` is Alpha-Vantage-only and premium-gated — left
  configured but keyless installs pay nothing, since the vendor raises
  before any network call and the category just degrades to a stated gap.

`tool_vendors` overrides a specific tool over its category default — e.g.
`{"get_stock_data": "alpha_vantage"}` — and is empty by default.
