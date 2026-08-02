# FINNALLY

An AI trading workstation: live simulated market data, portfolio tracking, and an LLM chat assistant that can execute trades and manage a watchlist through natural language.

See `planning/PLAN.md` for the full project specification.

## Stack

- `frontend/` — Next.js (TypeScript), client-rendered
- `backend/` — FastAPI (Python, uv), SQLite
- LLM chat via LiteLLM through OpenRouter (Cerebras inference)

## Status

Planning complete. Implementation has not started yet — see `planning/PLAN.md` and `planning/REVIEW.md`.

## Environment

Copy `.env.example` to `.env` and set `OPENROUTER_API_KEY`. See `planning/PLAN.md` section 5 for all variables.
