# Intelligent DB Kernel

A from-scratch database engine that uses **online reinforcement learning** to pick
its query execution strategy (Sequential Scan vs. Index Scan) at runtime, learning
from the actual latency of every query it runs.

The system has three tiers:

| Tier | Tech | Port(s) | Role |
|------|------|---------|------|
| **Frontend** | React + Vite | `5173` | SQL playground in the browser |
| **Kernel** | Java 17 + Spring Boot 3.4 | `8080` (REST), `9090` (gRPC) | Storage engine, query engine, SQL parser |
| **Optimizer** | Python 3 + FastAPI + gRPC | `8000` (HTTP), `50051` (gRPC) | RL agent that recommends scan strategy |

```
┌──────────────┐   HTTP /api/execute   ┌──────────────────┐   gRPC :50051   ┌──────────────────┐
│  React SQL   │ ────────────────────► │   Java Kernel    │ ──────────────► │  Python RL       │
│  Playground  │ ◄──────────────────── │  (Spring Boot)   │ ◄────────────── │  Optimizer       │
│  :5173       │   results + strategy  │  :8080 / :9090   │  predict/reward │  :8000 / :50051  │
└──────────────┘                       └──────────────────┘                 └──────────────────┘
```

The kernel talks to the optimizer over **gRPC**. If the optimizer is down, the kernel
uses a **heuristic fallback** (a circuit breaker trips and scan choices are marked `[H]`),
so the database keeps working even without the RL service.

---

## Repository layout

```
Intelligent-DB-Kernel/
├── kernel/       # Java 17 / Spring Boot database engine (Maven, ./mvnw)
├── optimizer/    # Python RL optimizer (FastAPI + gRPC)
├── frontend/     # React + Vite SQL playground
├── papers/       # Research paper / patent LaTeX sources (not part of the running system)
└── README.md
```

---

## Prerequisites

