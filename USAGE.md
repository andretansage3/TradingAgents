# Using TradingAgents

A practical guide to running the framework, with a worked example for **AAOI**
(Applied Optoelectronics). For architecture and contribution conventions see
[`CLAUDE.md`](CLAUDE.md); for the project overview see [`README.md`](README.md).

> **Research software, not financial advice.** Output is LLM-driven and
> non-deterministic — two runs of the same ticker and date can differ.

---

## 1. What you can do with this repo

Give it a **ticker** and a **date**; a simulated trading firm of LLM agents
analyzes it and returns a **5-tier rating** — `Buy / Overweight / Hold /
Underweight / Sell` — plus a full written report from every agent.

### The pipeline

```
Analysts gather data via tools
   ├─ Market Analyst        → price action + technical indicators (MACD, RSI, Bollinger, ATR…)
   ├─ Sentiment Analyst     → news headlines + StockTwits + Reddit mood
   ├─ News Analyst          → global/macro news, insider trades, prediction markets
   └─ Fundamentals Analyst  → balance sheet, cash flow, income statement
        ↓
Bull vs Bear researchers debate
        ↓
Research Manager → writes an investment plan
        ↓
Trader → proposes a concrete trade
        ↓
Risk team debates (Aggressive / Conservative / Neutral)
        ↓
Portfolio Manager → final 5-tier rating
```

A persistent **decision log** records each run; the next run for the same
ticker fetches the realized return (raw and alpha vs a benchmark) and feeds a
reflection back into the Portfolio Manager prompt.

---

## 2. Install

```bash
git clone https://github.com/TauricResearch/TradingAgents.git
cd TradingAgents

conda create -n tradingagents python=3.12 && conda activate tradingagents
pip install .                 # plain install
# or, for development:
pip install -e ".[dev]"       # adds ruff + pytest
pip install ".[bedrock]"      # optional: AWS Bedrock support
```

---

## 3. Configure keys

You need **(a)** one LLM provider key. Data uses **yfinance by default, which is
free and needs no key**; other vendors are optional.

```bash
cp .env.example .env          # then edit
```

**LLM provider — pick one:**

```bash
export OPENAI_API_KEY=...          # OpenAI (default models)
export GOOGLE_API_KEY=...          # Gemini
export ANTHROPIC_API_KEY=...       # Claude
export XAI_API_KEY=...             # Grok
export DEEPSEEK_API_KEY=...        # DeepSeek
# also supported: Qwen, GLM, MiniMax, OpenRouter, Mistral, Kimi, Groq, NVIDIA,
# Azure, AWS Bedrock, Ollama (local), and any OpenAI-compatible endpoint.
```

**Optional data vendors (defaults work without them):**

```bash
export ALPHA_VANTAGE_API_KEY=...   # alternative price/fundamentals/news vendor
export FRED_API_KEY=...            # macro data (rates, inflation) for the News Analyst
# Polymarket (prediction markets) is keyless
```

**Local / enterprise:** for Ollama set `llm_provider: "ollama"` (and
`OLLAMA_BASE_URL` for a remote server); for Azure copy
`.env.enterprise.example`; for any OpenAI-compatible server use
`llm_provider: "openai_compatible"` and set `backend_url`.

---

## 4. Worked example — analyze AAOI

AAOI is a **US-listed** stock, so the ticker needs **no exchange suffix**.
(Non-US markets use a suffix — `0700.HK`, `7203.T`, `RELIANCE.NS`; crypto uses
`BTC-USD`. Identity and the alpha benchmark resolve automatically per market.)

### Option A — Interactive CLI (recommended)

```bash
tradingagents          # installed command
python -m cli.main     # same CLI, from source
```

You'll be prompted in order:

| Prompt | Enter for AAOI |
|---|---|
| **Ticker** | `AAOI` |
| **Analysis date** | e.g. `2026-06-20` (a trading day with available price data) |
| **Analysts** | Space to toggle Market / Sentiment / News / Fundamentals; `a` for all; Enter |
| **Research depth** | `Shallow` (1 debate round) · `Medium` (3) · `Deep` (5) |
| **LLM provider** | OpenAI, Google, Anthropic, … |
| **Quick model** | used by analysts, trader, risk debators |
| **Deep model** | used by research manager + portfolio manager |

The TUI then streams the agents working and prints the final decision. Reports
are written to disk (see §6).

CLI flags:

```bash
tradingagents analyze --checkpoint          # save state after each node; resume a crashed run
tradingagents analyze --clear-checkpoints   # reset all checkpoints before running
```

### Option B — Programmatic (Python)

