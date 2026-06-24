# CLAUDE.md

Guidance for AI assistants (Claude Code and similar) working in this repository.

## What this project is

**TradingAgents** is a multi-agent LLM framework for financial trading research. It
models a trading firm: specialized LLM agents (analysts, researchers, a trader, a risk
team, and a portfolio manager) collaborate — and debate — to produce a final trade
decision for a given ticker and date. It is built on **LangGraph** and is **research
software, not financial advice**.

The pipeline runs as a directed graph: analysts gather data via tools → bull/bear
researchers debate → research manager writes an investment plan → trader proposes a
trade → a 3-way risk team debates → the portfolio manager issues a final 5-tier rating.

## Repository layout

```
tradingagents/                  # The core package
├── default_config.py           # DEFAULT_CONFIG + TRADINGAGENTS_* env-var overrides
├── reporting.py                # write_report_tree() — shared markdown report writer
├── graph/                      # LangGraph orchestration
│   ├── trading_graph.py        # TradingAgentsGraph — the main entry-point class
│   ├── setup.py                # GraphSetup — builds/wires the StateGraph (nodes + edges)
│   ├── conditional_logic.py    # Branch conditions (when to loop tools / continue debate)
│   ├── analyst_execution.py    # Maps selected analysts → node specs
│   ├── propagation.py          # Initial-state construction + graph run args
│   ├── reflection.py           # Post-hoc reflection on realized returns
│   ├── signal_processing.py    # Extracts the 5-tier rating from the PM decision
│   └── checkpointer.py         # Per-ticker SQLite checkpoint/resume
├── agents/
│   ├── analysts/               # market, news, sentiment(=social), fundamentals
│   ├── researchers/            # bull_researcher, bear_researcher
│   ├── managers/               # research_manager, portfolio_manager
│   ├── risk_mgmt/              # aggressive, conservative, neutral debators
│   ├── trader/                 # trader
│   ├── schemas.py              # Pydantic structured-output schemas + renderers
│   └── utils/
│       ├── agent_states.py     # AgentState (LangGraph state TypedDict), debate states
│       ├── agent_utils.py      # Tool definitions bound to agents (get_stock_data, etc.)
│       ├── structured.py       # bind_structured / invoke_structured_or_freetext helpers
│       ├── rating.py           # Canonical 5-tier rating vocabulary + parser
│       ├── memory.py           # TradingMemoryLog — persistent decision log
│       └── *_tools.py          # Tool groupings (fundamental, news, macro, technical, ...)
├── dataflows/                  # Data-vendor layer (the "where does data come from")
│   ├── interface.py            # Vendor routing: TOOLS_CATEGORIES, VENDOR_METHODS, route_to_vendor
│   ├── config.py               # get_config/set_config (module-level config the tools read)
│   ├── y_finance.py, alpha_vantage*.py, fred.py, polymarket.py, reddit.py, stocktwits.py
│   └── symbol_utils.py         # Ticker normalization + market/exchange detection
└── llm_clients/                # Provider abstraction (the "which LLM")
    ├── factory.py              # create_llm_client(provider, model, ...) dispatcher
    ├── openai_client.py        # OpenAI + the OPENAI_COMPATIBLE_PROVIDERS registry
    ├── anthropic_client.py, google_client.py, azure_client.py, bedrock_client.py
    ├── model_catalog.py        # Curated per-provider model lists for the CLI
    ├── capabilities.py         # Per-model structured-output / tool_choice capabilities
    └── api_key_env.py          # Single source of truth: provider → API-key env var

cli/                            # Interactive Typer/Rich CLI (entry point: cli.main:app)
tests/                          # pytest suite (unit / integration / smoke markers)
scripts/                        # smoke_structured_output.py and other dev scripts
main.py                         # Minimal programmatic-usage example
```

## Running the project

```bash
pip install -e ".[dev]"        # editable install + ruff/pytest (use this for development)
pip install .                  # plain install
pip install ".[bedrock]"       # optional AWS Bedrock support

tradingagents                  # launch the interactive CLI (installed command)
python -m cli.main             # same CLI, run from source
python main.py                 # minimal programmatic example
```

Programmatic usage:

```python
from tradingagents.graph.trading_graph import TradingAgentsGraph
from tradingagents.default_config import DEFAULT_CONFIG

config = DEFAULT_CONFIG.copy()
config["llm_provider"] = "openai"
config["deep_think_llm"] = "gpt-5.5"
config["quick_think_llm"] = "gpt-5.4-mini"

ta = TradingAgentsGraph(debug=True, config=config)
final_state, decision = ta.propagate("NVDA", "2026-01-15")
print(decision)   # one of: Buy / Overweight / Hold / Underweight / Sell
```

## Development workflow & checks

The CI gate (`.github/workflows/ci.yml`) runs three jobs — match them locally before
pushing:

```bash
pytest -q                       # full test suite (matrix: py3.10–3.13 in CI)
ruff check .                    # strict lint, full repo (must be clean)
pip install . && python -c "import tradingagents, cli.main"   # clean-install smoke
```

- **Tests**: `pytest` (config in `pyproject.toml`, `testpaths=["tests"]`). Markers:
  `unit`, `integration`, `smoke` (`--strict-markers` is on). Tests that need live API
  keys or optional deps (`langchain_aws`, `DEEPSEEK_API_KEY`) skip gracefully — keep them
  that way. The suite must pass with **no network and no API keys**.
