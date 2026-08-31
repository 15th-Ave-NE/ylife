---
name: tradingagents
description: "Operational reference for the TradingAgents multi-agent LLM trading framework — a separate git repo checked out at /Users/yuanxili/workspace/TradingAgents (deployed to /opt/tradingagents), the engine behind yStocker's /agents feature. Use this whenever the user works inside the TradingAgents repo, runs its `tradingagents` CLI or `python -m cli.main`, edits tradingagents/default_config.py or a TRADINGAGENTS_* env var, adds or debugs an LLM provider or model (openai, google, anthropic, azure, bedrock, deepseek, qwen, glm, minimax, ollama, openai_compatible, ...), touches llm_clients/ or a data-vendor chain (a_stock, yfinance, alpha_vantage, fred, polymarket), or asks how yStocker picks a model for an agent report. Also check this before pushing to this repo's git remotes, or before assuming an unrecognized model id will fail fast — it won't."
---

# TradingAgents — operations & gotchas

TradingAgents is a LangGraph-based multi-agent LLM trading framework —
analyst team → bull/bear researchers → trader → risk team → portfolio
manager. It lives in its **own git repo** at
`/Users/yuanxili/workspace/TradingAgents`, checked out separately from
`ystocker/` and deployed to `/opt/tradingagents` in production. It is **not**
vendored into yStocker; yStocker's `/agents` feature shells out to it as a
subprocess. There is no `AGENTS.md`/`AGENTS.md` in this repo yet — everything
here was verified directly against the checkout (git remotes, config files,
CHANGELOG, pyproject.toml) rather than assumed.

## Git remotes — check before you push

```
origin   → https://github.com/15th-Ave-NE/TradingAgents.git   (our fork; production tracks this)
upstream → https://github.com/TauricResearch/TradingAgents.git (what it was forked from)
astock   → https://github.com/simonlin1212/TradingAgents-astock.git (community A-share fork)
```

These two used to be named backwards — `origin` was TauricResearch and
`upstream` was our fork — so a bare `git push` from `main` aimed at
TauricResearch by accident. They're now swapped to the normal convention, so
a bare push is safe. yStocker's `deploy.sh` / `deploy/install-tradingagents.sh`
will repoint `/opt/tradingagents`'s `origin` if they still find the old
TauricResearch-as-origin remote there, but will **never** rewrite ystocker's
own remote — a mismatch there is only reported, since silently rewriting a
working remote can turn an SSH deploy-key URL into an HTTPS one the box has
no credentials for. That repointing logic matches by **URL**, not by remote
name, which is why the origin/upstream swap needed no script change.

Old install methods — a pinned TauricResearch commit plus
`deploy/tradingagents.patch`, and later a `git am` series in
`deploy/tradingagents/` — are both **deleted** from ystocker's deploy
tooling. Production now just tracks this fork's `main` directly; don't
resurrect the patch-file pattern.

## Install & run

```bash
pip install .                  # in a venv
tradingagents                  # installed console script (cli.main:app, a Typer app)
python -m cli.main             # equivalent, run from source
docker compose run --rm tradingagents            # Docker alternative
docker compose --profile ollama run --rm tradingagents-ollama   # local models
```

Programmatic usage:

```python
from tradingagents.graph.trading_graph import TradingAgentsGraph
from tradingagents.default_config import DEFAULT_CONFIG

ta = TradingAgentsGraph(debug=True, config=DEFAULT_CONFIG.copy())
_, decision = ta.propagate("NVDA", "2026-01-15")
```

Package layout: `tradingagents/{agents,dataflows,graph,llm_clients}/`,
`default_config.py`, `reporting.py`, `risk_engine.py`. `llm_clients/` holds
`base_client.py`, `factory.py`, `validators.py`, `model_catalog.py`,
`capabilities.py`, and one client per provider.

Markets: any Yahoo-Finance-covered ticker via its exchange suffix — US (no
suffix), `.HK`, `.T`, `.L`, `.NS`/`.BO`, `.TO`, `.AX`, `.SS`/`.SZ`
(A-shares), `*-USD` (crypto). Company identity and the alpha benchmark
resolve automatically per market (see `benchmark_map` in
`references/config-and-vendors.md`).

## The two headline gotchas

