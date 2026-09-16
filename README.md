# Mintzy Plugin — Kelly

Automated intraday trading plugin for the Angel One broker. Runs a **paper simulation**, then hands
off to **live trading** on the same session when the simulation is profitable. Built on FastAPI +
Gunicorn, coordinates trader workers across processes with Redis, and persists session/order state
in MongoDB.

> This repository is the maintained/fixed copy. It tracks the audit issues found in review and the
> fixes applied to them — see **Error → Fix log** below.

---

## What it does

- Runs one **AutoTrader worker per session** in an isolated process (own interpreter, broker session,
  Mongo connection).
- Supports three strategy engines:
  | Strategy | File | Role |
  |----------|------|------|
  | `A` | `auto_trader.py` | Standard live trader |
  | `B` | `auto_trader_exposure_expansion.py` | Paper / simulation trader (also the sim→live driver) |
  | `C` | `auto_trader_exposure_expansion_org.py` | Exposure-expansion variant |
- At stop time the simulation applies a **capital pyramid** and produces a handoff result
  (`live_allowed`, `reason`, `profitable_count`, `symbols_for_live`).
- If the handoff says **live_allowed**, the session stays authenticated and live trading starts on the
  same session; otherwise the session is closed and the user creates a new one.
- Session-scoped EOD exits: the engine only exits **session symbols and the exact engine-attributed
  qty** (never a blind full-account flatten).

---

## Architecture

```
                     ┌──────────────────────────── gunicorn (2 uvicorn workers) ───────────────────────────┐
   Frontend / REST ──▶  api_server.py (FastAPI)                                                              │
                     └──────────────┬────────────────────────────────────────────────────────────────────────┘
                                    │ run_in_threadpool() for all blocking work (Mongo, SmartAPI, joins)
                    ┌───────────────▼───────────────────────────────────────────────┐
                    │  SessionManager (session_manager.py)                          │
                    │  - spawn/stop workers (multiprocessing)                       │
                    │  - Redis coordination (shared across gunicorn workers)        │
                    │  - prepare_for_live_start (guarded clear of paper only)       │
                    │  - stop-simulation pipeline + pyramid handoff                 │
                    └───────────────┬───────────────────────────────────────────────┘
                                    │ SIGTERM / Event
                    ┌───────────────▼───────────────────────────────────────────────┐
                    │  _trader_worker (one process per session)                     │
                    │  - broker: broker_angle.SmartAPI (restored session, threadpool-safe)│
                    │  - market feed: LiveLTPStream                                  │
                    │  - engine: AutoTrader A / B / C + parallel order executor      │
                    └───────┬──────────────────────┬─────────────────────────────────┘
                            │                      │
                       ┌────▼─────┐          ┌─────▼──────┐
                       │  Redis   │          │  MongoDB   │
                       │ keys/TTL │          │ sessions/  │
                       │ (see §6) │          │ logs/PNL   │
                       └──────────┘          └────────────┘
```