- **Lint**: ruff, `line-length=100` but `E501` is ignored (formatter owns layout). Rule
  set: `E,W,F,I,B,UP,C4,SIM`. `__init__.py` files ignore `F401` (intentional re-exports).
  The whole repo is clean under this config — keep it that way.
- **Python**: target `>=3.10`. Use modern typing (`str | None`, `dict[str, Any]`).

## Key conventions & architecture notes

- **Three-layer separation.** Keep these concerns apart:
  1. `graph/` — orchestration (who runs when).
  2. `agents/` — prompts and per-agent logic (what each role does).
  3. `dataflows/` — data sourcing; `llm_clients/` — model sourcing.
  Agents call abstract tools (e.g. `get_stock_data` in `agent_utils.py`); the tool routes
  through `dataflows/interface.py:route_to_vendor` to the configured vendor. Don't let an
  agent reach into a specific vendor module directly.

- **Configuration flows through `DEFAULT_CONFIG`.** To add a config key, add it in
  `default_config.py`. To make it overridable from the environment, add one row to
  `_ENV_OVERRIDES` (`TRADINGAGENTS_*` → key) — values are coerced to the existing default's
  type, and invalid values fail loudly at startup. No entry-point changes needed.

- **Provider registry is the single source of truth.** Add an OpenAI-compatible provider
  by adding a `ProviderSpec` row to `OPENAI_COMPATIBLE_PROVIDERS` in `openai_client.py`,
  plus its key env var in `api_key_env.py`. Native (non-OpenAI-wire) providers
  (anthropic/google/azure/bedrock) have their own client and are dispatched in
  `factory.py`. Dual-region providers (qwen/glm/minimax) keep separate `-cn` endpoints
  because international and China credentials cannot be shared.

- **Two LLM tiers per run.** `deep_think_llm` (heavy reasoning: research manager,
  portfolio manager) and `quick_think_llm` (analysts, trader, risk debators, reflection).
  Provider-specific reasoning knobs live in `_get_provider_kwargs` (Google
  `thinking_level`, OpenAI `reasoning_effort`, Anthropic `effort`); `temperature` is
  cross-provider.

- **Structured output with graceful fallback.** Research Manager, Trader, and Portfolio
  Manager emit typed Pydantic objects (`agents/schemas.py`) via the shared helpers in
  `agents/utils/structured.py`. If a model can't do structured output (or a call fails),
  the agent falls back to free-text — the pipeline must never block. Reuse
  `bind_structured` / `invoke_structured_or_freetext` rather than re-implementing this.

- **5-tier rating vocabulary is centralized** in `agents/utils/rating.py`
  (`Buy / Overweight / Hold / Underweight / Sell`). The research manager, portfolio
  manager, signal processor, and memory log all reference it — don't hardcode rating
  strings elsewhere.

- **Determinism mechanisms.** Company identity is resolved deterministically from the
  ticker before any agent runs (`resolve_instrument_context`), and the market analyst must
  ground exact price/indicator claims in `get_verified_market_snapshot`. Preserve these
  when touching the market analyst or instrument resolution — they exist to stop
  hallucinated companies and fabricated price levels.

- **Persistence.** Two kinds, both under `~/.tradingagents/` (override via
  `TRADINGAGENTS_*` env vars): the always-on **decision log** (`memory/trading_memory.md`,
  managed by `TradingMemoryLog`) that feeds realized-return reflections back into the PM
  prompt on the next same-ticker run; and opt-in **checkpoint resume** (`--checkpoint`,
  per-ticker SQLite under `cache/checkpoints/`).

- **Ticker safety.** Tickers become filesystem paths (report trees, logs). Always sanitize
  with `safe_ticker_component` / `dataflows/symbol_utils` helpers — never join a raw ticker
  into a path. Symbol normalization also resolves non-US and crypto symbols to canonical
  Yahoo symbols.

- **Markets.** Any Yahoo-Finance-covered market works via exchange-suffixed tickers
  (`.HK`, `.T`, `.L`, `.NS`, `.SS`, ...) and crypto (`BTC-USD`). The alpha benchmark is
  auto-selected per market via `benchmark_map`.

## Git workflow

- Develop on the assigned feature branch; **create it locally if missing**.
- Commit with clear, descriptive messages; this repo uses Conventional Commit prefixes
  (`feat:`, `fix:`, `docs:`, `chore:`, `test:`) — match that style.
- Push with `git push -u origin <branch>`; retry on network errors with backoff.
- **Do NOT open a pull request unless explicitly asked.**
- Never push to a different branch without explicit permission.

## Gotchas

- The default config models (`gpt-5.5`, `gpt-5.4-mini`) are placeholders for the latest
  OpenAI tier; switch providers/models via config or `TRADINGAGENTS_*` env vars, not by
  editing call sites.
- LLM output is non-deterministic by design — runs of the same ticker/date can differ
  (see README "Reproducibility"). Don't treat run-to-run variation as a bug.
- `dataflows/config.py` holds module-level config that the tool functions read; the graph
  pushes the active config there via `set_config` at startup. Tools read config globally
  rather than receiving it as an argument.
- The `social` analyst was renamed to `sentiment` in v0.2.5; the `social` wire value and
  `create_social_media_analyst` alias are kept for saved-config back-compat. Use
  `sentiment`/`create_sentiment_analyst` for new code.
