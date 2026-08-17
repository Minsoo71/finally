# FinAlly — AI Trading Workstation

## Project Specification

## 1. Vision

FinAlly (Finance Ally) is a visually stunning AI-powered trading workstation that streams live market data, lets users trade a simulated portfolio, and integrates an LLM chat assistant that can analyze positions and execute trades on the user's behalf. It looks and feels like a modern Bloomberg terminal with an AI copilot.

This is the capstone project for an agentic AI coding course. It is built entirely by Coding Agents demonstrating how orchestrated AI agents can produce a production-quality full-stack application. Agents interact through files in `planning/`.

## 2. User Experience

### First Launch

The user runs a single Docker command (or a provided start script). A browser opens to `http://localhost:8000`. No login, no signup. They immediately see:

- A watchlist of 10 default tickers with live-updating prices in a grid
- $10,000 in virtual cash
- A dark, data-rich trading terminal aesthetic
- An AI chat panel ready to assist

### What the User Can Do

- **Watch prices stream** — prices flash green (uptick) or red (downtick) with subtle CSS animations that fade
- **View sparkline mini-charts** — price action beside each ticker in the watchlist, accumulated on the frontend from the SSE stream since page load (sparklines fill in progressively)
- **Click a ticker** to see a larger detailed chart in the main chart area
- **Buy and sell shares** — market orders only, instant fill at current price, no fees, no confirmation dialog
- **Monitor their portfolio** — a heatmap (treemap) showing positions sized by weight and colored by P&L, plus a P&L chart tracking total portfolio value over time
- **View a positions table** — ticker, quantity, average cost, current price, unrealized P&L, % change
- **Chat with the AI assistant** — ask about their portfolio, get analysis, and have the AI execute trades and manage the watchlist through natural language
- **Manage the watchlist** — add/remove tickers manually or via the AI chat

### Visual Design

- **Dark theme**: backgrounds around `#0d1117` or `#1a1a2e`, muted gray borders, no pure black
- **Price flash animations**: brief green/red background highlight on price change, fading over ~500ms via CSS transitions
- **Connection status indicator**: a small colored dot (green = connected, yellow = reconnecting, red = disconnected) visible in the header
- **Professional, data-dense layout**: inspired by Bloomberg/trading terminals — every pixel earns its place
- **Responsive but desktop-first**: optimized for wide screens, functional on tablet

### Color Scheme
- Accent Yellow: `#ecad0a`
- Blue Primary: `#209dd7`
- Purple Secondary: `#753991` (submit buttons)

## 3. Architecture Overview

### Single Container, Single Port

```
┌─────────────────────────────────────────────────┐
│  Docker Container (port 8000)                   │
│                                                 │
│  FastAPI (Python/uv)                            │
│  ├── /api/*          REST endpoints             │
│  ├── /api/stream/*   SSE streaming              │
│  └── /*              Static file serving         │
│                      (Next.js export)            │
│                                                 │
│  SQLite database (volume-mounted)               │
│  Background task: market data polling/sim        │
└─────────────────────────────────────────────────┘
```

- **Frontend**: Next.js with TypeScript, built as a static export (`output: 'export'`), served by FastAPI as static files
- **Backend**: FastAPI (Python), managed as a `uv` project
- **Database**: SQLite, single file at `db/finally.db`, volume-mounted for persistence
- **Real-time data**: Server-Sent Events (SSE) — simpler than WebSockets, one-way server→client push, works everywhere
- **AI integration**: LiteLLM → OpenRouter (Cerebras for fast inference), with structured outputs for trade execution
- **Market data**: Environment-variable driven — simulator by default, real data via Massive API if key provided

### Why These Choices

| Decision | Rationale |
|---|---|
| SSE over WebSockets | One-way push is all we need; simpler, no bidirectional complexity, universal browser support |
| Static Next.js export | Single origin, no CORS issues, one port, one container, simple deployment |
| SQLite over Postgres | No auth = no multi-user = no need for a database server; self-contained, zero config |
| Single Docker container | Students run one command; no docker-compose for production, no service orchestration |
| uv for Python | Fast, modern Python project management; reproducible lockfile; what students should learn |
| Market orders only | Eliminates order book, limit order logic, partial fills — dramatically simpler portfolio math |

---

## 4. Directory Structure