```python
from tradingagents.graph.trading_graph import TradingAgentsGraph
from tradingagents.default_config import DEFAULT_CONFIG

config = DEFAULT_CONFIG.copy()
config["llm_provider"]    = "openai"
config["deep_think_llm"]  = "gpt-5.5"        # heavy reasoning: research + portfolio mgr
config["quick_think_llm"] = "gpt-5.4-mini"   # analysts, trader, risk debators
config["max_debate_rounds"] = 2              # deeper bull/bear debate

ta = TradingAgentsGraph(debug=True, config=config)
final_state, decision = ta.propagate("AAOI", "2026-06-20")

print(decision)                          # -> Buy / Overweight / Hold / Underweight / Sell
ta.save_reports(final_state, "AAOI")     # write the markdown report tree to disk
```

`final_state` holds every section in memory:
`market_report`, `sentiment_report`, `news_report`, `fundamentals_report`,
`investment_plan`, `trader_investment_plan`, `final_trade_decision`.

### Option C — `.env` + `main.py` (unattended)

Put your choices in `.env` (no code edits), point `main.py` at AAOI, run it:

```bash
# .env
TRADINGAGENTS_LLM_PROVIDER=anthropic
TRADINGAGENTS_DEEP_THINK_LLM=claude-opus-4-8
TRADINGAGENTS_QUICK_THINK_LLM=claude-haiku-4-5-20251001
TRADINGAGENTS_MAX_DEBATE_ROUNDS=2
```

```bash
python main.py     # edit its ta.propagate(...) call to ("AAOI", "2026-06-20")
```

---

## 5. Tuning a run

Set these in code via `config[...]`, **or** as env vars in `.env` (no code edits):

| Goal | Config key | Env var |
|---|---|---|
| Switch provider | `llm_provider` | `TRADINGAGENTS_LLM_PROVIDER` |
| Deep / quick model | `deep_think_llm` / `quick_think_llm` | `TRADINGAGENTS_DEEP_THINK_LLM` / `…_QUICK_THINK_LLM` |
| Debate depth | `max_debate_rounds`, `max_risk_discuss_rounds` | `…_MAX_DEBATE_ROUNDS` / `…_MAX_RISK_ROUNDS` |
| Less run-to-run variation | `temperature` | `TRADINGAGENTS_TEMPERATURE` |
| Report language | `output_language` | `TRADINGAGENTS_OUTPUT_LANGUAGE` |
| Checkpoint resume | `checkpoint_enabled` | `TRADINGAGENTS_CHECKPOINT_ENABLED` |
| Custom endpoint | `backend_url` | `TRADINGAGENTS_LLM_BACKEND_URL` |
| Data vendors | `data_vendors` / `tool_vendors` | — |

See `tradingagents/default_config.py` for every key. Env-var values are coerced
to the default's type and fail loudly on invalid input.

---

## 6. Where results go

Everything lands under `~/.tradingagents/` (override paths via `TRADINGAGENTS_*`):

```
~/.tradingagents/
├── logs/                                       # results_dir
│   ├── AAOI/TradingAgentsStrategy_logs/full_states_log_2026-06-20.json
│   └── reports/AAOI_<timestamp>/               # markdown report tree (CLI / save_reports)
│       ├── 1_analysts/  market.md sentiment.md news.md fundamentals.md
│       └── complete_report.md                  # consolidated full report
├── memory/trading_memory.md                    # decision log — feeds future same-ticker runs
└── cache/checkpoints/AAOI.db                   # only with --checkpoint
```

---

## 7. Notes specific to AAOI

- **Free path works:** with just one LLM key and the default yfinance vendors,
  no paid data subscription is required.
- **Pick a valid date** with available price history. To see the decision log's
  realized-return reflection, run AAOI once, then again days/weeks later.
- **Fundamentals coverage:** AAOI is small/mid-cap; if a vendor lacks a figure,
  that analyst degrades gracefully instead of crashing the run.
- **Determinism:** the analyzed company identity and exact price/indicator
  claims are pinned deterministically; the narrative and rating can still vary.

---

## 8. Development checks

Match the CI gate (`.github/workflows/ci.yml`) before pushing:

```bash
pytest -q                                                    # full suite (no network / no keys needed)
ruff check .                                                 # strict lint, must be clean
pip install . && python -c "import tradingagents, cli.main"  # clean-install smoke
```

---

## 9. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `API key for provider 'openai' is not set` | Export the provider's key (e.g. `OPENAI_API_KEY`) or set it in `.env`. |
| `CONNECT tunnel failed, response 403` / proxy errors | Outbound network is blocked by your environment's policy — data vendors (Yahoo Finance) and LLM APIs are unreachable. Run where outbound HTTPS is allowed. |
| Model "not in the known model list" warning | Harmless — a curated-list miss; the run continues. Use the Custom model ID option for unlisted models. |
| Empty / missing analyst section | That vendor returned no data for the ticker; the pipeline degrades gracefully and continues. |
| Run-to-run differences | Expected (LLM sampling + live news/social). Lower `temperature` and pin the date to reduce, not eliminate, variation. |
