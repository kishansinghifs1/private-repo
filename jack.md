# Intelligent DB Kernel — High-Level Architecture

This document describes the **complete system architecture** across all three
tiers. It is a high-level design (HLD) intended for readers who want to
understand *what* the system is, *how the tiers fit together*, and *how the
online reinforcement-learning loop drives query execution*. For a deep dive into
the Java kernel internals, see [`kernel/KERNEL_DESIGN.md`](kernel/KERNEL_DESIGN.md).

---

## 1. What this system is

A from-scratch relational database engine that decides its own query execution
strategy — **Sequential Scan vs. Index Scan** — at runtime, using an **online
reinforcement-learning agent** that learns from the *measured latency* of every
query it runs.

Unlike a traditional cost-based optimizer that relies on static heuristics and
table statistics, this system treats scan-strategy selection as a **contextual
bandit / Q-learning problem**: each query shape is a *state*, each scan type is
an *action*, and the negative of the execution latency is the *reward*. Over
time the agent converges on the fastest strategy for each query shape.

The system is deliberately **fault-tolerant**: if the learning service is
unavailable, the database keeps serving queries using a cost heuristic (marked
`[H]`), protected by a circuit breaker.

---

## 2. The three tiers

| Tier | Tech | Port(s) | Responsibility |
|------|------|---------|----------------|
| **Frontend** | React + Vite | `5173` | Browser SQL playground; renders results, strategy badges, live DB state |
| **Kernel** | Java 17 + Spring Boot 3.4 | `8080` (REST), `9090` (gRPC) | Storage engine, query engine, SQL parser, RL advisor client |
| **Optimizer** | Python 3 + FastAPI + gRPC | `8000` (HTTP), `50051` (gRPC) | Q-learning RL agent that recommends scan strategy and learns from rewards |

```
┌──────────────┐   HTTP /api/execute   ┌──────────────────┐   gRPC :50051   ┌──────────────────┐
│  React SQL   │ ────────────────────► │   Java Kernel    │ ──────────────► │  Python RL       │
│  Playground  │ ◄──────────────────── │  (Spring Boot)   │ ◄────────────── │  Optimizer       │
│  :5173       │   results + strategy  │  :8080 / :9090   │  predict/reward │  :8000 / :50051  │
└──────────────┘                       └──────────────────┘                 └──────────────────┘
```

---

## 3. End-to-end request flow

A single `SELECT ... WHERE` query travels through the system as follows:

```
 1. Browser              User types SQL, hits Run.
        │                POST /api/execute  (text/plain body)
        ▼
 2. Kernel: SqlController Splits the script into statements, runs them serially
        │                under a lock, builds a REPL-style transcript + JSON results.
        ▼
 3. Kernel: SqlParser     Tokenizes/parses one statement into a typed command
        │                (table name + Predicate).
        ▼
 4. Kernel: QueryEngine   Checks the query result cache first (Caffeine).
        │                On miss, builds a QueryState (rows, range?, has-index?).
        ▼
 5. Kernel: ModelAdvisor  gRPC Predict(state) ───────────────► Optimizer
        │                Returns action: 0=SeqScan, 1=IndexScan (or -1 fallback).
        ▼
 6. Kernel: Executor      Runs SeqScanExecutor or IndexScanExecutor over the
        │                storage engine (BufferPool → DiskManager → Page).
        ▼
 7. Kernel: ModelAdvisor  Measures elapsed ns, sends gRPC ReportReward(state,
        │                action, reward=-latency_ms) ────────► Optimizer (async)
        ▼
 8. Optimizer: RLAgent    Normalizes reward, applies Q-update, stores experience,
        │                decays ε, periodically replays a mini-batch.
        ▼
 9. Kernel → Browser      Returns { ok, output, results[{ strategy, elapsedMs,
                          rowCount, ... }] }; UI shows the strategy badge + latency.
```

Steps 5 and 7 are the RL loop. Step 7 is **fire-and-forget** — the query result
is returned to the user without waiting on the reward write.

---

## 4. The reinforcement-learning loop

### 4.1 Formulation

- **State** — a discretized encoding of the query shape. The optimizer buckets
  five features into a compact state key `size-selectivity-index-predType-cardinality`:
  - size bucket (table rows): tiny / small / medium / large
  - selectivity: equality / narrow range / wide range
  - index present: yes / no
  - predicate type: equality / narrow / wide
  - cardinality bucket (estimated matches): low / medium / high
- **Action** — `0 = SeqScan`, `1 = IndexScan`.
- **Reward** — `-latency_ms` (faster execution = higher reward), then
  normalized against a running per-state baseline (EMA) so the agent reacts to
  *relative* improvement rather than absolute machine speed.

### 4.2 Learning algorithm

The agent (`optimizer/agent.py`) is a single-step Q-learning / contextual-bandit
hybrid:

1. **Action selection** — decaying **ε-greedy** for random exploration, plus
   **UCB** (Upper Confidence Bound) to prefer under-explored actions. Unseen
   states get a small heuristic bias (slight IndexScan edge for indexed equality).
2. **Q-update** — `Q(s,a) += α · (normalized_reward − Q(s,a))`.
3. **Experience replay** — every reward is stored in a SQLite-backed buffer
   (`experiences.db`); a mini-batch is periodically replayed to stabilize
   learning.
4. **Persistence** — the Q-table is periodically snapshotted to `q_table.json`
   (with rotating backups) so learning survives restarts.

### 4.3 Why it converges

For a given query shape, IndexScan and SeqScan have systematically different
latencies. As the agent samples both actions and receives latency-derived
rewards, the Q-value of the faster action rises and UCB/ε-greedy stops exploring
the slower one — the policy converges to the correct scan per state.