**An unrecognized model id does not fail fast.**
`base_client.warn_if_unknown_model()` only emits a `RuntimeWarning` reading
"Continuing anyway" — the run then dies *inside the vendor SDK* on its first
real call, which can be minutes into a run, after spend already happened.
`validators.validate_model` compounds this: it returns `True` unconditionally
for eight providers where any model string is accepted by design —
`ollama`, `openrouter`, `openai_compatible`, `mistral`, `kimi`, `groq`,
`nvidia`, `bedrock` (local/relay/aggregator endpoints where enumerating
every valid model isn't practical) — and it *also* returns `True` for any
provider name it simply doesn't recognize at all. If you're wiring up a new
model id anywhere upstream (like yStocker's `agent_models.py`), don't rely
on this layer to catch a typo — validate before you get here.

**Per-provider thinking levels are not symmetric.** Gemini Pro accepts only
`low`/`high`; Flash also accepts `minimal`/`medium`. `google_client.py`
remaps exactly **one** of the mismatches — `minimal` on Pro becomes `low` —
and forwards `medium` on Pro **verbatim**, which reaches the API and 400s.
Don't assume a level valid for one Google model is valid for another, and
don't assume the client will quietly fix a mismatch for you.

## Config & environment overrides

`tradingagents/default_config.py`'s `DEFAULT_CONFIG` dict is the single
source of truth for run behavior — provider, models, debate depth, data
vendor chains, benchmarks. Nearly every key has a matching `TRADINGAGENTS_*`
environment variable that overrides it (table in `_ENV_OVERRIDES`), and an
invalid value (a misspelled boolean, a non-numeric int) **raises at startup**
rather than silently falling back — this project fails loud on
misconfiguration by design. The full key list and env-var table is in
`references/config-and-vendors.md`; read it before changing a config key or
adding a new override.

The same fail-loud philosophy shows up from the *other* side in yStocker:
`agent_models.py`'s per-job override must be **assigned** into the child
process's environment, never `setdefault`-ed, or a box with one of these
`TRADINGAGENTS_*` vars pinned in its systemd unit will silently win over
whatever the reader picked in the UI — the job record and the finished
report would both show the reader's choice while the run itself used
something else entirely.

## Data vendor chains are literal, not additive

Whatever string is configured for a category or tool **is the entire
fallback chain** — being registered in `VENDOR_METHODS` is not sufficient on
its own; an unlisted vendor is simply never tried. Chain *order* encodes real
failure-mode reasoning, not just preference — see
`references/config-and-vendors.md` for the two concrete examples (why
`a_stock` leads for stock data but `yfinance` leads for earnings) before
reordering a chain.

## Provider API keys

Set the key for whichever provider(s) you use (`.env`, or `.env.enterprise`
for Azure): `OPENAI_API_KEY`, `GOOGLE_API_KEY`, `ANTHROPIC_API_KEY`,
`XAI_API_KEY`, `DEEPSEEK_API_KEY`, `DASHSCOPE_API_KEY`/`_CN` (Qwen),
`ZHIPU_API_KEY`/`_CN` (GLM), `MINIMAX_API_KEY`/`_CN`, `OPENROUTER_API_KEY`,
`MISTRAL_API_KEY`, `MOONSHOT_API_KEY`, `GROQ_API_KEY`, `NVIDIA_API_KEY`.
Optional: `FRED_API_KEY` (macro data), `OPENAI_COMPATIBLE_API_KEY` (vLLM/LM
Studio/llama.cpp/relay), `OLLAMA_BASE_URL` (remote Ollama, default
`http://localhost:11434/v1`). Bedrock needs `pip install ".[bedrock]"` plus
either `AWS_BEARER_TOKEN_BEDROCK` (takes precedence over an ambient
`AWS_PROFILE`) or the normal AWS credential chain, and `AWS_DEFAULT_REGION`.

## Before assuming current behavior: check the CHANGELOG

This project ships frequent correctness patches — don't assume last month's
understanding of a subsystem still holds. Example, v0.3.1 (2026-07-05):
the Alpha Vantage look-ahead filter was silently skipped because the
fundamentals payload is a JSON *string* and the filter only guarded dicts;
checkpoint-resume identity now folds in selected analysts + debate/risk
depth + asset mode, so resuming under different choices can't continue the
wrong graph; `llm_max_retries` / `TRADINGAGENTS_LLM_MAX_RETRIES` was added so
a 429 burst doesn't abort a run; Bedrock gained bearer-token auth via
`AWS_BEARER_TOKEN_BEDROCK`.

## Test culture

`tests/` has ~70 files covering, among others: `test_astock_*` (A-share
pipeline, earnings, signal data/routing, trading constraints),
`test_earnings_analyst.py`, `test_provider_registry.py`,
`test_model_validation.py`, `test_checkpoint_resume.py`,
`test_symbol_normalization_paths.py`, `test_vendor_routing.py`,
`test_seven_report_downstream.py`. Check for an existing test matching the
area you're touching before assuming behavior is unspecified — this
codebase, like yStocker's, tests its sharp edges deliberately.

## Cross-repo planning docs

A feature spanning both this repo and yStocker gets a paired design doc:
`Codex/research-<slug>.md` + `Codex/todo-<slug>.md` (lowercase `Codex/`,
a plain directory — distinct from `.Codex/`). The pair can live in *either*
repo depending on which was the session root when it was written. Two
examples worth knowing exist (contents not fully read — treat these as
pointers, not summaries):

- `ystocker/Codex/research-tradingagents-astock-seven-analysts.md` +
  `todo-...md` — upgrading A-share runs from four analysts to seven (adding
  Policy / Hot-Money / Lock-up analysts). Looks **implemented**: this repo's
  `tests/` has matching `test_astock_analyst_pipeline.py` and
  `test_seven_report_downstream.py`.
- `TradingAgents/Codex/research-earnings-estimate-revision-agent.md` +
  `todo-...md` (modified today) — the opt-in Earnings & Estimate Revision
  Analyst mentioned in this repo's own README. Looks **actively in
  progress**, not yet settled.

## Integration with yStocker

Full detail lives in the `ystocker` skill / that repo's `AGENTS.md` under
"Choosing the model and thinking depth" — brief pointer here: yStocker's
`agent_models.py` maps a small UI-facing table (`google-pro`, `google-flash`,
`google-lite`, `deepseek-pro`, `deepseek-flash`) onto real
`(provider, deep_think_llm, quick_think_llm, thinking-level)` triples from
this repo, and freezes the resolved triple on the job record at submit time
rather than re-resolving later — a queued job must reproduce the choice the
reader made even if the table changes under it before the job runs.

## See also

- `references/config-and-vendors.md` — full `DEFAULT_CONFIG` key list, the
  complete `TRADINGAGENTS_*` env-var table, and the data-vendor chain
  rationale.
- `ystocker` skill — the Flask app that drives this engine in production.
