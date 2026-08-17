# Market Data Backend — Detailed Design

Implementation-ready, as-built design for the FinAlly market data subsystem: the unified `MarketDataSource` interface, the GBM simulator, the Massive (Polygon.io) REST client, the thread-safe price cache, and the SSE streaming endpoint that serves them to the frontend.

**Status:** The subsystem described here is implemented, tested (73 tests, 84% coverage) and reviewed — see `planning/MARKET_DATA_SUMMARY.md` for the build summary and `planning/archive/MARKET_DATA_REVIEW.md` for the review that produced the final state. This document supersedes the earlier drafts in `planning/archive/` (`MARKET_DATA_DESIGN.md`, `MARKET_INTERFACE.md`, `MARKET_SIMULATOR.md`, `MASSIVE_API.md`) — it reflects the code as it actually exists in `backend/app/market/` today, plus the integration points the rest of the backend (portfolio, watchlist, trade execution) needs to build against.

Everything in Sections 1–9 lives under `backend/app/market/`. Sections 10–13 are guidance for the agents building the rest of the backend on top of it.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [File Structure](#2-file-structure)
3. [Data Model — `models.py`](#3-data-model--modelspy)
4. [Price Cache — `cache.py`](#4-price-cache--cachepy)
5. [Abstract Interface — `interface.py`](#5-abstract-interface--interfacepy)
6. [Seed Prices & Ticker Parameters — `seed_prices.py`](#6-seed-prices--ticker-parameters--seed_pricespy)
7. [GBM Simulator — `simulator.py`](#7-gbm-simulator--simulatorpy)
8. [Massive API Client — `massive_client.py`](#8-massive-api-client--massive_clientpy)
9. [Factory & SSE Streaming — `factory.py`, `stream.py`](#9-factory--sse-streaming)
10. [FastAPI Lifecycle Integration](#10-fastapi-lifecycle-integration)
11. [Watchlist & Portfolio Coordination](#11-watchlist--portfolio-coordination)
12. [Testing Strategy](#12-testing-strategy)
13. [Error Handling, Edge Cases & Configuration](#13-error-handling-edge-cases--configuration)

---

## 1. Architecture Overview

```
                 MarketDataSource (ABC)
                 ┌──────────────┴──────────────┐
                 │                              │
       SimulatorDataSource              MassiveDataSource
       (GBM math engine,                (Polygon.io REST poller,
        default, no API key)             MASSIVE_API_KEY required)
                 │                              │
                 └──────────────┬───────────────┘
                                 ▼
                          PriceCache
                    (thread-safe, in-memory,
                     version-counted)
                                 │
                  ┌──────────────┼──────────────┐
                  ▼              ▼              ▼
         SSE stream       Portfolio        Trade
         /api/stream/     valuation        execution
         prices
```

Both data sources implement the same `MarketDataSource` abstract interface and **push** price updates into a single shared `PriceCache` on their own schedule (simulator: ~500ms, Massive: ~15s). Every downstream consumer — SSE streaming, portfolio valuation, trade execution — reads exclusively from the cache and is completely agnostic to which source is active. This is a **strategy pattern**: `create_market_data_source()` picks the implementation once at startup based on `MASSIVE_API_KEY`, and nothing else in the codebase branches on it again.

### Why push-to-cache instead of pull-on-demand

Decouples timing. The simulator ticks every 500ms; Massive polls every 15s (free tier); SSE always reads the cache at its own fixed cadence. The SSE layer never needs to know the active source's update interval, and adding a third source (e.g. a WebSocket feed) would require zero changes to SSE or the REST layer.

---

## 2. File Structure

```
backend/
  app/
    market/
      __init__.py             # Re-exports: PriceUpdate, PriceCache, MarketDataSource,
                               #   create_market_data_source, create_stream_router
      models.py                # PriceUpdate dataclass
      cache.py                 # PriceCache (thread-safe in-memory store)
      interface.py              # MarketDataSource ABC
      seed_prices.py            # SEED_PRICES, TICKER_PARAMS, DEFAULT_PARAMS, CORRELATION_GROUPS
      simulator.py               # GBMSimulator + SimulatorDataSource
      massive_client.py          # MassiveDataSource
      factory.py                  # create_market_data_source()
      stream.py                   # SSE endpoint (FastAPI router factory)
  market_data_demo.py          # Rich terminal demo (`uv run market_data_demo.py`)
  tests/
    market/
      test_models.py            # 11 tests — models.py: 100%
      test_cache.py              # 13 tests — cache.py: 100%
      test_simulator.py           # 17 tests — simulator.py: 98%
      test_simulator_source.py     # 10 tests — integration tests
      test_factory.py               # 7 tests — factory.py: 100%
      test_massive.py                # 13 tests — massive_client.py: 56% (API calls mocked)
```

Each file has a single responsibility. `app/market/__init__.py` re-exports the public API so the rest of the backend imports from `app.market` without reaching into submodules:

```python
from app.market import PriceCache, PriceUpdate, MarketDataSource, create_market_data_source, create_stream_router
```

---

## 3. Data Model — `models.py`

`PriceUpdate` is the only data structure that leaves the market data layer. Every downstream consumer works exclusively with this type.

```python
"""Data models for market data."""

from __future__ import annotations

import time
from dataclasses import dataclass, field


@dataclass(frozen=True, slots=True)
class PriceUpdate:
    """Immutable snapshot of a single ticker's price at a point in time."""

    ticker: str
    price: float
    previous_price: float
    timestamp: float = field(default_factory=time.time)  # Unix seconds

    @property
    def change(self) -> float:
        """Absolute price change from previous update."""
        return round(self.price - self.previous_price, 4)

    @property
    def change_percent(self) -> float:
        """Percentage change from previous update."""
        if self.previous_price == 0:
            return 0.0
        return round((self.price - self.previous_price) / self.previous_price * 100, 4)

    @property
    def direction(self) -> str:
        """'up', 'down', or 'flat'."""
        if self.price > self.previous_price:
            return "up"
        elif self.price < self.previous_price:
            return "down"
        return "flat"

    def to_dict(self) -> dict:
        """Serialize for JSON / SSE transmission."""
        return {
            "ticker": self.ticker,
            "price": self.price,
            "previous_price": self.previous_price,
            "timestamp": self.timestamp,
            "change": self.change,
            "change_percent": self.change_percent,
            "direction": self.direction,
        }
```

### Design decisions

- **`frozen=True`** — price updates are immutable value objects, safe to share across async tasks and threads without copying or locking.
- **`slots=True`** — memory optimization; many of these are created per second across all tickers.
- **Computed properties** (`change`, `change_percent`, `direction`) are derived from `price`/`previous_price` rather than stored, so they can never drift out of sync with the underlying prices.
- **`to_dict()`** is the single serialization point used by both the SSE endpoint and any future REST response that embeds a price.
- **What "previous_price" means**: the price at the *previous tick* (≈500ms earlier for the simulator, ≈15s for Massive) — not the prior session's close. If the product later needs a "daily change %" for the UI, that requires a separate session-open price tracked outside this model (see §13).

---

## 4. Price Cache — `cache.py`

The central data hub. Data sources write to it; everything else reads from it.

```python
"""Thread-safe in-memory price cache."""

from __future__ import annotations

import time
from threading import Lock

from .models import PriceUpdate


class PriceCache:
    """Thread-safe in-memory cache of the latest price for each ticker.

    Writers: SimulatorDataSource or MassiveDataSource (one at a time).
    Readers: SSE streaming endpoint, portfolio valuation, trade execution.
    """

    def __init__(self) -> None:
        self._prices: dict[str, PriceUpdate] = {}
        self._lock = Lock()
        self._version: int = 0  # Monotonically increasing; bumped on every update

    def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate:
        """Record a new price for a ticker. Returns the created PriceUpdate.

        Automatically computes direction and change from the previous price.
        If this is the first update for the ticker, previous_price == price (direction='flat').
        """
        with self._lock:
            ts = timestamp or time.time()
            prev = self._prices.get(ticker)
            previous_price = prev.price if prev else price

            update = PriceUpdate(
                ticker=ticker,
                price=round(price, 2),
                previous_price=round(previous_price, 2),
                timestamp=ts,
            )
            self._prices[ticker] = update
            self._version += 1
            return update

    def get(self, ticker: str) -> PriceUpdate | None:
        """Get the latest price for a single ticker, or None if unknown."""
        with self._lock:
            return self._prices.get(ticker)

    def get_all(self) -> dict[str, PriceUpdate]:
        """Snapshot of all current prices. Returns a shallow copy."""
        with self._lock:
            return dict(self._prices)

    def get_price(self, ticker: str) -> float | None:
        """Convenience: get just the price float, or None."""
        update = self.get(ticker)
        return update.price if update else None

    def remove(self, ticker: str) -> None:
        """Remove a ticker from the cache (e.g., when removed from watchlist)."""
        with self._lock:
            self._prices.pop(ticker, None)

    @property
    def version(self) -> int:
        """Current version counter. Useful for SSE change detection."""
        return self._version

    def __len__(self) -> int:
        with self._lock:
            return len(self._prices)

    def __contains__(self, ticker: str) -> bool:
        with self._lock:
            return ticker in self._prices
```

### Why a version counter

The SSE loop polls the cache every ~500ms. Without a version counter it would re-serialize and re-send every price on every tick even when nothing changed (Massive only updates every 15s). The counter lets the SSE loop skip sends when nothing is new — see §9 for the loop that uses it:

```python
last_version = -1
while True:
    if price_cache.version != last_version:
        last_version = price_cache.version
        yield format_sse(price_cache.get_all())
    await asyncio.sleep(0.5)
```

### Thread safety rationale

`threading.Lock`, not `asyncio.Lock`, because:
- The Massive client's synchronous `get_snapshot_all()` runs via `asyncio.to_thread()`, i.e. in a real OS thread — `asyncio.Lock` provides no protection there.
- `threading.Lock` correctly serializes access from both sync threads and the async event loop.
- The critical section (a dict lookup + assignment) is tiny, so contention is negligible even at the SSE endpoint's 500ms polling cadence with a handful of concurrent readers.

---

## 5. Abstract Interface — `interface.py`

```python
"""Abstract interface for market data sources."""

from __future__ import annotations

from abc import ABC, abstractmethod


class MarketDataSource(ABC):
    """Contract for market data providers.

    Implementations push price updates into a shared PriceCache on their own
    schedule. Downstream code never calls the data source directly for prices —
    it reads from the cache.

    Lifecycle:
        source = create_market_data_source(cache)
        await source.start(["AAPL", "GOOGL", ...])
        # ... app runs ...
        await source.add_ticker("TSLA")
        await source.remove_ticker("GOOGL")
        # ... app shutting down ...
        await source.stop()
    """

    @abstractmethod
    async def start(self, tickers: list[str]) -> None:
        """Begin producing price updates for the given tickers.

        Starts a background task that periodically writes to the PriceCache.
        Must be called exactly once. Calling start() twice is undefined behavior.
        """

    @abstractmethod
    async def stop(self) -> None:
        """Stop the background task and release resources.

        Safe to call multiple times. After stop(), the source will not write
        to the cache again.
        """

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """Add a ticker to the active set. No-op if already present.

        The next update cycle will include this ticker.
        """

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker from the active set. No-op if not present.

        Also removes the ticker from the PriceCache.
        """

    @abstractmethod
    def get_tickers(self) -> list[str]:
        """Return the current list of actively tracked tickers."""
```

Any third data source (a future WebSocket feed, a different vendor) drops in by implementing these five methods — no changes required anywhere else.

---

## 6. Seed Prices & Ticker Parameters — `seed_prices.py`

Constants only — no logic, no imports beyond type annotations. Shared by the simulator for initial prices, GBM parameters, and correlation grouping.

```python
"""Seed prices and per-ticker parameters for the market simulator."""

# Realistic starting prices for the default watchlist (as of project creation)
SEED_PRICES: dict[str, float] = {
    "AAPL": 190.00,
    "GOOGL": 175.00,
    "MSFT": 420.00,
    "AMZN": 185.00,
    "TSLA": 250.00,
    "NVDA": 800.00,
    "META": 500.00,
    "JPM": 195.00,
    "V": 280.00,
    "NFLX": 600.00,
}

# Per-ticker GBM parameters
# sigma: annualized volatility (higher = more price movement)
# mu: annualized drift / expected return
TICKER_PARAMS: dict[str, dict[str, float]] = {
    "AAPL": {"sigma": 0.22, "mu": 0.05},
    "GOOGL": {"sigma": 0.25, "mu": 0.05},
    "MSFT": {"sigma": 0.20, "mu": 0.05},
    "AMZN": {"sigma": 0.28, "mu": 0.05},
    "TSLA": {"sigma": 0.50, "mu": 0.03},  # High volatility
    "NVDA": {"sigma": 0.40, "mu": 0.08},  # High volatility, strong drift
    "META": {"sigma": 0.30, "mu": 0.05},
    "JPM": {"sigma": 0.18, "mu": 0.04},  # Low volatility (bank)
    "V": {"sigma": 0.17, "mu": 0.04},  # Low volatility (payments)
    "NFLX": {"sigma": 0.35, "mu": 0.05},
}

# Default parameters for tickers not in the list above (dynamically added)
DEFAULT_PARAMS: dict[str, float] = {"sigma": 0.25, "mu": 0.05}

# Correlation groups for the simulator's Cholesky decomposition
# Tickers in the same group have higher intra-group correlation
CORRELATION_GROUPS: dict[str, set[str]] = {
    "tech": {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"},
    "finance": {"JPM", "V"},
}

# Correlation coefficients
INTRA_TECH_CORR = 0.6  # Tech stocks move together
INTRA_FINANCE_CORR = 0.5  # Finance stocks move together
CROSS_GROUP_CORR = 0.3  # Between sectors / unknown tickers
TSLA_CORR = 0.3  # TSLA does its own thing
```

Tickers not in `SEED_PRICES` (e.g. a user or the LLM adds an arbitrary symbol like `PYPL`) get a uniform-random seed price in `$50–$300` and `DEFAULT_PARAMS` — see `GBMSimulator._add_ticker_internal` in §7.

---

## 7. GBM Simulator — `simulator.py`

Two classes: `GBMSimulator` (pure math engine, stateful, holds current prices and advances them) and `SimulatorDataSource` (the `MarketDataSource` implementation wrapping it in an async loop that writes to `PriceCache`).

### 7.1 The math

**Geometric Brownian Motion** — the standard model underlying Black-Scholes — evolves each price as:

```
S(t+dt) = S(t) * exp((mu - sigma^2/2) * dt + sigma * sqrt(dt) * Z)
```

where `S(t)` is the current price, `mu` the annualized drift, `sigma` the annualized volatility, `dt` the time step as a fraction of a trading year, and `Z` a (correlated) standard normal draw. For 500ms ticks over a 252-day, 6.5-hour trading year:

```
dt = 0.5 / (252 * 6.5 * 3600) ≈ 8.48e-8
```

This tiny `dt` produces small, realistic per-tick moves that accumulate naturally, and — because the update is multiplicative via `exp()` — prices can never go negative.

**Correlated moves**: real stocks don't move independently. A Cholesky decomposition `L` of the ticker correlation matrix `C` (`L = cholesky(C)`) turns independent standard normals into correlated ones: `Z_correlated = L @ Z_independent`. Correlation structure: tech stocks at `0.6`, finance stocks at `0.5`, everything else (including TSLA, which is deliberately excluded from the tech coefficient) at `0.3`.

**Random shock events**: each tick, each ticker has a ~0.1% chance of a sudden 2–5% move, for visual drama on the dashboard. At 10 tickers × 2 ticks/sec, expect one roughly every 50 seconds.

### 7.2 `GBMSimulator`

```python
"""GBM-based market simulator."""

from __future__ import annotations

import asyncio
import logging
import math
import random

import numpy as np

from .cache import PriceCache
from .interface import MarketDataSource
from .seed_prices import (
    CORRELATION_GROUPS,
    CROSS_GROUP_CORR,
    DEFAULT_PARAMS,
    INTRA_FINANCE_CORR,
    INTRA_TECH_CORR,
    SEED_PRICES,
    TICKER_PARAMS,
    TSLA_CORR,
)

logger = logging.getLogger(__name__)


class GBMSimulator:
    """Geometric Brownian Motion simulator for correlated stock prices.

    Math:
        S(t+dt) = S(t) * exp((mu - sigma^2/2) * dt + sigma * sqrt(dt) * Z)

    The tiny dt (~8.5e-8 for 500ms ticks over 252 trading days * 6.5h/day)
    produces sub-cent moves per tick that accumulate naturally over time.
    """

    # 500ms expressed as a fraction of a trading year
    # 252 trading days * 6.5 hours/day * 3600 seconds/hour = 5,896,800 seconds
    TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600  # 5,896,800
    DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR  # ~8.48e-8

    def __init__(
        self,
        tickers: list[str],
        dt: float = DEFAULT_DT,
        event_probability: float = 0.001,
    ) -> None:
        self._dt = dt
        self._event_prob = event_probability

        self._tickers: list[str] = []
        self._prices: dict[str, float] = {}
        self._params: dict[str, dict[str, float]] = {}
        self._cholesky: np.ndarray | None = None

        for ticker in tickers:
            self._add_ticker_internal(ticker)
        self._rebuild_cholesky()

    # --- Public API ---

    def step(self) -> dict[str, float]:
        """Advance all tickers by one time step. Returns {ticker: new_price}.

        This is the hot path — called every 500ms. Keep it fast.
        """
        n = len(self._tickers)
        if n == 0:
            return {}

        z_independent = np.random.standard_normal(n)
        z_correlated = self._cholesky @ z_independent if self._cholesky is not None else z_independent

        result: dict[str, float] = {}
        for i, ticker in enumerate(self._tickers):
            params = self._params[ticker]
            mu, sigma = params["mu"], params["sigma"]

            drift = (mu - 0.5 * sigma**2) * self._dt
            diffusion = sigma * math.sqrt(self._dt) * z_correlated[i]
            self._prices[ticker] *= math.exp(drift + diffusion)

            # Random event: ~0.1% chance per tick per ticker
            if random.random() < self._event_prob:
                shock_magnitude = random.uniform(0.02, 0.05)
                shock_sign = random.choice([-1, 1])
                self._prices[ticker] *= 1 + shock_magnitude * shock_sign
                logger.debug(
                    "Random event on %s: %.1f%% %s",
                    ticker, shock_magnitude * 100, "up" if shock_sign > 0 else "down",
                )

            result[ticker] = round(self._prices[ticker], 2)

        return result

    def add_ticker(self, ticker: str) -> None:
        """Add a ticker to the simulation. Rebuilds the correlation matrix."""
        if ticker in self._prices:
            return
        self._add_ticker_internal(ticker)
        self._rebuild_cholesky()

    def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker from the simulation. Rebuilds the correlation matrix."""
        if ticker not in self._prices:
            return
        self._tickers.remove(ticker)
        del self._prices[ticker]
        del self._params[ticker]
        self._rebuild_cholesky()

    def get_price(self, ticker: str) -> float | None:
        """Current price for a ticker, or None if not tracked."""
        return self._prices.get(ticker)

    def get_tickers(self) -> list[str]:
        """Return the list of currently tracked tickers."""
        return list(self._tickers)

    # --- Internals ---

    def _add_ticker_internal(self, ticker: str) -> None:
        """Add a ticker without rebuilding Cholesky (for batch initialization)."""
        if ticker in self._prices:
            return
        self._tickers.append(ticker)
        self._prices[ticker] = SEED_PRICES.get(ticker, random.uniform(50.0, 300.0))
        self._params[ticker] = TICKER_PARAMS.get(ticker, dict(DEFAULT_PARAMS))

    def _rebuild_cholesky(self) -> None:
        """Rebuild the Cholesky decomposition of the ticker correlation matrix.

        Called whenever tickers are added or removed. O(n^2) but n < 50.
        """
        n = len(self._tickers)
        if n <= 1:
            self._cholesky = None
            return

        corr = np.eye(n)
        for i in range(n):
            for j in range(i + 1, n):
                rho = self._pairwise_correlation(self._tickers[i], self._tickers[j])
                corr[i, j] = corr[j, i] = rho

        self._cholesky = np.linalg.cholesky(corr)

    @staticmethod
    def _pairwise_correlation(t1: str, t2: str) -> float:
        """Determine correlation between two tickers based on sector grouping."""
        tech = CORRELATION_GROUPS["tech"]
        finance = CORRELATION_GROUPS["finance"]

        # TSLA is in the tech set but behaves independently
        if t1 == "TSLA" or t2 == "TSLA":
            return TSLA_CORR
        if t1 in tech and t2 in tech:
            return INTRA_TECH_CORR
        if t1 in finance and t2 in finance:
            return INTRA_FINANCE_CORR
        return CROSS_GROUP_CORR
```

### 7.3 `SimulatorDataSource` — async wrapper

```python
class SimulatorDataSource(MarketDataSource):
    """MarketDataSource backed by the GBM simulator.

    Runs a background asyncio task that calls GBMSimulator.step() every
    `update_interval` seconds and writes results to the PriceCache.
    """

    def __init__(
        self,
        price_cache: PriceCache,
        update_interval: float = 0.5,
        event_probability: float = 0.001,
    ) -> None:
        self._cache = price_cache
        self._interval = update_interval
        self._event_prob = event_probability
        self._sim: GBMSimulator | None = None
        self._task: asyncio.Task | None = None

    async def start(self, tickers: list[str]) -> None:
        self._sim = GBMSimulator(tickers=tickers, event_probability=self._event_prob)
        # Seed the cache with initial prices so SSE has data immediately
        for ticker in tickers:
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)
        self._task = asyncio.create_task(self._run_loop(), name="simulator-loop")
        logger.info("Simulator started with %d tickers", len(tickers))

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None
        logger.info("Simulator stopped")

    async def add_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.add_ticker(ticker)
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)
            logger.info("Simulator: added ticker %s", ticker)

    async def remove_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.remove_ticker(ticker)
        self._cache.remove(ticker)
        logger.info("Simulator: removed ticker %s", ticker)

    def get_tickers(self) -> list[str]:
        return self._sim.get_tickers() if self._sim else []

    async def _run_loop(self) -> None:
        """Core loop: step the simulation, write to cache, sleep."""
        while True:
            try:
                if self._sim:
                    prices = self._sim.step()
                    for ticker, price in prices.items():
                        self._cache.update(ticker=ticker, price=price)
            except Exception:
                logger.exception("Simulator step failed")
            await asyncio.sleep(self._interval)
```

### Key behaviors

- **Immediate seeding**: `start()` populates the cache with seed prices *before* the loop begins, so the SSE endpoint has data on its very first tick — no blank-screen delay on first load.
- **Graceful cancellation**: `stop()` cancels the task and awaits it, swallowing `CancelledError`, for clean shutdown during FastAPI lifespan teardown.
- **Exception resilience**: `_run_loop` catches exceptions per-step, so one bad tick can't kill the feed.
- **`get_tickers()` is public on `GBMSimulator`** — `SimulatorDataSource.get_tickers()` calls it rather than reaching into `_tickers`, keeping the boundary between the two classes clean.

---

## 8. Massive API Client — `massive_client.py`

Polls the Massive (formerly Polygon.io) REST snapshot endpoint on a configurable interval for all watched tickers **in one API call**. The synchronous `massive` client runs in `asyncio.to_thread()` so it never blocks the event loop.

```python
"""Massive (Polygon.io) API client for real market data."""

from __future__ import annotations

import asyncio
import logging

from massive import RESTClient
from massive.rest.models import SnapshotMarketType

from .cache import PriceCache
from .interface import MarketDataSource

logger = logging.getLogger(__name__)


class MassiveDataSource(MarketDataSource):
    """MarketDataSource backed by the Massive (Polygon.io) REST API.

    Polls GET /v2/snapshot/locale/us/markets/stocks/tickers for all watched
    tickers in a single API call, then writes results to the PriceCache.

    Rate limits:
      - Free tier: 5 req/min → poll every 15s (default)
      - Paid tiers: higher limits → poll every 2-5s
    """

    def __init__(
        self,
        api_key: str,
        price_cache: PriceCache,
        poll_interval: float = 15.0,
    ) -> None:
        self._api_key = api_key
        self._cache = price_cache
        self._interval = poll_interval
        self._tickers: list[str] = []
        self._task: asyncio.Task | None = None
        self._client: RESTClient | None = None

    async def start(self, tickers: list[str]) -> None:
        self._client = RESTClient(api_key=self._api_key)
        self._tickers = list(tickers)

        # Do an immediate first poll so the cache has data right away
        await self._poll_once()

        self._task = asyncio.create_task(self._poll_loop(), name="massive-poller")
        logger.info(
            "Massive poller started: %d tickers, %.1fs interval", len(tickers), self._interval
        )

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None
        self._client = None
        logger.info("Massive poller stopped")

    async def add_ticker(self, ticker: str) -> None:
        ticker = ticker.upper().strip()
        if ticker not in self._tickers:
            self._tickers.append(ticker)
            logger.info("Massive: added ticker %s (will appear on next poll)", ticker)

    async def remove_ticker(self, ticker: str) -> None:
        ticker = ticker.upper().strip()
        self._tickers = [t for t in self._tickers if t != ticker]
        self._cache.remove(ticker)
        logger.info("Massive: removed ticker %s", ticker)

    def get_tickers(self) -> list[str]:
        return list(self._tickers)

    # --- Internal ---

    async def _poll_loop(self) -> None:
        """Poll on interval. First poll already happened in start()."""
        while True:
            await asyncio.sleep(self._interval)
            await self._poll_once()

    async def _poll_once(self) -> None:
        """Execute one poll cycle: fetch snapshots, update cache."""
        if not self._tickers or not self._client:
            return

        try:
            # The Massive RESTClient is synchronous — run in a thread to
            # avoid blocking the event loop.
            snapshots = await asyncio.to_thread(self._fetch_snapshots)
            processed = 0
            for snap in snapshots:
                try:
                    price = snap.last_trade.price
                    # Massive timestamps are Unix milliseconds → convert to seconds
                    timestamp = snap.last_trade.timestamp / 1000.0
                    self._cache.update(ticker=snap.ticker, price=price, timestamp=timestamp)
                    processed += 1
                except (AttributeError, TypeError) as e:
                    logger.warning(
                        "Skipping snapshot for %s: %s", getattr(snap, "ticker", "???"), e
                    )
            logger.debug("Massive poll: updated %d/%d tickers", processed, len(self._tickers))

        except Exception as e:
            logger.error("Massive poll failed: %s", e)
            # Don't re-raise — the loop retries on the next interval.
            # Common failures: 401 (bad key), 429 (rate limit), network errors.

    def _fetch_snapshots(self) -> list:
        """Synchronous call to the Massive REST API. Runs in a thread."""
        return self._client.get_snapshot_all(
            market_type=SnapshotMarketType.STOCKS,
            tickers=self._tickers,
        )
```

### `massive` client reference — endpoints available for future use

The snapshot-all call above is the only endpoint the poller uses today, but the client exposes more that later work (a ticker detail view, historical charts) can call directly:

```python
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

client = RESTClient(api_key="...")  # or reads MASSIVE_API_KEY from env if omitted

# Single-ticker snapshot (e.g. for a detail panel)
snapshot = client.get_snapshot_ticker(market_type=SnapshotMarketType.STOCKS, ticker="AAPL")
print(snapshot.last_trade.price, snapshot.last_quote.bid_price, snapshot.day.high)

# Previous day's close (useful for a real "daily change %")
for agg in client.get_previous_close_agg(ticker="AAPL"):
    print(agg.close, agg.open, agg.high, agg.low, agg.volume)

# Historical OHLCV bars (for a real chart, not the SSE-accumulated one)
for bar in client.list_aggs(
    ticker="AAPL", multiplier=1, timespan="day", from_="2024-01-01", to="2024-01-31", limit=50000,
):
    print(bar.timestamp, bar.open, bar.high, bar.low, bar.close, bar.volume)
```

**Snapshot response shape** (per ticker, what `get_snapshot_all` / `get_snapshot_ticker` return):

```json
{
  "ticker": "AAPL",
  "day": {
    "open": 129.61, "high": 130.15, "low": 125.07, "close": 125.07,
    "volume": 111237700, "previous_close": 129.61,
    "change": -4.54, "change_percent": -3.50
  },
  "last_trade": { "price": 125.07, "size": 100, "exchange": "XNYS", "timestamp": 1675190399000 },
  "last_quote": { "bid_price": 125.06, "ask_price": 125.08, "bid_size": 500, "ask_size": 1000 }
}
```

`last_trade.timestamp` and `last_quote.timestamp` are **Unix milliseconds** — always divide by 1000 before passing to `PriceCache.update()`, which expects Unix seconds (matching `time.time()`).

### Error handling philosophy

| Error | Behavior |
|-------|----------|
| **401 Unauthorized** (bad key) | Logged as error. Poller keeps running so the user can fix `.env` and restart without a code change. |
| **429 Rate Limited** | Logged as error. Next poll retries after `poll_interval` seconds — no backoff, since the interval already respects the tier's limit. |
| **Network timeout** | Logged as error. Retries automatically on the next cycle. |
| **Malformed snapshot** (one ticker) | That ticker is skipped with a warning; every other ticker in the same batch is still processed. |
| **All tickers fail** | Cache retains last-known prices. SSE keeps streaming stale data — better than the frontend seeing nothing. |

### Import strategy: real dependency, not lazy

`massive` is declared a core dependency in `pyproject.toml` (`massive>=1.0.0`) and imported at module top-level in `massive_client.py`. (An earlier draft used a lazy import inside `start()` to make `massive` optional when only the simulator is used — see `planning/archive/MARKET_DATA_DESIGN.md` §7 — but the review in `planning/archive/MARKET_DATA_REVIEW.md` §3.2 found this made the client hard to mock in tests, since `patch("app.market.massive_client.RESTClient")` needs the name to exist at module level. The fix moved the import to the top; `massive` is now installed unconditionally via `uv sync`, and `MassiveDataSource` is simply never instantiated — see `factory.py` below — when `MASSIVE_API_KEY` is unset.)

---

## 9. Factory & SSE Streaming

### 9.1 Factory — `factory.py`

```python
"""Factory for creating market data sources."""

from __future__ import annotations

import logging
import os

from .cache import PriceCache
from .interface import MarketDataSource
from .massive_client import MassiveDataSource
from .simulator import SimulatorDataSource

logger = logging.getLogger(__name__)


def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    """Create the appropriate market data source based on environment variables.

    - MASSIVE_API_KEY set and non-empty → MassiveDataSource (real market data)
    - Otherwise → SimulatorDataSource (GBM simulation)

    Returns an unstarted source. Caller must await source.start(tickers).
    """
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()

    if api_key:
        logger.info("Market data source: Massive API (real data)")
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    else:
        logger.info("Market data source: GBM Simulator")
        return SimulatorDataSource(price_cache=price_cache)
```

Usage:

```python
price_cache = PriceCache()
source = create_market_data_source(price_cache)
await source.start(initial_tickers)  # e.g. ["AAPL", "GOOGL", ...]
```

### 9.2 SSE Streaming Endpoint — `stream.py`

A FastAPI route holding a long-lived connection, pushing `text/event-stream` updates.

```python
"""SSE streaming endpoint for live price updates."""

from __future__ import annotations

import asyncio
import json
import logging
from collections.abc import AsyncGenerator

from fastapi import APIRouter, Request
from fastapi.responses import StreamingResponse

from .cache import PriceCache

logger = logging.getLogger(__name__)

router = APIRouter(prefix="/api/stream", tags=["streaming"])


def create_stream_router(price_cache: PriceCache) -> APIRouter:
    """Create the SSE streaming router with a reference to the price cache.

    This factory pattern lets us inject the PriceCache without globals.
    """

    @router.get("/prices")
    async def stream_prices(request: Request) -> StreamingResponse:
        """SSE endpoint for live price updates.

        Streams all tracked ticker prices every ~500ms. The client connects
        with EventSource and receives events in the format:

            data: {"AAPL": {"ticker": "AAPL", "price": 190.50, ...}, ...}

        Includes a retry directive so the browser auto-reconnects on
        disconnection (EventSource built-in behavior).
        """
        return StreamingResponse(
            _generate_events(price_cache, request),
            media_type="text/event-stream",
            headers={
                "Cache-Control": "no-cache",
                "Connection": "keep-alive",
                "X-Accel-Buffering": "no",  # Disable nginx buffering if proxied
            },
        )

    return router


async def _generate_events(
    price_cache: PriceCache,
    request: Request,
    interval: float = 0.5,
) -> AsyncGenerator[str, None]:
    """Async generator that yields SSE-formatted price events.

    Sends all prices every `interval` seconds. Stops when the client
    disconnects (detected via request.is_disconnected()).
    """
    yield "retry: 1000\n\n"  # Browser retries after 1s if the connection drops

    last_version = -1
    client_ip = request.client.host if request.client else "unknown"
    logger.info("SSE client connected: %s", client_ip)

    try:
        while True:
            if await request.is_disconnected():
                logger.info("SSE client disconnected: %s", client_ip)
                break

            current_version = price_cache.version
            if current_version != last_version:
                last_version = current_version
                prices = price_cache.get_all()

                if prices:
                    data = {ticker: update.to_dict() for ticker, update in prices.items()}
                    yield f"data: {json.dumps(data)}\n\n"

            await asyncio.sleep(interval)
    except asyncio.CancelledError:
        logger.info("SSE stream cancelled for: %s", client_ip)
```

### Wire format

```
data: {"AAPL":{"ticker":"AAPL","price":190.50,"previous_price":190.42,"timestamp":1707580800.5,"change":0.08,"change_percent":0.042,"direction":"up"},"GOOGL":{"ticker":"GOOGL","price":175.12,...}}

```

Frontend consumption:

```javascript
const eventSource = new EventSource('/api/stream/prices');
eventSource.onmessage = (event) => {
  const prices = JSON.parse(event.data);
  // { "AAPL": { ticker, price, previous_price, change, change_percent, direction, timestamp }, ... }
};
```

### Why poll-and-push, not event-driven

The endpoint polls the cache on a fixed interval rather than being notified by the data source. This produces evenly-spaced updates, which matters because the frontend accumulates ticks into sparkline charts — irregular spacing would visually distort them. The `version` counter (§4) keeps this cheap: a tick where nothing changed costs one integer comparison, no serialization.

---

## 10. FastAPI Lifecycle Integration

The market data system starts and stops with the FastAPI app via the `lifespan` context manager. This is the integration point the rest of the backend (not yet built — see `planning/PLAN.md`) plugs into.

**`backend/app/main.py`:**

```python
from contextlib import asynccontextmanager

from fastapi import FastAPI

from app.market.cache import PriceCache
from app.market.factory import create_market_data_source
from app.market.interface import MarketDataSource
from app.market.stream import create_stream_router


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Manage startup and shutdown of background services."""

    # --- STARTUP ---

    price_cache = PriceCache()
    app.state.price_cache = price_cache

    source = create_market_data_source(price_cache)
    app.state.market_source = source

    # Tickers to track = watchlist ∪ open positions (see §11) — not watchlist alone
    initial_tickers = await load_priced_ticker_set()  # reads from SQLite
    await source.start(initial_tickers)

    stream_router = create_stream_router(price_cache)
    app.include_router(stream_router)

    yield  # App is running

    # --- SHUTDOWN ---
    await source.stop()


app = FastAPI(title="FinAlly", lifespan=lifespan)


def get_price_cache() -> PriceCache:
    return app.state.price_cache


def get_market_source() -> MarketDataSource:
    return app.state.market_source
```

### Accessing market data from other routes

```python
from fastapi import APIRouter, Depends, HTTPException

router = APIRouter(prefix="/api")


@router.post("/portfolio/trade")
async def execute_trade(
    trade: TradeRequest,
    price_cache: PriceCache = Depends(get_price_cache),
):
    current_price = price_cache.get_price(trade.ticker)
    if current_price is None:
        raise HTTPException(400, f"Price not yet available for {trade.ticker}")
    # ... validate cash/shares, write trades + positions rows at current_price ...


@router.post("/watchlist")
async def add_to_watchlist(
    payload: WatchlistAdd,
    source: MarketDataSource = Depends(get_market_source),
):
    # ... insert into watchlist table ...
    await source.add_ticker(payload.ticker.upper().strip())
    # ...
```

---

## 11. Watchlist & Portfolio Coordination

The market data layer tracks whatever ticker set it is told to via `start()` / `add_ticker()` / `remove_ticker()` — it has no concept of "watchlist" or "position". The route layer is responsible for keeping that ticker set correct as the user's watchlist and holdings change.

### The priced-ticker-set rule

**The set of tickers the data source tracks must always be `watchlist ∪ open positions`, not the watchlist alone.** If a user removes a ticker from their watchlist while still holding shares in it, dropping it from the price source would break portfolio valuation, the positions table, the P&L heatmap, and the P&L chart — all of which need a live price for every held ticker regardless of watchlist membership. Concretely:

```python
@router.delete("/watchlist/{ticker}")
async def remove_from_watchlist(
    ticker: str,
    source: MarketDataSource = Depends(get_market_source),
):
    ticker = ticker.upper().strip()
    await db.delete_watchlist_entry(ticker)

    # Only stop tracking the price if there's no open position holding it
    position = await db.get_position(ticker)
    if position is None or position.quantity == 0:
        await source.remove_ticker(ticker)

    return {"status": "ok"}
```

Symmetrically, on app startup `load_priced_ticker_set()` (§10) must union the watchlist table with any ticker that has a nonzero `positions.quantity`, not just read the watchlist.

### Flow: adding a ticker

```
User (or LLM) → POST /api/watchlist {ticker: "PYPL"}
  → Insert into watchlist table (SQLite)
  → await source.add_ticker("PYPL")
      Simulator: adds to GBMSimulator (random $50–$300 seed price + default
                 params since PYPL isn't in SEED_PRICES), rebuilds Cholesky,
                 seeds cache immediately
      Massive:   appends to the tracked-ticker list; price appears on the
                 next poll (up to poll_interval seconds later)
  → Return success
```

Because the simulator seeds the cache synchronously inside `add_ticker()`, a trade against a freshly-added ticker can succeed immediately. Against Massive, there is a real gap of up to `poll_interval` seconds before `price_cache.get_price(ticker)` returns non-`None` — the trade route's `if current_price is None: raise HTTPException(400, ...)` (§10) is the correct, and only necessary, handling for that gap.

### Flow: removing a ticker

```
User (or LLM) → DELETE /api/watchlist/PYPL
  → Delete from watchlist table (SQLite)
  → If no open position in PYPL: await source.remove_ticker("PYPL")
      Simulator: removes from GBMSimulator, rebuilds Cholesky, removes from cache
      Massive:   removes from tracked-ticker list, removes from cache
  → Return success
```

---

## 12. Testing Strategy

The existing suite (`backend/tests/market/`, 73 tests, 84% coverage) is the template for testing any future addition to this layer.

### 12.1 `GBMSimulator` — pure math, no I/O (`test_simulator.py`, 17 tests, 98% coverage)

```python
from app.market.simulator import GBMSimulator
from app.market.seed_prices import SEED_PRICES


class TestGBMSimulator:

    def test_step_returns_all_tickers(self):
        sim = GBMSimulator(tickers=["AAPL", "GOOGL"])
        result = sim.step()
        assert set(result.keys()) == {"AAPL", "GOOGL"}

    def test_prices_are_positive(self):
        """GBM prices can never go negative (exp() is always positive)."""
        sim = GBMSimulator(tickers=["AAPL"])
        for _ in range(10_000):
            assert sim.step()["AAPL"] > 0

    def test_initial_prices_match_seeds(self):
        sim = GBMSimulator(tickers=["AAPL"])
        assert sim.get_price("AAPL") == SEED_PRICES["AAPL"]

    def test_unknown_ticker_gets_random_seed_price(self):
        sim = GBMSimulator(tickers=["ZZZZ"])
        assert 50.0 <= sim.get_price("ZZZZ") <= 300.0

    def test_cholesky_rebuilds_on_add(self):
        sim = GBMSimulator(tickers=["AAPL"])
        assert sim._cholesky is None  # 1 ticker, no correlation matrix needed
        sim.add_ticker("GOOGL")
        assert sim._cholesky is not None
```

Covers: correct ticker set on `step()`, positivity invariant under 10k steps, seed price and param lookup (known and unknown tickers), add/remove semantics (including no-op on duplicate add / unknown remove), Cholesky matrix rebuilding on membership change, and empty-simulator `step()` returning `{}`.

### 12.2 `PriceCache` — thread-safe store (`test_cache.py`, 13 tests, 100% coverage)

```python
from app.market.cache import PriceCache


class TestPriceCache:

    def test_first_update_is_flat(self):
        cache = PriceCache()
        update = cache.update("AAPL", 190.50)
        assert update.direction == "flat"
        assert update.previous_price == 190.50

    def test_direction_up(self):
        cache = PriceCache()
        cache.update("AAPL", 190.00)
        update = cache.update("AAPL", 191.00)
        assert update.direction == "up"
        assert update.change == 1.00

    def test_version_increments(self):
        cache = PriceCache()
        v0 = cache.version
        cache.update("AAPL", 190.00)
        assert cache.version == v0 + 1
```

Covers: update/get round-trip, first-update-is-flat, up/down/flat direction, `remove`, `get_all` snapshot independence, version increments per write, and the `get_price` convenience accessor on hit and miss.

### 12.3 `SimulatorDataSource` — integration (`test_simulator_source.py`, 10 tests)

```python
import asyncio
import pytest
from app.market.cache import PriceCache
from app.market.simulator import SimulatorDataSource


@pytest.mark.asyncio
class TestSimulatorDataSource:

    async def test_start_populates_cache(self):
        cache = PriceCache()
        source = SimulatorDataSource(price_cache=cache, update_interval=0.1)
        await source.start(["AAPL", "GOOGL"])

        # Cache has seed prices immediately — before the first loop tick
        assert cache.get("AAPL") is not None
        assert cache.get("GOOGL") is not None
        await source.stop()

    async def test_add_and_remove_ticker(self):
        cache = PriceCache()
        source = SimulatorDataSource(price_cache=cache, update_interval=0.1)
        await source.start(["AAPL"])

        await source.add_ticker("TSLA")
        assert "TSLA" in source.get_tickers()
        assert cache.get("TSLA") is not None

        await source.remove_ticker("TSLA")
        assert "TSLA" not in source.get_tickers()
        assert cache.get("TSLA") is None
        await source.stop()

    async def test_stop_is_clean(self):
        cache = PriceCache()
        source = SimulatorDataSource(price_cache=cache, update_interval=0.1)
        await source.start(["AAPL"])
        await source.stop()
        await source.stop()  # Double stop should not raise
```

Uses real `asyncio.sleep` with a short `update_interval` (0.05–0.1s) rather than mocking the clock — the loop body is trivial enough that this stays fast and avoids coupling tests to internal timer implementation.

### 12.4 `MassiveDataSource` — mocked API (`test_massive.py`, 13 tests, 56% coverage)

56% coverage here is expected and correct: the real `_fetch_snapshots()` → `RESTClient.get_snapshot_all()` path is never exercised (that would be a live network call), only `_poll_once()`'s orchestration around it is tested, by patching `_fetch_snapshots` directly.

```python
from unittest.mock import MagicMock, patch
import pytest
from app.market.cache import PriceCache
from app.market.massive_client import MassiveDataSource


def _make_snapshot(ticker: str, price: float, timestamp_ms: int) -> MagicMock:
    snap = MagicMock()
    snap.ticker = ticker
    snap.last_trade = MagicMock()
    snap.last_trade.price = price
    snap.last_trade.timestamp = timestamp_ms
    return snap


@pytest.mark.asyncio
class TestMassiveDataSource:

    async def test_poll_updates_cache(self):
        cache = PriceCache()
        source = MassiveDataSource(api_key="test-key", price_cache=cache, poll_interval=60.0)
        source._tickers = ["AAPL", "GOOGL"]
        source._client = MagicMock()  # satisfy the _poll_once guard

        mock_snapshots = [
            _make_snapshot("AAPL", 190.50, 1707580800000),
            _make_snapshot("GOOGL", 175.25, 1707580800000),
        ]
        with patch.object(source, "_fetch_snapshots", return_value=mock_snapshots):
            await source._poll_once()

        assert cache.get_price("AAPL") == 190.50
        assert cache.get_price("GOOGL") == 175.25

    async def test_malformed_snapshot_skipped(self):
        cache = PriceCache()
        source = MassiveDataSource(api_key="test-key", price_cache=cache, poll_interval=60.0)
        source._tickers = ["AAPL", "BAD"]
        source._client = MagicMock()

        good_snap = _make_snapshot("AAPL", 190.50, 1707580800000)
        bad_snap = MagicMock()
        bad_snap.ticker = "BAD"
        bad_snap.last_trade = None  # triggers AttributeError, caught and skipped

        with patch.object(source, "_fetch_snapshots", return_value=[good_snap, bad_snap]):
            await source._poll_once()

        assert cache.get_price("AAPL") == 190.50
        assert cache.get_price("BAD") is None

    async def test_timestamp_conversion(self):
        cache = PriceCache()
        source = MassiveDataSource(api_key="test-key", price_cache=cache, poll_interval=60.0)
        source._tickers = ["AAPL"]
        source._client = MagicMock()

        with patch.object(
            source, "_fetch_snapshots",
            return_value=[_make_snapshot("AAPL", 190.50, 1707580800000)],
        ):
            await source._poll_once()

        assert cache.get("AAPL").timestamp == 1707580800.0  # ms -> s

    async def test_stop_cancels_task(self):
        cache = PriceCache()
        source = MassiveDataSource(api_key="test-key", price_cache=cache, poll_interval=10.0)

        with patch("app.market.massive_client.RESTClient"):
            with patch.object(source, "_fetch_snapshots", return_value=[]):
                await source.start(["AAPL"])

        assert source._task is not None
        assert not source._task.done()
        await source.stop()
        assert source._task is None
```

Also covers: ticker normalization (`add_ticker("aapl")` → tracked as `"AAPL"`, whitespace stripped), API-error resilience (`_poll_once` swallows exceptions from `_fetch_snapshots`), empty-ticker-list short-circuit (no API call made), and idempotent `stop()`.

`test_factory.py` (7 tests, 100% coverage) verifies `create_market_data_source` picks `MassiveDataSource` iff `MASSIVE_API_KEY` is set and non-empty (including whitespace-only being treated as unset), by monkeypatching `os.environ`.

`test_models.py` (11 tests, 100% coverage) verifies `PriceUpdate.change`/`change_percent`/`direction` for up/down/flat/zero-previous-price cases, and that `to_dict()` round-trips through `json.dumps`.

`stream.py` sits at 31% coverage — exercising the SSE generator meaningfully requires a running ASGI test client (`httpx.AsyncClient(app=app)` streaming a live response), which the current suite doesn't include. This is a known gap (`planning/archive/MARKET_DATA_REVIEW.md` §4.2) worth closing when the SSE endpoint is wired into `main.py`.

---

## 13. Error Handling, Edge Cases & Configuration

### 13.1 Startup with an empty watchlist

If the database has no watchlist entries, `start()` receives `[]`. Both sources handle this cleanly — the simulator produces no prices, the Massive poller's `_poll_once` returns immediately (`if not self._tickers: return`). The SSE endpoint sends nothing (`if prices:` guard). When the first ticker is added, the source begins tracking it on the next call.

### 13.2 Price cache miss during a trade

```python
price = price_cache.get_price(ticker)
if price is None:
    raise HTTPException(
        status_code=400,
        detail=f"Price not yet available for {ticker}. Please wait a moment and try again.",
    )
```

The simulator avoids this almost entirely by seeding synchronously in `add_ticker()`. Against Massive there's a real window (up to `poll_interval` seconds) — the 400 above is the correct and sufficient handling; no retry loop is needed since the *next* trade attempt will succeed once the first poll lands.

### 13.3 Invalid Massive API key

A bad key surfaces as a 401 on the first poll. The poller logs it and keeps retrying every `poll_interval` — it does not crash the app or fall back to the simulator (falling back silently would hide a misconfiguration behind what looks like working data). The SSE endpoint keeps streaming (connection is fine), just with no price data, until the key is fixed and the app restarted.

### 13.4 Thread safety under load

`PriceCache`'s `threading.Lock` serializes all reads/writes; the critical section is a dict lookup + assignment. At the project's scale (≤50 tickers, one writer, a handful of SSE readers at 500ms) this is not a bottleneck. If it ever became one, the fix would be a reader/writer lock — not needed here.

### 13.5 Simulator numerical precision

Prices are `round()`ed to 2 decimals on every `PriceCache.update()` write (and independently in `GBMSimulator.step()`). The exponential formulation is numerically stable, and multiplicative updates guarantee positivity — no epsilon-clamping or negative-price guards are needed.

### 13.6 Extending to a real "daily change %" (forward-looking)

`PriceUpdate.previous_price` is the *previous tick*, not the session's opening price — see `planning/PLAN.md` §13 item A1. If a later iteration wants an honest daily-change figure instead of change-since-page-load:

- **Simulator**: record each ticker's price at `SimulatorDataSource.start()` time as a `session_open` value alongside the existing seed-and-cache step, and expose it via a second cache method (e.g. `PriceCache.get_session_open(ticker)`), separate from the tick-to-tick `PriceUpdate`.
- **Massive**: `snapshot.day.previous_close` and `snapshot.day.change_percent` (see §8's response shape) already carry this — pull it out of `_poll_once()` alongside `last_trade.price` today; it's already being fetched in the same API call at no extra cost.

This is not implemented today — noted here so the two sources don't diverge if it's picked up later.

### 13.7 Configuration summary

| Parameter | Location | Default | Description |
|-----------|----------|---------|-------------|
| `MASSIVE_API_KEY` | Environment variable | `""` (empty) | If set and non-empty, use Massive API; otherwise use the simulator |
| `update_interval` | `SimulatorDataSource.__init__` | `0.5` (s) | Time between simulator ticks |
| `event_probability` | `GBMSimulator.__init__` | `0.001` | Chance of a random shock event per ticker per tick |
| `dt` | `GBMSimulator.__init__` | `~8.48e-8` | GBM time step (fraction of a trading year) |
| `poll_interval` | `MassiveDataSource.__init__` | `15.0` (s) | Time between Massive API polls (free tier: 5 req/min) |
| SSE push interval | `_generate_events()` | `0.5` (s) | Cache-polling interval inside the SSE generator |
| SSE retry directive | `_generate_events()` | `1000` (ms) | Browser `EventSource` reconnection delay |

### Package `__init__.py`

```python
"""Market data subsystem for FinAlly.

Public API:
    PriceUpdate         - Immutable price snapshot dataclass
    PriceCache          - Thread-safe in-memory price store
    MarketDataSource    - Abstract interface for data providers
    create_market_data_source - Factory that selects simulator or Massive
    create_stream_router - FastAPI router factory for SSE endpoint
"""

from .cache import PriceCache
from .factory import create_market_data_source
from .interface import MarketDataSource
from .models import PriceUpdate
from .stream import create_stream_router

__all__ = [
    "PriceUpdate",
    "PriceCache",
    "MarketDataSource",
    "create_market_data_source",
    "create_stream_router",
]
```

### `pyproject.toml` — what makes the package buildable

```toml
[project]
name = "finally-backend"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.32.0",
    "numpy>=2.0.0",
    "massive>=1.0.0",
    "rich>=13.0.0",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["app"]
```

The `[tool.hatch.build.targets.wheel] packages = ["app"]` line is required — without it `uv sync` fails with `Unable to determine which files to ship inside the wheel` (this was the one High-severity finding in the code review; it's already fixed in the current `pyproject.toml`).

### Manual/demo verification

```bash
cd backend
uv sync --extra dev
uv run pytest -v                    # 73 tests
uv run pytest --cov=app             # 84% coverage
uv run ruff check app/ tests/       # clean
uv run market_data_demo.py          # live terminal dashboard, 60s, Ctrl+C to exit early
```