You need three toolchains. On macOS with [Homebrew](https://brew.sh):

| Tool | Version | Install (macOS / Homebrew) |
|------|---------|----------------------------|
| **JDK** | 17 | `brew install openjdk@17` |
| **Python** | 3.9+ | `brew install python@3.11` (or use system `python3`) |
| **Node.js** | 18+ | `brew install node` |

> **Note on JDK 17:** the Homebrew `openjdk@17` formula is *keg-only* (not auto-added to
> your PATH). Every command below that builds/runs the kernel sets `JAVA_HOME` explicitly:
> ```bash
> export JAVA_HOME=/opt/homebrew/opt/openjdk@17
> export PATH="$JAVA_HOME/bin:$PATH"
> ```
> Add those two lines to your `~/.zshrc` to make it permanent.

Verify:
```bash
export JAVA_HOME=/opt/homebrew/opt/openjdk@17
java -version   # -> openjdk version "17.x"
python3 --version
node --version
```

---

## Quick start (three terminals)

Run each tier in its own terminal, in this order.

### 1. Optimizer (Python RL service)

```bash
cd optimizer
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python -m uvicorn optimizer:app --host 0.0.0.0 --port 8000
```

This starts **both** the FastAPI HTTP server (`:8000`) and the gRPC server (`:50051`)
that the kernel connects to. You should see:
```
RL Optimizer ready — Q-table: N states, Experiences: M
Endpoints: HTTP=8000, gRPC=50051
```

### 2. Kernel (Java database engine)

```bash
cd kernel
export JAVA_HOME=/opt/homebrew/opt/openjdk@17
export PATH="$JAVA_HOME/bin:$PATH"

./mvnw -DskipTests package        # first build downloads Maven + deps (~1–2 min)
java -jar target/kernel-1.0.0-SNAPSHOT.jar
```

The kernel starts on `:8080` (REST) and `:9090` (gRPC). Confirm it's up:
```bash
curl http://localhost:8080/api/health
```

### 3. Frontend (React SQL playground)

```bash
cd frontend
npm install
npm run dev
```

Open **http://localhost:5173**. Vite proxies `/api` to the kernel on `:8080`, so no
CORS setup is needed. Type SQL, press **⌘/Ctrl+Enter** (or click **Run**), and watch
the **Strategy** badge show which scan the RL agent chose and how long it took.

---

## Using the playground

The kernel supports a focused SQL subset:

```sql
CREATE TABLE users (id INT, name VARCHAR(20));
INSERT INTO users VALUES (1, Alice);
INSERT INTO users VALUES (2, Bob);
SELECT * FROM users;
SELECT * FROM users WHERE id = 1;
SELECT * FROM users WHERE id BETWEEN 1 AND 5;
CREATE INDEX ON users (id);
DROP TABLE users;
```

Each `SELECT` result shows:
- a **strategy badge** — `SeqScan`, `IndexScan`, or `CACHED`
  (a `[H]` suffix means the RL optimizer was offline and the heuristic fallback was used),
- **latency** in milliseconds,
- **row count**.

The right-hand **Database** panel shows live tables, columns, indexes, and kernel health.

---

## Verifying each tier independently

**Optimizer** — run the standalone learning demo (no kernel needed). It drives the real
RL agent against a simulated workload and proves it learns the correct policy:
```bash
cd optimizer
./run_demo.sh          # expects: "PASS — optimizer learned the correct policy"
# or the built-in service smoke test:
./run_demo.sh server
```

**Kernel** — after packaging, run SQL directly over REST:
```bash
curl -X POST http://localhost:8080/api/execute \
  -H 'Content-Type: text/plain' \
  --data-binary 'CREATE TABLE t (id INT, name VARCHAR(10)); INSERT INTO t VALUES (1, a); SELECT * FROM t;'
```

**Frontend** — produce a production build:
```bash
cd frontend
npm run build         # outputs dist/
```

---

## Kernel REST API

| Method | Endpoint | Body | Description |
|--------|----------|------|-------------|
| `POST` | `/api/execute` | `text/plain` SQL | Run one or more SQL statements |
| `GET`  | `/api/state` | — | Tables, columns, indexes, data files |
| `GET`  | `/api/health` | — | Health + cache stats |
| `GET`  | `/api/cache/stats` | — | Query cache statistics |
| `POST` | `/api/cache/clear` | — | Clear the query result cache |

`/api/execute` returns:
```json
{
  "ok": true,
  "output": "...REPL-style transcript...",
  "commandCount": 3,
  "results": [
    { "sql": "SELECT * FROM t;", "isQuery": true,
      "strategy": "SeqScan (RL)", "elapsedMs": 0.27, "rowCount": 2,
      "output": "+----+...ASCII table..." }
  ]
}
```

---

## Configuration

**Kernel** (`kernel/src/main/resources/application.properties`):
- `server.port` — REST port (default `8080`)
- `grpc.server.port` — kernel gRPC port (default `9090`)
- `minipostgres.optimizer.grpc.host/port` — where to reach the Python optimizer (default `localhost:50051`)
- `minipostgres.buffer-pool-size` — buffer pool pages (default `100`)
- `minipostgres.cache.ttl-seconds` — query cache TTL (default `5`)

**Optimizer** (env vars, see `optimizer/config.py`):
- `RL_PORT` (HTTP, default `8000`), `RL_GRPC_PORT` (default `50051`)
- `RL_EPSILON_START` (`0.3`), `RL_EPSILON_DECAY` (`0.994`), `RL_UCB_COEFFICIENT` (`1.414`)
- `Q_TABLE_FILE`, `EXPERIENCE_DB` — persistence paths

**Frontend**:
- `VITE_KERNEL_URL` — override the kernel origin at build time (default: dev proxy to `:8080`)

---

## How the RL loop works

1. On a `SELECT` with a predicate, the kernel builds a **query state**
   (row count, range vs. equality, index present) and calls the optimizer's
   `Predict` gRPC method.
2. The optimizer's Q-learning agent returns an action: `0 = SeqScan`, `1 = IndexScan`.
   It explores with ε-greedy + UCB and exploits the best-known action otherwise.
3. The kernel executes that strategy, measures elapsed time, and reports
   `reward = -latency_ms` back via `ReportReward` (fire-and-forget, idempotent).
4. Over many queries the agent learns which scan is faster for each query shape.

Resilience: a **circuit breaker** (50% failure over 10 calls, 30s open) plus retries
protect the kernel. If the optimizer is unreachable, the kernel falls back to a
cost heuristic and tags the strategy with `[H]`.

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `Unable to locate a Java Runtime` | `export JAVA_HOME=/opt/homebrew/opt/openjdk@17` before building/running the kernel |
| Frontend shows "kernel offline" | Make sure the kernel is running on `:8080` (`curl http://localhost:8080/api/health`) |
| Strategy always shows `[H]` | The optimizer isn't running or isn't reachable on `:50051`; start it first |
| `port already in use` | Change `server.port` / `RL_PORT` / Vite `--port`, or stop the conflicting process |
| `npm run dev` proxy errors | Set `VITE_KERNEL_URL` if the kernel isn't on `localhost:8080` |

---

## Notes

- Research paper and patent sources live in `papers/`. They document the design but are
  not required to build or run the system. See `papers/PAPER_NOTES.md` for the paper-restructuring notes.
- Runtime artifacts (logs, `experiences.db`, `q_table.json` backups, `__pycache__`,
  `node_modules`, `dist`, Maven `target/`) are git-ignored and regenerated on demand.