**Why the threading/process rules matter:** Gunicorn runs only **2 uvicorn workers**. Any blocking call
(Redis, Mongo, broker API, session joins) run directly on the event loop stalls one worker — 50% of
your capacity. Every such call must be offloaded with `run_in_threadpool`. Cross-process state (a live
worker spawned by worker #1 being seen by worker #2) is shared **only through Redis**, never through a
Python class attribute.

---

## Simulation → Live handoff (the "spine")

The chain that decides whether a stopped simulation becomes a live session:

```
API (gunicorn proc)
  → Redis  flag / stop-job          (autotrader:simulation_stop:{id}, autotrader:stop_job:{id})
  → SIGTERM → worker proc
      → shutdown()                  (square-off + capital pyramid)
      → in-process _pyramid_handoff_result
      → Redis pyramid_result:{id}   (TTL 3600)
      → Mongo plugin_sessions.pyramid_handoff   (durable copy)
      → Mongo status simulation_stopped / stopped
  → API reads handoff (chain, Mongo last-resort)
  → API finalizes: authenticated + live start  OR  stopped
```

A failure anywhere in that chain was silently degrading profitable handoffs to `pyramid_not_run`
(fixing that was the whole point of **Round 3** below).

---

## Error → Fix log

Every item below was **verified against the code**, **fixed**, and **shipped** in this repository.

### Round 1 — Gunicorn worker blocking (`35c8030 … c52eae2`)

| # | Error found | Root cause | Fix |
|---|-------------|------------|-----|
| R1-1 | `GET /session` style lookups hung the API under load | `get_or_restore_session` did Mongo decrypt + broker-session restore **on the event loop** | Moved restore off the event loop via `run_in_threadpool` |
| R1-2 | Login/TOTP route blocked a worker | SmartAPI `login`+`generateTOTP` are slow sync calls on the loop | Broker calls off-loaded to threadpool |
| R1-3 | Start/stop routes stalled other requests | `persist_session_metadata` + session stop routes ran blocking IO on the loop | Persistence shipped to `persist_session_metadata_sync` under threadpool; stop routes off-loaded |
| R1-4 | Read-only endpoint latency / jitter | Opportunistic sync broker calls during requests | Confined to threadpool; read-only paths kept non-blocking |

**Why it matters:** with 2 workers, one blocking call halves throughput; a hang froze the whole API.

### Round 2 — Live-start / cross-process coordination (`2b69121`)

| # | Error found | Root cause | Fix |
|---|-------------|------------|-----|
| R2-1 | Live worker force-killed on live start (false “clear”) | Live worker started by gunicorn worker #2 is **invisible** to #1; if its Redis meta was evicted it looked like a paper worker and `prepare_for_live_start` killed it | Refuse to kill undocumented workers; detect cross-process live worker; only clear “provably-our-paper” |
| R2-2 | Evicted pid key caused a live worker to look paper | `_session_has_live_worker` fell back wrongly when `pid` key was LRU-evicted | Republish pid key for live workers; keep strategy on the local record as a second source of truth |
| R2-3 | Concurrent live start could spawn a **second** worker (reservation dead window) | `start_session` wrote the Redis pid key only **after** a fragile meta save; on failed save a concurrent live start saw CLEAR | Live Redis-meta save failure is **always fatal** (terminate worker + raise); fresh-reservation in-flight guard returns NOT_CLEARED |
| R2-4 | Strategy A crashed on single-symbol exit | `AutoTrader` (A) had **no `exit_single_position`** — latent `AttributeError` | Added `exit_single_position` (parallel order executor + pending-order reconciliation); result TTL bumped 60→300s |

**Why it matters:** these were race conditions around the *money transition* — clearing a live worker or
double-spawning on a live session.

### Round 3 — Simulation-stop spine reliability (`5e5f7e7`)

| # | Error found | Root cause | Fix |
|---|-------------|------------|-----|
| R3-1 | Simulation stop misclassified as plain “stopped” when Redis is down → session de-authenticated in Mongo | The SIGTERM handler sets only `stop_event`; the sim-stop discriminator lived **only in Redis** (flag / stop job / latch) | API persists a `simulation_stop_pending` intent marker in Mongo **before signaling**; the worker consumes it as a 4th classifier (fresh within `SIMULATION_STOP_TTL`). Marker cleared on consume and on abort |
| R3-2 | Profitable pyramid handoff silently lost → `pyramid_not_run` → session closed | Handoff had **no durable copy** (Redis `pyramid_result` + `stop_job.pyramid`, both Redis-only; both writes could fail) | Worker writes `pyramid_handoff` to Mongo as a durable copy; API resolves via `stop_job.pyramid → pyramid_result → Mongo` |
| R3-3 | Status endpoint reports `not_started` after the stop already finished; exit results collide | Handoff TTL 600s < slow square-off; three divergent copies; shared `exit_result:{id}:{symbol}` key | TTLs bumped to **3600** (env-overridable); canonical read `read_pyramid_handoff_snapshot`; status endpoint **finalizes from the durable Mongo record** when the job expired; exit results **nonce-scoped** per attempt |
| R3-4 | Mongo status writes fire-and-forget → DB shows session alive while worker is dead | Worker `update_one` inside `try/except` that only prints | `_persist_session_mongo_status`: bounded retry (3×, backoff) + Redis reconciliation marker `autotrader:session:{id}:mongo_status` on final failure |

---

## Repository layout

```
api_server.py                          FastAPI app: start/stop/simulation/live/exit/pnl/CSV endpoints
session_manager.py                    SessionManager + worker lifecycle + Redis coordination + handoff
auto_trader.py                        Strategy A
auto_trader_exposure_expansion.py     Strategy B (paper/sim driver)
auto_trader_exposure_expansion_org.py Strategy C
broker_angle.py                       Angel One SmartAPI wrapper (threadpool-safe calls)
client.py                             Prediction client (market + prediction APIs, key check)
live_ltp_ws.py                        Live LTP WebSocket feed
alerts.py / trading_snapshot.py /      AlertManager, snapshot helpers
csv_snapshot_logger.py                Async CSV writer
utils/redis_keys.py                   CANONICAL Redis key builders (single source of truth)
utils/session_ledger.py               Engine-attributed position ledger (session-scoped EOD)
utils/eod_exit.py / session_symbols   EOD exit helpers + session symbol allow-list
utils/token_manager.py                Broker token wallet
scripts/verify_redis_keys.py          Regression guard: key-format assertions must pass
Dockerfile                            Gunicorn: -w 2 -k uvicorn.workers.UvicornWorker
```

---

## Redis keys

Canonical builders live in `utils/redis_keys.py` — never duplicate the prefix strings elsewhere. The
most important:

| Key pattern | Type | TTL | Purpose |
|-------------|------|-----|---------|
| `autotrader:session:{id}` | STRING | 86400 | Worker PID |
| `autotrader:session:{id}:meta` | STRING | 86400 | Strategy/pid/symbols JSON |
| `autotrader:session:{id}:order_ids` | SET | 86400 | Engine order IDs |
| `autotrader:simulation_stop:{id}` | STRING | 1800 | Sim-stop flag → worker |
| `autotrader:pyramid_result:{id}` | STRING | **3600** | Pyramid handoff |
| `autotrader:stop_job:{id}` | STRING | **3600** | Async stop-job status + handoff |
| `autotrader:exit_request:{id}` | LIST | 300 | Single-symbol exit queue |
| `autotrader:exit_result:{id}:{SYM}:{nonce}` | STRING | 300 | Per-attempt exit result |
| `autotrader:exit_status:{id}` | STRING | 86400 | EOD initiated flag |

`scripts/verify_redis_keys.py` asserts these formats — run it after touching the module.

---

## Environment variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `REDIS_HOST` / `REDIS_PORT` | — / 6379 | Redis for cross-process coordination |
| `MONGO_URI` | **required** | MongoDB connection string (never commit a real one) |
| `MONGO_DB_NAME` | `mintzy_plugin` | Primary DB |
| `MONGO_CONFIG_DB_NAME` | `test` | Config / pyramid PnL DB |
| `MAX_TRADER_WORKERS` | `6` | Worker-process cap |
| `LIVE_START_WORKER_GRACE_SECONDS` | `60` | Fresh-reservation grace at live start |
| `EXIT_ONLY_SESSION_SYMBOLS` | `true` | Exit only session symbols at EOD |
| `EOD_USE_SESSION_LEDGER` | `true` | Exit engine-attributed qty only |
| `EOD_STRICT_REDIS_META` | `true` | Live start fails hard if Redis meta save fails |
| `SIMULATION_STOP_TTL` | `1800` | Sim-stop flag lifetime |
| `PYRAMID_RESULT_TTL` / `STOP_JOB_TTL` | `3600` | Handoff/stop-job lifetime (must outlive square-off) |

Broker credentials (Angel API key, client code, password, TOTP) are supplied at runtime via `.env`
**only** — see **Security note**.

---

## Running / deploying

```bash
# install
pip install -r requirements.txt

# set env (copy env template; never commit real values)
set MONGO_URI=...   # etc.

# run (matches Dockerfile)
gunicorn api_server:app \
  -w 2 -k uvicorn.workers.UvicornWorker \
  --timeout 120 --graceful-timeout 60

# key-format regression guard
py -m scripts.verify_redis_keys
```

---

## Validation

- `py -m py_compile session_manager.py api_server.py utils/redis_keys.py` — all files compile.
- `py -m scripts.verify_redis_keys` — all key assertions pass.
- Manual stop-simulation + live-start matrix on a test VM (see `implementation_plan.md` §5).

---

## Known open items

Tracked but **not yet fixed** in this copy (pull requests welcome):

- `auto_trader.py` calls `self.broker.get_bulk_ltp(...)`, which does not exist on `broker_angle.py`
  (would raise `AttributeError` on that code path).
- A double `_track_engine_fill()` call in the strategy-A exit path.
- `test.py` references a non-existent `Client`.

---

## Security note

This codebase previously shipped **live credentials in tracked files** (Angel client/password/TOTP,
Mongo URIs with embedded passwords, and an active Upstox access token). Before this repository is made
public, those secrets must be moved to environment variables and purged from **git history** — a later
redaction commit does **not** remove them from old commits. Reviewer/ops assumption: one live session per
broker login per day (see `implementation_plan.md` §7).