```
finally/
├── frontend/                 # Next.js TypeScript project (static export)
├── backend/                  # FastAPI uv project (Python)
│   └── db/                   # Schema definitions, seed data, migration logic
├── planning/                 # Project-wide documentation for agents
│   ├── PLAN.md               # This document
│   └── ...                   # Additional agent reference docs
├── scripts/
│   ├── start_mac.sh          # Launch Docker container (macOS/Linux)
│   ├── stop_mac.sh           # Stop Docker container (macOS/Linux)
│   ├── start_windows.ps1     # Launch Docker container (Windows PowerShell)
│   └── stop_windows.ps1      # Stop Docker container (Windows PowerShell)
├── test/                     # Playwright E2E tests + docker-compose.test.yml
├── db/                       # Volume mount target (SQLite file lives here at runtime)
│   └── .gitkeep              # Directory exists in repo; finally.db is gitignored
├── Dockerfile                # Multi-stage build (Node → Python)
├── docker-compose.yml        # Optional convenience wrapper
├── .env                      # Environment variables (gitignored, .env.example committed)
└── .gitignore
```

### Key Boundaries

- **`frontend/`** is a self-contained Next.js project. It knows nothing about Python. It talks to the backend via `/api/*` endpoints and `/api/stream/*` SSE endpoints. Internal structure is up to the Frontend Engineer agent.
- **`backend/`** is a self-contained uv project with its own `pyproject.toml`. It owns all server logic including database initialization, schema, seed data, API routes, SSE streaming, market data, and LLM integration. Internal structure is up to the Backend/Market Data agents.
- **`backend/db/`** contains schema SQL definitions and seed logic. The backend lazily initializes the database on first request — creating tables and seeding default data if the SQLite file doesn't exist or is empty.
- **`db/`** at the top level is the runtime volume mount point. The SQLite file (`db/finally.db`) is created here by the backend and persists across container restarts via Docker volume.
- **`planning/`** contains project-wide documentation, including this plan. All agents reference files here as the shared contract.
- **`test/`** contains Playwright E2E tests and supporting infrastructure (e.g., `docker-compose.test.yml`). Unit tests live within `frontend/` and `backend/` respectively, following each framework's conventions.
- **`scripts/`** contains start/stop scripts that wrap Docker commands.

---

## 5. Environment Variables

```bash
# Required: OpenRouter API key for LLM chat functionality
OPENROUTER_API_KEY=your-openrouter-api-key-here

# Optional: Massive (Polygon.io) API key for real market data
# If not set, the built-in market simulator is used (recommended for most users)
MASSIVE_API_KEY=

# Optional: Set to "true" for deterministic mock LLM responses (testing)
LLM_MOCK=false
```

### Behavior

- If `MASSIVE_API_KEY` is set and non-empty → backend uses Massive REST API for market data
- If `MASSIVE_API_KEY` is absent or empty → backend uses the built-in market simulator
- If `LLM_MOCK=true` → backend returns deterministic mock LLM responses (for E2E tests)
- The backend reads `.env` from the project root (mounted into the container or read via docker `--env-file`)

---

## 6. Market Data

### Two Implementations, One Interface

Both the simulator and the Massive client implement the same abstract interface. The backend selects which to use based on the environment variable. All downstream code (SSE streaming, price cache, frontend) is agnostic to the source.

### Simulator (Default)

- Generates prices using geometric Brownian motion (GBM) with configurable drift and volatility per ticker
- Updates at ~500ms intervals
- Correlated moves across tickers (e.g., tech stocks move together)
- Occasional random "events" — sudden 2-5% moves on a ticker for drama
- Starts from realistic seed prices (e.g., AAPL ~$190, GOOGL ~$175, etc.)
- Runs as an in-process background task — no external dependencies

### Massive API (Optional)

- REST API polling (not WebSocket) — simpler, works on all tiers
- Polls for the union of all watched tickers on a configurable interval
- Free tier (5 calls/min): poll every 15 seconds
- Paid tiers: poll every 2-15 seconds depending on tier
- Parses REST response into the same format as the simulator

### Shared Price Cache

- A single background task (simulator or Massive poller) writes to an in-memory price cache
- The cache holds the latest price, previous price, and timestamp for each ticker
- SSE streams read from this cache and push updates to connected clients
- This architecture supports future multi-user scenarios without changes to the data layer

### SSE Streaming

