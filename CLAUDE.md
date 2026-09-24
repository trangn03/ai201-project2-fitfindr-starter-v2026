# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

FitFindr is a course assignment (AI201, unit 3/4): a small agent that takes a
thrifting query, searches a mock listings dataset, suggests an outfit from the
user's wardrobe, and writes a "fit card" caption — via a hand-built planning
loop that is later partially moved onto MCP. Most core files ship as
documented stubs with `# TODO` blocks; the docstrings in each file *are* the
spec, not just documentation.

## Setup and commands

```bash
python -m venv .venv
.venv\Scripts\Activate.ps1        # Windows; source .venv/bin/activate elsewhere
pip install -r requirements.txt
copy .env.example .env            # then paste GEMINI_API_KEY in
python test.py                    # environment/API sanity check — run this first
```

Common commands:

```bash
python app.py fields                    # field list for a listing + wardrobe item
python app.py listings --full -n 6      # read raw data before writing tools against it
python app.py examples                  # example queries, including one that matches nothing
python app.py ask '...'                 # run the agent once (single quotes — see below)
python app.py ask                       # interactive loop, blank line to quit
python app.py ask '...' --trace         # print agent.py's trace.step() calls
python app.py ask '...' --empty-wardrobe

python agent.py                         # runs the two built-in example paths
python mcp_server.py                    # start the MCP server (unit 4)
python mcp_client.py                    # list what the MCP server offers
python run_eval.py --label before       # run scenarios.py 5x each, write results/ table
python run_eval.py --label after
python serve.py                         # same agent over HTTP (unit 9)
```

Testing a single tool directly (no loop, no HTTP):

```bash
python -c "from tools import search_listings; print(search_listings('graphic tee', max_price=30))"
python -c "from tools import suggest_outfit; from utils.data_loader import get_example_wardrobe, load_listings; print(suggest_outfit(load_listings()[0], get_example_wardrobe()))"
python -c "from tools import create_fit_card; from utils.data_loader import load_listings; print(create_fit_card('jeans and white sneakers', load_listings()[0]))"
```

There is no lint/format config; `test.py` is the only checked-in check, and it's an environment probe, not a unit test suite.

**Always single-quote queries containing `$`.** In PowerShell, `"under $30"` silently becomes `"under "` (`$30` is read as an unset variable) with no error — you get a search with no price ceiling.

## Architecture

Everything funnels through `generate.py::generate()` — the single function that talks to the model (`google-genai`, model name from `config.MODEL`). It owns request pacing (`REQUESTS_PER_MINUTE`), a per-prompt disk cache (`CACHE_ENABLED`, keyed on model+system+prompt+temperature, disabled automatically by `run_eval.py`), a session request budget (`QuotaGuard`), retry/backoff on rate-limit errors, and translates unreachable-model errors into `ModelUnavailable` rather than raw stack traces. Any new model call must go through this function, not a fresh `genai` client.

The three tools (`tools.py`) are independent and individually testable:
- `search_listings(description, size, max_price)` — pure data filtering over `data/listings.json` (via `utils/data_loader.py`), no model call. Returns `[]` on no match (never `None`/exception) — this empty list is exactly what the agent loop branches on.
- `suggest_outfit(new_item, wardrobe)` — calls `generate()`, must special-case an empty wardrobe (`wardrobe['items'] == []`) rather than failing.
- `create_fit_card(outfit, new_item)` — calls `generate()`, must guard against an empty/whitespace `outfit`.

`agent.py::run_agent(query, wardrobe)` is the planning loop and the only graded "intelligence": it parses the query, calls `search_listings`, and **must stop and set `session["error"]` without calling `suggest_outfit`/`create_fit_card` when search returns nothing** — that branch (not the mere sequence of calls) is what's being assessed. All state flows through a single session dict built by `new_session()` (query → parsed → search_results → selected_item → outfit_suggestion → fit_card, plus `error`); nothing is passed tool-to-tool as bare variables so that a caller can log/trace every intermediate value. Callers must check `session["error"]` before touching later fields, which will still be `None` on an early stop. `config.MAX_ITERATIONS` + `trace.check_iterations()` are the loop's own runaway-guard, checked once per iteration.

`mcp_server.py` / `mcp_client.py`: exactly one tool (`search_listings`) is meant to move behind MCP, registered with `@mcp.tool()` in `mcp_server.py` (name = function name = what `call_tool()` looks up) and invoked from `agent.py` via `mcp_client.call_tool("search_listings", {...})` instead of a direct import. `call_tool()` starts/stops a fresh stdio subprocess per call and unwraps MCP's content-block response back into native Python (list/dict) — the contract is that the return shape must not change vs. the direct call. `mcp_client.py` is given as working plumbing and isn't meant to be modified.

`trace.py` provides `start_trace()`/`step()`/`get_trace()` for `--trace` output and the run-log stop condition (`check_iterations`); trace calls belong inside `run_agent()`, one per tool call, with an MCP-routed call labeled to say so (e.g. `search_listings (via MCP)`).

`scenarios.py` defines the fixed set of `(query, wardrobe, criterion)` runs; `run_eval.py` executes each scenario 5x with the cache forced off, against `criteria.md`'s five acceptance criteria, and writes a pass/fail table into `results/` (intentionally not gitignored — it's graded evidence).

`config.py` centralizes the two settings students are told to check first when output looks static (`TEMPERATURE`, `CACHE_ENABLED`), plus `MODEL`, iteration/rate-limit/budget constants, and Windows UTF-8 console setup (every entry point imports `config` for this reason — don't bypass it by running a module that doesn't).

`serve.py` wraps `agent.run_agent()` behind `POST /ask` / `GET /health` for deployment (unit 9); it serializes requests on purpose (one MCP call spins up a whole subprocess, and the free-tier host has no headroom for two concurrent runs), reads `PORT` from the environment, and must be run under plain `gunicorn serve:app` — an async worker breaks every MCP call because `mcp_client.call_tool` uses `asyncio.run()`.

## Constraints particular to this repo

- Do not delete/recreate this repository or rewrite its commit history — grading is tied to the existing commit hashes (see `.github/MAINTAINERS.md`); mirror-push if a copy is ever genuinely needed.
- `README.md` is the actual submission (tool inventory, planning-loop writeup, run logs, etc.) — treat edits to it as filling in a spec, not incidental docs.
- Don't change the `search_listings` / `suggest_outfit` / `create_fit_card` signatures or empty-case return values without also updating the Tool Inventory in `README.md` and the type hints in `mcp_server.py` — the three (docstring spec, MCP schema, README) are meant to agree.
