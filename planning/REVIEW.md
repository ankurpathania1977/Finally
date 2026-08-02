# Plan Review

Review of `planning/PLAN.md`, 2026-08-02.

## Critical

### 1. A live API key is embedded in the plan

Section 5 contains an OpenRouter credential in plaintext. Treat it as compromised: rotate it, remove it from this document, and replace it with a placeholder such as `OPENROUTER_API_KEY=<your-key-here>`. Keep only the real value in a gitignored local `.env` file. Do not copy the credential into issues, logs, tests, or examples.

### 2. The required LLM integration dependency is unavailable

Section 9 requires a `cerebras-inference` skill, but no such usable skill is available. The plan therefore cannot be implemented as written. Before LLM work begins, either provide that skill with the exact LiteLLM/OpenRouter/Cerebras structured-output workflow, or replace the dependency with concrete, repository-local implementation guidance.

### 3. The plan has no phases or measurable success criteria

The Strategy explicitly requires a phased plan with checkable success criteria, but the document only lists requirements. Add implementation phases, each with acceptance criteria and tests. For example: scaffolding and local startup; market-data/SSE; portfolio and persistence; frontend; LLM/mock mode; Docker and E2E. Without this, “all criteria are met” cannot be verified reliably.

## Contradictions and decisions to resolve

### Persistence scope

“No persistence” in Technical Details conflicts with the SQLite schema and the persistent Docker volume in sections 7 and 11. State the intended rule directly: there are no accounts, authentication, or user-managed sessions; single-user portfolio, trade, watchlist, snapshot, and chat data persist in SQLite.

### Static export versus server-served application

The directory structure calls the Next.js app a “static export,” while section 11 says FastAPI serves the frontend and API/SSE routes from one container. Clarify the build/runtime contract: produce static frontend assets, have FastAPI serve those assets, and let client-side code call same-origin `/api/*`. Also specify the Next.js configuration required for static output and that server-side Next.js features are out of scope.

### Database initialization timing

Section 4 says initialization is lazy on first request, while section 7 says the backend checks on startup “(or first request).” Choose one. Startup initialization is easier to make deterministic for health checks and E2E tests; lazy initialization is acceptable only if every dependent route awaits the same idempotent initializer.

### Single-user design

Every table includes `user_id` solely for a future multi-user feature that is explicitly out of scope. That adds noise and potential uniqueness/query mistakes. Prefer a truly single-user schema for the MVP, or explicitly retain the column as a deliberate future-compatibility trade-off.

### Database volume wording

The plan refers to both a named Docker volume (`finally-data:/app/db`) and the top-level `db/` directory as the runtime mount target. These are different persistence mechanisms. Decide whether the launch scripts use a named volume or a bind mount such as `./db:/app/db`; document one canonical command and test it.

## Missing API contracts

The endpoint list gives paths but not response shapes or error semantics. Define JSON schemas/examples for every endpoint, including:

- `GET /api/portfolio`: cash, positions, total value, cost basis, unrealized P&L, and calculation timestamp.
- `POST /api/portfolio/trade`: ticker normalization, positive quantity requirement, accepted `side` values, execution price source, success response, and validation error status/body.
- Watchlist operations: duplicate-add and missing-remove behavior, ticker validation, and whether removal is allowed when a position remains open.
- `GET /api/portfolio/history`: ordering, interval, initial snapshot guarantee, and empty-history behavior.
- `POST /api/chat`: response schema including successful and failed executed actions; clarify whether failed actions use HTTP failure or an otherwise-successful chat result containing per-action errors.
- SSE: event name, exact payload schema, heartbeat/keepalive behavior, update cadence, and behavior for a ticker added while a client is connected.

This contract should be the source for both unit tests and frontend types.

## Data and financial-modeling gaps

- Define trading validation precisely: cash may not go below zero; sells may not exceed holdings; fractional quantities must be finite and positive; price must be available and positive. Execute cash/position/trade-log updates atomically in one SQLite transaction.
- State a single valuation formula: `cash + sum(quantity * latest_price)`, and define how missing prices are handled. The dashboard, trade validation, snapshots, and LLM context must use the same service.
- Define the simulator’s deterministic configuration for tests (seed, cadence, and clock) so tests do not depend on random price movement or timing.
- Clarify whether the “daily change %” comes from a simulated prior-close field, the previous tick, or the market-data provider. It is not interchangeable with the flash direction.
- Specify a bounded history for client-side sparklines and main-chart samples; retaining every 500 ms tick for a long-lived tab will grow memory without limit.

## Testing and delivery gaps

- Add explicit acceptance criteria for each visual component. “Heatmap renders with correct colors” needs threshold definitions, for example profit/loss color mapping and empty-portfolio state.
- The E2E plan should not rely on arbitrary sleep for SSE. Expose deterministic test hooks or wait for observable UI/API state.
- Include a clean-state test method. Persistent Docker storage otherwise causes tests to contaminate later runs. A dedicated test volume or reset endpoint available only under test configuration would work.
- Include tests for malformed LLM structured output, timeouts/provider errors, and invalid agent-requested trades. Mock mode needs fixture-driven cases beyond a single happy-path reply.
- Define the browser/test startup URL and readiness condition in `docker-compose.test.yml`; `/api/health` should only return ready after database initialization and the price source are usable.

## Documentation cleanup

- Number the opening sections consistently; numbering currently begins at 4.
- Consolidate the no-confirmation, instant-fill simulation rule into one canonical policy and reference it elsewhere.
- Add a chat-history retrieval decision. Messages are persisted, but no endpoint restores them after a browser refresh.
- Verify the specified OpenRouter model/provider route before implementation; provider availability can change. Make the model/provider configurable via environment variables rather than hard-coding an unverified route.

## Recommended next step

Resolve the three critical issues, then revise the plan into phases with API schemas and acceptance criteria before building. This will make the requested unit and Playwright coverage concrete and keep the MVP aligned with its single-user scope.