- Endpoint: `GET /api/stream/prices`
- Long-lived SSE connection; client uses native `EventSource` API
- Server pushes price updates for all tickers known to the system at a regular cadence (~500ms) — in the single-user model this is equivalent to the user's watchlist
- Each SSE event contains ticker, price, previous price, timestamp, and change direction
- Client handles reconnection automatically (EventSource has built-in retry)

---

## 7. Database

### SQLite with Lazy Initialization

The backend checks for the SQLite database on startup (or first request). If the file doesn't exist or tables are missing, it creates the schema and seeds default data. This means:

- No separate migration step
- No manual database setup
- Fresh Docker volumes start with a clean, seeded database automatically

### Schema

All tables include a `user_id` column defaulting to `"default"`. This is hardcoded for now (single-user) but enables future multi-user support without schema migration.

**users_profile** — User state (cash balance)
- `id` TEXT PRIMARY KEY (default: `"default"`)
- `cash_balance` REAL (default: `10000.0`)
- `created_at` TEXT (ISO timestamp)

**watchlist** — Tickers the user is watching
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `added_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**positions** — Current holdings (one row per ticker per user)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `quantity` REAL (fractional shares supported)
- `avg_cost` REAL
- `updated_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**trades** — Trade history (append-only log)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `side` TEXT (`"buy"` or `"sell"`)
- `quantity` REAL (fractional shares supported)
- `price` REAL
- `executed_at` TEXT (ISO timestamp)

**portfolio_snapshots** — Portfolio value over time (for P&L chart). Recorded every 30 seconds by a background task, and immediately after each trade execution.
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `total_value` REAL
- `recorded_at` TEXT (ISO timestamp)

**chat_messages** — Conversation history with LLM
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `role` TEXT (`"user"` or `"assistant"`)
- `content` TEXT
- `actions` TEXT (JSON — trades executed, watchlist changes made; null for user messages)
- `created_at` TEXT (ISO timestamp)

### Default Seed Data

- One user profile: `id="default"`, `cash_balance=10000.0`
- Ten watchlist entries: AAPL, GOOGL, MSFT, AMZN, TSLA, NVDA, META, JPM, V, NFLX

---

## 8. API Endpoints

### Market Data
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/stream/prices` | SSE stream of live price updates |

### Portfolio
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/portfolio` | Current positions, cash balance, total value, unrealized P&L |
| POST | `/api/portfolio/trade` | Execute a trade: `{ticker, quantity, side}` |
| GET | `/api/portfolio/history` | Portfolio value snapshots over time (for P&L chart) |

### Watchlist
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/watchlist` | Current watchlist tickers with latest prices |
| POST | `/api/watchlist` | Add a ticker: `{ticker}` |
| DELETE | `/api/watchlist/{ticker}` | Remove a ticker |

### Chat
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/chat` | Send a message, receive complete JSON response (message + executed actions) |