---

## 5. Resilience & fault tolerance

The kernel must keep working even if the Python optimizer is down. The
`ModelAdvisorService` (gRPC client) is wrapped with **Resilience4j**:

- **Circuit breaker** — opens at a 50% failure rate over a sliding window of 10
  calls (min 5 calls), stays open 30s, then half-opens with 3 trial calls.
- **Retries** — predict: 1 retry (latency-sensitive, 200 ms deadline); reward:
  3 retries (less latency-sensitive, 500 ms deadline).
- **Timeouts / deadlines** — every gRPC call has an explicit deadline.
- **Idempotency keys** — reward writes carry a UUID so retries are safe.
- **Heuristic fallback** — if the optimizer is unreachable, `predict()` returns
  `-1` and the kernel falls back to a cost heuristic (`canUseIndex`), tagging the
  strategy with `[H]`.

Net effect: the database degrades gracefully to a classic heuristic optimizer
instead of failing.

---

## 6. Transport & contracts

- **Frontend ↔ Kernel** — HTTP/JSON. `POST /api/execute` (text/plain SQL body),
  plus `GET /api/state`, `GET /api/health`, cache endpoints. In dev, Vite proxies
  `/api` to `:8080` so there is no CORS setup.
- **Kernel ↔ Optimizer** — **gRPC** over `:50051`, contract defined in
  `kernel/src/main/proto/query_optimizer.proto` (mirrored in `optimizer/proto/`).
  Three RPCs:
  - `Predict(PredictRequest) → PredictResponse` — recommend an action.
  - `ReportReward(RewardRequest) → RewardResponse` — submit learning feedback
    (idempotent).
  - `HealthCheck(HealthCheckRequest) → HealthCheckResponse` — readiness probe.
  The kernel also exposes its **own** gRPC service on `:9090`
  (`query_service.proto`) for query execution/state streaming.
- **Proto contract rules** — additive only: new fields take the next field
  number, removed fields are `reserved`, and field numbers are never reused.

---

## 7. Repository layout

```
Intelligent-DB-Kernel/
├── kernel/          # Java 17 / Spring Boot database engine (Maven, ./mvnw)
│   └── KERNEL_DESIGN.md   # Low-level design of the Java kernel
├── optimizer/       # Python RL optimizer (FastAPI HTTP + gRPC server)
├── frontend/        # React + Vite SQL playground
├── benchmarks/      # Reproducible experiments backing the paper's claims
├── papers/          # Research paper / patent LaTeX sources
├── ARCHITECTURE.md  # ← this file (system-wide HLD)
└── README.md        # Setup & quick-start guide
```

---

## 8. Component responsibilities (system view)

### Frontend (`frontend/`)
- `src/App.jsx` — SQL editor, run button (⌘/Ctrl+Enter), results table,
  strategy badge, live Database panel.
- `src/api.js` — thin fetch client for `/api/execute`, `/api/state`,
  `/api/health`.
- `vite.config.js` — dev proxy of `/api` → `:8080`.

### Kernel (`kernel/`)
- **Gateway** — REST controller + gRPC server; SQL parser.
- **Query Engine** — strategy routing, SeqScan/IndexScan executors, predicate
  evaluation, result caching.
- **Storage Engine** — BufferPool (LRU), DiskManager, Catalog, HeapFile (slotted
  pages), B+Tree indexes.
- **Advisor** — gRPC client to the optimizer with circuit breaker + retries.
- Detailed in [`kernel/KERNEL_DESIGN.md`](kernel/KERNEL_DESIGN.md).

### Optimizer (`optimizer/`)
- `optimizer.py` — FastAPI app (HTTP) + lifespan that also boots the gRPC
  server.
- `agent.py` — the RL agent (state encoding, UCB + ε-greedy, Q-update, replay).
- `q_store.py` — Q-table storage + periodic persistence to `q_table.json`.
- `experience_store.py` — SQLite experience-replay buffer (`experiences.db`).
- `grpc_server.py` — gRPC service implementation.
- `idempotency.py` — idempotency cache for reward writes.
- `config.py` — tunable RL hyperparameters (ε, decay, UCB coefficient, ports).

---

## 9. Configuration surface

| Tier | Where | Key knobs |
|------|-------|-----------|
| Kernel | `kernel/src/main/resources/application.properties` | `server.port` (8080), `grpc.server.port` (9090), `minipostgres.optimizer.grpc.host/port`, `minipostgres.buffer-pool-size` (100), `minipostgres.cache.ttl-seconds` (5) |
| Optimizer | env vars (`optimizer/config.py`) | `RL_PORT` (8000), `RL_GRPC_PORT` (50051), `RL_EPSILON_START` (0.3), `RL_EPSILON_DECAY` (0.994), `RL_UCB_COEFFICIENT` (1.414), `Q_TABLE_FILE`, `EXPERIENCE_DB` |
| Frontend | build env | `VITE_KERNEL_URL` (override kernel origin) |

---

## 10. Startup order & health

Because the kernel probes the optimizer on startup (and falls back gracefully),
the recommended boot order is **Optimizer → Kernel → Frontend**:

1. Optimizer first, so the kernel's startup health probe succeeds and the RL
   loop is live from the first query.
2. Kernel next (`curl http://localhost:8080/api/health`).
3. Frontend last (`npm run dev`, open `http://localhost:5173`).

If the optimizer is not up, the kernel still starts and serves queries with the
`[H]` heuristic fallback until the optimizer becomes reachable.

See the top-level [`README.md`](README.md) for exact commands.