### System
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/health` | Health check (for Docker/deployment) |

---

## 9. LLM Integration

When writing code to make calls to LLMs, use cerebras-inference skill to use LiteLLM via OpenRouter to the `openrouter/openai/gpt-oss-120b` model with Cerebras as the inference provider. Structured Outputs should be used to interpret the results.

There is an OPENROUTER_API_KEY in the .env file in the project root.

### How It Works

When the user sends a chat message, the backend:

1. Loads the user's current portfolio context (cash, positions with P&L, watchlist with live prices, total portfolio value)
2. Loads recent conversation history from the `chat_messages` table
3. Constructs a prompt with a system message, portfolio context, conversation history, and the user's new message
4. Calls the LLM via LiteLLM → OpenRouter, requesting structured output, using the cerebras-inference skill
5. Parses the complete structured JSON response
6. Auto-executes any trades or watchlist changes specified in the response
7. Stores the message and executed actions in `chat_messages`
8. Returns the complete JSON response to the frontend (no token-by-token streaming — Cerebras inference is fast enough that a loading indicator is sufficient)

### Structured Output Schema

The LLM is instructed to respond with JSON matching this schema:

```json
{
  "message": "Your conversational response to the user",
  "trades": [
    {"ticker": "AAPL", "side": "buy", "quantity": 10}
  ],
  "watchlist_changes": [
    {"ticker": "PYPL", "action": "add"}
  ]
}
```

- `message` (required): The conversational text shown to the user
- `trades` (optional): Array of trades to auto-execute. Each trade goes through the same validation as manual trades (sufficient cash for buys, sufficient shares for sells)
- `watchlist_changes` (optional): Array of watchlist modifications

### Auto-Execution

Trades specified by the LLM execute automatically — no confirmation dialog. This is a deliberate design choice:
- It's a simulated environment with fake money, so the stakes are zero
- It creates an impressive, fluid demo experience
- It demonstrates agentic AI capabilities — the core theme of the course

If a trade fails validation (e.g., insufficient cash), the error is included in the chat response so the LLM can inform the user.

### System Prompt Guidance

The LLM should be prompted as "FinAlly, an AI trading assistant" with instructions to:
- Analyze portfolio composition, risk concentration, and P&L
- Suggest trades with reasoning
- Execute trades when the user asks or agrees
- Manage the watchlist proactively
- Be concise and data-driven in responses
- Always respond with valid structured JSON

### LLM Mock Mode

When `LLM_MOCK=true`, the backend returns deterministic mock responses instead of calling OpenRouter. This enables:
- Fast, free, reproducible E2E tests
- Development without an API key
- CI/CD pipelines

---

## 10. Frontend Design

### Layout

The frontend is a single-page application with a dense, terminal-inspired layout. The specific component architecture and layout system is up to the Frontend Engineer, but the UI should include these elements:

- **Watchlist panel** — grid/table of watched tickers with: ticker symbol, current price (flashing green/red on change), daily change %, and a sparkline mini-chart (accumulated from SSE since page load)
- **Main chart area** — larger chart for the currently selected ticker, with at minimum price over time. Clicking a ticker in the watchlist selects it here.
- **Portfolio heatmap** — treemap visualization where each rectangle is a position, sized by portfolio weight, colored by P&L (green = profit, red = loss)
- **P&L chart** — line chart showing total portfolio value over time, using data from `portfolio_snapshots`
- **Positions table** — tabular view of all positions: ticker, quantity, avg cost, current price, unrealized P&L, % change
- **Trade bar** — simple input area: ticker field, quantity field, buy button, sell button. Market orders, instant fill.
- **AI chat panel** — docked/collapsible sidebar. Message input, scrolling conversation history, loading indicator while waiting for LLM response. Trade executions and watchlist changes shown inline as confirmations.
- **Header** — portfolio total value (updating live), connection status indicator, cash balance

### Technical Notes

- Use `EventSource` for SSE connection to `/api/stream/prices`
- Canvas-based charting library preferred (Lightweight Charts or Recharts) for performance
- Price flash effect: on receiving a new price, briefly apply a CSS class with background color transition, then remove it
- All API calls go to the same origin (`/api/*`) — no CORS configuration needed
- Tailwind CSS for styling with a custom dark theme

---

## 11. Docker & Deployment

### Multi-Stage Dockerfile

```
Stage 1: Node 20 slim
  - Copy frontend/
  - npm install && npm run build (produces static export)

Stage 2: Python 3.12 slim
  - Install uv
  - Copy backend/
  - uv sync (install Python dependencies from lockfile)
  - Copy frontend build output into a static/ directory
  - Expose port 8000
  - CMD: uvicorn serving FastAPI app
```

FastAPI serves the static frontend files and all API routes on port 8000.

### Docker Volume

The SQLite database persists via a named Docker volume:

```bash
docker run -v finally-data:/app/db -p 8000:8000 --env-file .env finally
```

The `db/` directory in the project root maps to `/app/db` in the container. The backend writes `finally.db` to this path.

### Start/Stop Scripts

**`scripts/start_mac.sh`** (macOS/Linux):
- Builds the Docker image if not already built (or if `--build` flag passed)
- Runs the container with the volume mount, port mapping, and `.env` file
- Prints the URL to access the app
- Optionally opens the browser

**`scripts/stop_mac.sh`** (macOS/Linux):
- Stops and removes the running container
- Does NOT remove the volume (data persists)

**`scripts/start_windows.ps1`** / **`scripts/stop_windows.ps1`**: PowerShell equivalents for Windows.

All scripts should be idempotent — safe to run multiple times.

### Optional Cloud Deployment

The container is designed to deploy to AWS App Runner, Render, or any container platform. A Terraform configuration for App Runner may be provided in a `deploy/` directory as a stretch goal, but is not part of the core build.

---

## 12. Testing Strategy

### Unit Tests (within `frontend/` and `backend/`)

**Backend (pytest)**:
- Market data: simulator generates valid prices, GBM math is correct, Massive API response parsing works, both implementations conform to the abstract interface
- Portfolio: trade execution logic, P&L calculations, edge cases (selling more than owned, buying with insufficient cash, selling at a loss)
- LLM: structured output parsing handles all valid schemas, graceful handling of malformed responses, trade validation within chat flow
- API routes: correct status codes, response shapes, error handling

**Frontend (React Testing Library or similar)**:
- Component rendering with mock data
- Price flash animation triggers correctly on price changes
- Watchlist CRUD operations
- Portfolio display calculations
- Chat message rendering and loading state

### E2E Tests (in `test/`)

**Infrastructure**: A separate `docker-compose.test.yml` in `test/` that spins up the app container plus a Playwright container. This keeps browser dependencies out of the production image.

**Environment**: Tests run with `LLM_MOCK=true` by default for speed and determinism.

**Key Scenarios**:
- Fresh start: default watchlist appears, $10k balance shown, prices are streaming
- Add and remove a ticker from the watchlist
- Buy shares: cash decreases, position appears, portfolio updates
- Sell shares: cash increases, position updates or disappears
- Portfolio visualization: heatmap renders with correct colors, P&L chart has data points
- AI chat (mocked): send a message, receive a response, trade execution appears inline
- SSE resilience: disconnect and verify reconnection

---

## 13. Review Notes — Questions, Clarifications & Simplifications

Added by a documentation review pass. Items are grouped by how much they block implementation. Nothing above this section was altered; resolve these by editing the relevant section and deleting the item here.

### A. Blocking — an agent cannot implement these without a decision

**A1. "Daily change %" has no data source.** §2 and §10 require a daily change % in the watchlist, but the `PriceUpdate` model (per `MARKET_DATA_SUMMARY.md`) carries only `previous_price` — the *previous tick*, ~500ms ago, not the previous close. Nothing in the schema, the simulator, or the SSE contract provides a session open or prior close. Options: (a) define "daily change" as **change since page load / process start** and rename the column to "Change %" — simplest, no backend work; (b) have the simulator record a session-open price per ticker and add it to `PriceUpdate`; (c) fetch prior close from Massive (unavailable in simulator mode, so the two sources diverge). **Recommend (a)** — it is honest about what a simulator can know.

**A2. Prices are needed for tickers *not* on the watchlist.** Removing a ticker from the watchlist calls `source.remove_ticker()`, which stops pricing it. If the user holds a position in that ticker, portfolio valuation, the positions table, the heatmap and P&L all break. Rule needed: **the priced ticker set is the union of watchlist ∪ open positions**, and removal from the watchlist must not drop a held ticker from the price source. Section 6 says "the union of all watched tickers" — clarify that "watched" means this union.

**A3. Adding an arbitrary ticker to the watchlist is unspecified.** §8 allows `POST /api/watchlist {ticker}` and §9 lets the LLM add e.g. `PYPL`, but `seed_prices.py` only knows the 10 defaults. What happens for an unknown symbol? Needs: a seed-price rule for unknown tickers in simulator mode (e.g. deterministic pseudo-random price in $20–$500 derived from a hash of the symbol, with default drift/vol), a validation rule (format only? a fixed allowlist?), and the error response for a rejected symbol. Also: does Massive mode validate against the real API before accepting?

**A4. Chat history is persisted but never readable.** `chat_messages` is in the schema and §9 step 7 stores messages, but §8 exposes no `GET /api/chat`. On page reload the chat panel would be empty despite the data existing. Either add `GET /api/chat/history?limit=N` or drop the persistence and state that chat is session-only. **Recommend adding the endpoint** — the table already exists and reload-survives-chat is a better demo.

**A5. Trade validation rules are undefined.** §8 gives the request shape but no semantics. Please specify: minimum quantity and whether fractional buys are allowed from the UI (the schema supports `REAL`); rejection of zero/negative/non-numeric quantity; behaviour when no cached price exists for the ticker; float rounding policy for cash and `avg_cost` (round to cents on write? keep full precision and round only at display?); whether a sell that zeroes a position **deletes** the row or leaves `quantity = 0`; and the exact `avg_cost` formula on a partial sell (convention: average cost is unchanged by sells, realized P&L is not tracked — confirm that realized P&L is genuinely out of scope, since §2 promises only *unrealized* P&L).

**A6. Error contract is unspecified.** No status codes or body shape are given for any failure (insufficient cash, unknown ticker, duplicate watchlist entry, LLM failure). Suggest one documented shape — `{"error": {"code": "INSUFFICIENT_CASH", "message": "..."}}` with 400 for validation, 404 for unknown resource, 502 for upstream LLM/market failures — so frontend and E2E tests can assert against it.

### B. Clarifications — implementable, but two agents could reasonably disagree

**B1. Skill name mismatch.** §9 says "use cerebras-inference skill" twice. The installed skill is named **`cerebras`**. An agent following this literally will fail to invoke it. Fix the name in §9.

**B2. Missing-key behaviour.** §5 marks `OPENROUTER_API_KEY` as *required*, but never says what happens if it is absent and `LLM_MOCK=false`. Fail fast at startup, or degrade to mock mode with a banner? (Degrading silently would make a broken demo look like a working one — recommend failing the `/api/chat` request with a clear error while the rest of the app runs normally.)

**B3. What the mock actually returns.** §9 says "deterministic mock responses" without defining them. E2E scenario "trade execution appears inline" can only be written if the mock's behaviour is pinned down — e.g. *if the message contains "buy" and a ticker, return a trade for 1 share; otherwise return a canned analysis message*. Specify it here so the test author and backend author agree.

**B4. Conversation history window.** §9 step 2 says "recent conversation history" — how many messages, or how many tokens? Suggest a fixed last-N (e.g. 20 messages) for predictability.

**B5. Does the LLM see prices, or a snapshot?** §9 step 1 says "watchlist with live prices". Confirm this is a point-in-time snapshot read from `PriceCache` at request time, and that trades auto-executed in step 6 fill at the price *at execution time*, not the price shown in the prompt — these can differ by several ticks and the difference will show up in the chat transcript.

**B6. How does the header's "portfolio total value (updating live)" update?** Options: frontend recomputes from SSE prices × locally-held positions (no extra requests, recommended), or polls `/api/portfolio`. State which — it determines whether `/api/portfolio` needs to be cheap enough to poll.

**B7. Portfolio snapshot growth.** A row every 30s is ~2,880/day, unbounded, on a persistent volume. `GET /api/portfolio/history` takes no parameters. Add a `?since=`/`?limit=` parameter and either a retention window or downsampling for the chart, or state explicitly that unbounded growth is acceptable for a demo.

**B8. SQLite concurrency.** A background snapshot task writes every 30s while request handlers read and write. Specify `PRAGMA journal_mode=WAL`, a busy timeout, and the connection strategy (connection-per-request vs. a single shared connection with a lock) — otherwise `database is locked` will appear intermittently under the SSE + snapshot load.

**B9. Timestamps.** All schema fields say "ISO timestamp" — confirm UTC with an explicit offset/`Z` suffix, so the frontend charts don't drift by the local timezone.

**B10. Static export routing.** Next.js `output: 'export'` emits `index.html` (and directory-style paths if `trailingSlash: true`). Clarify the FastAPI mount: `/api/*` first, everything else falling back to the exported `index.html`. Worth one line in §11 — it is a common first-run failure.

**B11. Charting library note is inaccurate.** §10 says "Canvas-based charting library preferred (Lightweight Charts or Recharts)". Recharts is **SVG**-based, not canvas. Pick one library and say so — mixing two chart libraries for sparkline / main chart / heatmap / P&L would be the single biggest source of frontend bloat. (Recharts covers line + treemap in one package; Lightweight Charts is faster but has no treemap.)

**B12. Main chart on a fresh load.** Like sparklines, the detail chart is fed from SSE since page load, so a newly-loaded page shows an empty chart until data accumulates. Confirm this is intended and describe the empty state — or add a bounded server-side ring buffer of recent ticks (~5 min) plus a `GET /api/history/{ticker}` so charts are populated immediately. This is a visible first-impression issue for a demo.

**B13. Health check contents.** Define what `/api/health` returns beyond 200 — e.g. market-data source name, whether the cache has been populated, DB reachability. Useful for the Docker healthcheck and for debugging.

### C. Internal inconsistencies

**C1. Volume mount is described two different ways.** §11 shows `docker run -v finally-data:/app/db` (a **named Docker volume**), while the following sentence and §4 say "the `db/` directory in the project root maps to `/app/db`" (a **bind mount**, `-v ./db:/app/db`). These are mutually exclusive: with the named volume, the project's `db/` directory stays empty and `.gitkeep` serves no purpose. Pick one. (Named volume is cleaner for students; bind mount makes the DB file inspectable — which is arguably better for a teaching project.)

**C2. `backend/db/` vs top-level `db/`.** Two directories one path segment apart, with completely different roles (schema code vs. runtime data file). This will be misread by both humans and agents. Rename the code one to `backend/app/storage/` or `backend/app/database/`.

**C3. §6 SSE cadence vs. the shipped implementation.** §6 says the server "pushes price updates ... at a regular cadence (~500ms)", while `MARKET_DATA_SUMMARY.md` describes **version-based change detection** on `PriceCache` — i.e. push on change, not on a fixed clock. Update §6 to match what was built, since §6 is the contract the frontend agent will code against.

### D. Simplification opportunities

**D1. Drop prices from `GET /api/watchlist`.** It returns "tickers with latest prices", but the SSE stream delivers prices for every known ticker within ~500ms of connecting. Returning just the ticker list removes a second source of truth for price (and a class of "the table showed a stale price for half a second" bugs).

**D2. Drop `docker-compose.yml`.** §3 explicitly argues against orchestration and §4 already marks it "optional convenience wrapper". The start scripts plus one documented `docker run` line cover it. One fewer file to keep in sync. (`test/docker-compose.test.yml` still earns its place.)

**D3. Simplify two tables' keys.** `watchlist` and `positions` both carry a UUID `id` *plus* a `UNIQUE(user_id, ticker)` constraint. The composite key is the real identity; the UUID is never referenced by anything. Making `(user_id, ticker)` the primary key removes a column, an index, and UUID-generation code from both. (`trades`, `portfolio_snapshots` and `chat_messages` are genuinely append-only and should keep their UUIDs.)

**D4. Fold `users_profile` down to what is used.** Only `cash_balance` is ever read. `created_at` is written and never used. Keep the table (it is the natural home for future fields), drop the unused column.

**D5. One payload for four widgets.** The positions table, heatmap, P&L figures and header all derive from the same data. Specify `GET /api/portfolio` as the single source — positions with `quantity`, `avg_cost`, `current_price`, `market_value`, `unrealized_pnl`, `pnl_pct`, `weight` — computed server-side once, rather than each frontend component recomputing P&L from raw fields. Removes the risk of four components disagreeing about the same number.

**D6. One start script, not four.** `start_mac.sh` / `stop_mac.sh` / `start_windows.ps1` / `stop_windows.ps1` are four files containing two `docker` commands. Since Python is already a project dependency, a single `scripts/run.py` with `start`/`stop` subcommands would work identically on both platforms. (Counter-argument: shell scripts are more legible to students and require no interpreter assumptions — a reasonable reason to keep them. Flagging the trade-off, not asserting the answer.)

**D7. Consider dropping the treemap.** The heatmap and the positions table convey the same information, and treemap layout is the most implementation-heavy visual in §10 for the least information gained on a portfolio of ~5 positions. If the visual drama is wanted, a row of P&L-coloured cards sized by weight gets ~90% of the effect for ~10% of the code. Worth an explicit keep/cut decision rather than defaulting to keep.

**D8. `CLAUDE.md` inlines this entire document.** Every session loads all of PLAN.md into context via `@planning/PLAN.md`, and it will keep growing as sections are completed. Consider replacing the inline include with a short summary plus a pointer ("read `planning/PLAN.md` when working on X"), mirroring how `MARKET_DATA_SUMMARY.md` is already referenced.

### E. Not addressed anywhere

- **Market hours.** Does the simulator run 24/7, or model a closed market? (24/7 is almost certainly right for a demo — but say so, because a user opening the app on a Sunday will notice either way.)
- **Multiple browser tabs.** Two tabs = two SSE connections against one shared cache. Presumably fine; worth one sentence confirming it is a supported case, since the plan repeatedly reasons from "single user".
- **Resetting the demo.** There is no way to get back to $10,000 short of deleting the volume. A `POST /api/reset` would be ~15 lines and makes the app far more demo-able and E2E-testable.
- **Frontend unit test runner.** §12 names React Testing Library but no runner (Vitest vs. Jest) — pick one, as it affects `frontend/` config.
- **Accessibility / motion.** Price flashing is the core visual effect. A `prefers-reduced-motion` fallback is a few lines of CSS and worth naming in §2.
