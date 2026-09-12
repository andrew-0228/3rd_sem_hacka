# Ruled by Secrecy

A predictive, physics-aware multi-agent resilience and flexibility exchange for commercial and critical-infrastructure microgrids. The system forecasts grid stress before it happens, lets on-site assets (batteries, EV chargers, flexible loads) bid to relieve it through a simulated market, and validates every proposed action against a real AC power-flow model before it's allowed to execute.

Built in 24 hours across three services by a three-person team.

## Architecture

```
                     ┌──────────────────────┐
                     │     orchestrator     │  Person 3
                     │  (drives the loop,   │
                     │  dashboard, chain)   │
                     └───────────┬──────────┘
                     ┌───────────┴─────────────────┐
                     ▼                             ▼
        ┌────────────────────────┐    ┌────────────────────────┐
        │      grid_engine       │    │     agent_engines      │
        │  Person 1 / port 8001  │    │  Person 2 / port 8002  │
        │                        │    │                        │
        │  pandapower AC power   │    │   offer generation,    │
        │   flow, forecasting,   │    │    market clearing,    │
        │   validation, faults   │    │       settlement       │
        └────────────────────────┘    └────────────────────────┘
```

**Canonical call sequence** (see `INTEGRATION_GUIDELINES.md` for the full, current version):

1. `grid_engine` → `GET /grid/state` — read or predict grid stress
2. `agent_engines` → `POST /agents/offers` — assets submit flexibility offers
3. `agent_engines` → `POST /agents/clear_market` — offers matched into proposed actions
4. `grid_engine` → `POST /grid/validate` — every proposed action checked against a real re-solved power flow
5. Orchestrator keeps only `feasible` results, substituting `adjusted_kw_amount` where set
6. `agent_engines` → `POST /agents/settle` — feasible actions settled into trades
7. Orchestrator builds a `BlockchainTransaction` from the resulting `Trade` list

Separately, for demos: `grid_engine` exposes `POST /grid/fault` to inject a simulated event (line fault, grid outage, battery failure, solar drop, demand spike, feeder overload) with real topology-based islanding, and `POST /grid/fault/clear(_all)` to remove it.

## Services

| Service | Owner | Port | Docs |
|---|---|---|---|
| `grid_engine` | Person 1 | 8001 | [`grid_engine/INTEGRATION.md`](grid_engine/INTEGRATION.md) |
| `agent_engines` | Person 2 | 8002 | `agent_engines/INTEGRATION.md` |
| `orchestrator` | Person 3 | — | *(in progress)* |

Start here: [`INTEGRATION_GUIDELINES.md`](INTEGRATION_GUIDELINES.md) — the single cross-service source of truth for how the three services fit together, what's verified working, and what's still open.

## Grid engine

A 5-bus representative campus microgrid modeled in [pandapower](https://www.pandapower.org/), with real AC power flow (Newton-Raphson) behind every result — nothing here is simulated or faked.

```
    UTILITY GRID (11 kV)
          |
      TRANSFORMER  (11/0.415 kV, 1 MVA)
          |
    CAMPUS MAIN BUS (0.415 kV)
      /        |         \
   F1 (x3)   F2 (x1)    F3 (x1)
    |          |            \
 HOSPITAL   ACADEMIC      FACILITY
    |        |    |
  BESS    SOLAR   EV
```

Delivered milestones:

- **M1** — static 5-bus network + single AC power-flow snapshot
- **M2** — synthetic time-series load/solar profiles
- **M3** — future-state forecasting
- **M4** — predicted-violation and required-relief detection
- **M5** — proposed-action validation against a real re-solved power flow
- **M6** — simulated fault injection with real topology-based islanding (`networkx` connected-components analysis, not a lookup table)

98/98 tests passing. Full endpoint documentation, verified example payloads, and known gaps are in [`grid_engine/INTEGRATION.md`](grid_engine/INTEGRATION.md).

## Running it locally

This section covers macOS/Linux (bash) and Windows (PowerShell) side by
side — pick the one that matches your machine. If something doesn't work,
check the **Troubleshooting** section right after this one before assuming
it's a new bug; every issue listed there was hit for real while setting
this up on Windows and has a one-line fix.

### 1. Clone and install

**bash (macOS/Linux):**

```bash
git clone https://github.com/andrew-0228/3rd_sem_hacka.git
cd 3rd_sem_hacka

python3 -m venv .venv
source .venv/bin/activate

# grid_engine's own deps (pandapower, pytest, fastapi, uvicorn[standard], httpx)
pip install -r requirements.txt
# agent_engines' own deps (fastapi, uvicorn[standard], httpx, pydantic, pytest)
pip install -r agent_engines/requirements.txt
```

**PowerShell (Windows):**

```powershell
git clone https://github.com/andrew-0228/3rd_sem_hacka.git
cd 3rd_sem_hacka

python -m venv .venv
.venv\Scripts\Activate.ps1
# if that errors with "running scripts is disabled": Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass

pip install -r requirements.txt
pip install -r agent_engines\requirements.txt
```

`orchestrator/` has no `requirements.txt` of its own — it only needs
`fastapi`, `uvicorn[standard]`, and `httpx`, all pulled in by the two
installs above. The `[standard]` extra matters, not just style: bare
`uvicorn` doesn't include a WebSocket implementation at all, and the
dashboard's entire live-update mechanism is a WebSocket
(`/ws/dashboard`) — see Troubleshooting below if you ever see the
dashboard stuck on "reconnecting…".

If you already have a `.venv` from before and just pulled this update,
re-run the two `pip install` commands with `--upgrade` so the newly
added dependencies (`httpx`, `uvicorn[standard]`) actually land:

```bash
pip install -r requirements.txt --upgrade
pip install -r agent_engines/requirements.txt --upgrade
```

### 2. Start all three services (three separate terminals)

Ports matter and are easy to swap by accident: **grid_engine is 8001,
agent_engines is 8002, orchestrator is 8000.**

**bash:**

```bash
# terminal 1 — grid_engine, port 8001
source .venv/bin/activate
uvicorn grid_engine.api:app --port 8001 --reload
curl http://localhost:8001/health   # -> {"status": "ok"}

# terminal 2 — agent_engines, port 8002
source .venv/bin/activate
uvicorn agent_engines.main:app --port 8002 --reload
curl http://localhost:8002/health   # -> {"status": "ok"}

# terminal 3 — orchestrator, port 8000, start last
source .venv/bin/activate
# GRID_SOURCE / AGENT_SOURCE default to "mock" if unset, which runs the
# orchestrator against its own built-in fake data instead of the two real
# services above — set both to "live" to actually wire them together:
GRID_SOURCE=live AGENT_SOURCE=live uvicorn orchestrator.main:app --port 8000 --reload
curl http://localhost:8000/health
# -> {"orchestrator": "ok", "tick_count": N, "grid_source": "live", "agent_source": "live"}
```

**PowerShell:**

Each terminal needs to `cd` back into the repo root you cloned in step 1
(the same folder, every time — that's `3rd_sem_hacka` unless you renamed
it) before activating the venv:

```powershell
# terminal 1 — grid_engine, port 8001
cd 3rd_sem_hacka
.venv\Scripts\Activate.ps1
uvicorn grid_engine.api:app --port 8001 --reload

# terminal 2 — agent_engines, port 8002 (NOT 8001)
cd 3rd_sem_hacka
.venv\Scripts\Activate.ps1
uvicorn agent_engines.main:app --port 8002 --reload

# terminal 3 — orchestrator, port 8000, start last
cd 3rd_sem_hacka
.venv\Scripts\Activate.ps1
$env:GRID_SOURCE="live"
$env:AGENT_SOURCE="live"
uvicorn orchestrator.main:app --port 8000 --reload
```

If you're not sure where you cloned it, `cd` to wherever you ran `git
clone` in step 1 first, then into that folder — every command below
assumes you're already inside the repo root, not any absolute path.

`$env:GRID_SOURCE="live"` only applies to the current terminal/session —
set it every time you open a fresh terminal to start the orchestrator in.
Also note PowerShell does **not** support the bash-style
`GRID_SOURCE=live uvicorn ...` inline-prefix syntax — it silently doesn't
set the variable, and the orchestrator quietly falls back to mock. Check
its startup log line (`grid=live, agents=live`, not `grid=mock,
agents=mock`) to confirm it actually took.

Order matters: start `grid_engine` and `agent_engines` first — the
orchestrator's first tick fires a couple of seconds after startup and
will just log a failed-call event (not crash) if it can't reach them yet.

### 3. Open the dashboard

With all three running, open **http://localhost:8000** in a browser —
the orchestrator serves its live dashboard there (`orchestrator/static/index.html`),
polling `/state/snapshot` and `/ws/dashboard` for the agent activity feed,
trades, and blockchain ledger as they happen in real time.

To manually trigger a demo event instead of waiting for the campus network's
own overloaded feeders to trip a market:

```bash
curl -X POST http://localhost:8000/scenario/line_fault
# other options: solar_drop, demand_spike, feeder_overload, battery_failure, grid_outage
curl -X POST http://localhost:8000/scenario/reset   # clear it
```

### 4. Run the test suites

```bash
pytest grid_engine/tests/ agent_engines/tests/ -q   # 126 tests, no server needed
```

`orchestrator/` doesn't have its own automated test suite yet (see
`INTEGRATION_GUIDELINES.md`) — the steps above are how to verify it by
actually running it.

See each service's own `INTEGRATION.md` for its full endpoint reference, verified example payloads, and known gaps.

## Troubleshooting

Every item below was an actual failure hit while setting this up (mostly
on Windows) — not hypothetical. Check here before assuming something's
newly broken.

**Dashboard stuck on "reconnecting…" forever, every panel empty
(Network Topology, System Status, Blockchain, Agent Activity all
blank).**
Open the browser's DevTools console (F12 → Console). If you see repeated
`WebSocket connection to 'ws://localhost:8000/ws/dashboard' failed`, and
the orchestrator's own terminal is logging `WARNING: No supported
WebSocket library detected. Please use "pip install 'uvicorn[standard]'"`
followed by `404 Not Found` for every `GET /ws/dashboard` — that's the
whole story. Plain `uvicorn` does not depend on `websockets` or
`wsproto` at all, so it has no protocol handler to upgrade the
connection; regular HTTP (health checks, the static dashboard file
itself) works fine, only the WebSocket fails, silently and forever.
Fix: stop the orchestrator, run
`pip install -r requirements.txt --upgrade` (this repo's `requirements.txt`
now specifies `uvicorn[standard]`), then restart it. Confirm with:
```bash
python -c "import websockets; print(websockets.__version__)"
```
If that import fails, the install didn't take — you're probably running
a different Python/venv than the one you just installed into.

**Orchestrator's startup log says `grid=mock, agents=mock` even though
you meant to run live.**
The `GRID_SOURCE`/`AGENT_SOURCE` env vars didn't reach the process. Two
common causes: you're in PowerShell and used the bash-style
`GRID_SOURCE=live AGENT_SOURCE=live uvicorn ...` inline prefix, which
PowerShell doesn't support at all (it won't error, it just silently
doesn't set anything) — use `$env:GRID_SOURCE="live"` on its own line
instead, before the `uvicorn` command, in the same terminal session. Or
you set the vars in one terminal and then started the orchestrator in a
*different* terminal — env vars set with `$env:`/`export` only apply to
that one shell session.

**`ModuleNotFoundError: No module named 'httpx'` when starting the
orchestrator.**
`orchestrator/` has no `requirements.txt` of its own; it borrows `httpx`
from the root `requirements.txt`. If you installed only an older copy of
that file (before `httpx` was added to it), or only ran
`pip install -r agent_engines/requirements.txt`, you'll hit this. Fix:
`pip install -r requirements.txt --upgrade` from the repo root, into the
same venv/interpreter the orchestrator actually runs from.

**`agent_engines` or `grid_engine` won't respond, or the orchestrator's
event log shows "Failed to fetch grid state" / "Failed to fetch
offers".**
Almost always a port mix-up — `grid_engine` must be `--port 8001` and
`agent_engines` must be `--port 8002`, easy to swap when copy-pasting
across terminals. Check each service's own terminal for
`Uvicorn running on http://127.0.0.1:PORT` and confirm it matches, and
`curl http://localhost:8001/health` / `:8002/health` both return
`{"status": "ok"}` before starting the orchestrator.

**Dashboard loads (title, buttons, layout all visible) but nothing ever
updates, and it's not even showing "reconnecting…".**
You likely opened `orchestrator/static/index.html` directly from the
file system (double-clicked it, or opened it via `file:///D:/...`)
instead of through the running server. The page's own JavaScript builds
its WebSocket URL from `location.host`, which is empty under `file://`,
so it can't even attempt a real connection. Always reach the dashboard
by typing `http://localhost:8000` into the address bar with all three
services already running — never open the HTML file itself.

**PowerShell: `.venv\Scripts\Activate.ps1` fails with "cannot be loaded
because running scripts is disabled on this system" (`PSSecurityException`
/ `UnauthorizedAccess`), and then `uvicorn` is "not recognized as the name
of a cmdlet, function, script file, or operable program."** The second
error is just a consequence of the first — activation never ran, so the
venv's `Scripts` folder never got added to `PATH` for that session, and
plain `uvicorn` doesn't resolve to anything. Windows' default execution
policy blocks running local `.ps1` scripts at all, even ones that came
with your own venv. Two fixes, either works:
- Relax it for just this window (no admin rights needed):
  ```powershell
  Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
  .venv\Scripts\Activate.ps1
  ```
  `-Scope Process` only affects the current PowerShell process and resets
  the moment you close it — it doesn't change any system-wide setting.
- Or skip activation entirely and call `uvicorn` through the venv's own
  interpreter directly, in each terminal:
  ```powershell
  .venv\Scripts\python.exe -m uvicorn grid_engine.api:app --port 8001 --reload
  .venv\Scripts\python.exe -m uvicorn agent_engines.main:app --port 8002 --reload
  .venv\Scripts\python.exe -m uvicorn orchestrator.main:app --port 8000 --reload
  ```
  (set `$env:GRID_SOURCE="live"` / `$env:AGENT_SOURCE="live"` first, in the
  orchestrator's terminal, same as usual) — this never touches the
  execution policy at all.

**Two copies of the integration guidelines** (`INTEGRATION_GUIDELINES.md`
and a stray extensionless `integration_guidelines` at the repo root) —
they've drifted apart. `INTEGRATION_GUIDELINES.md` is the current one;
ignore or delete the other.

## Known gaps

Tracked in detail in `INTEGRATION_GUIDELINES.md`.

- **Resolved:** `POST /agents/settle` and `POST /agents/reserve` were previously documented but not implemented. Both are now live in `agent_engines` (see its `CHANGELOG.md`), and the full canonical sequence — steps 1–7 above, `grid_engine` and `agent_engines` running as real services, orchestrator talking to both over live HTTP (`GRID_SOURCE=live AGENT_SOURCE=live`) — has been re-run end-to-end with real trades settling and confirming on the orchestrator's blockchain ledger. The demo fault-injection scenarios (`POST /scenario/{name}` → real topology-based islanding) were verified the same way.
- `shared/contracts.py` was missing `ClearFaultRequest` and `ValidateActionsRequest`, and the entire `orchestrator/` package was importing from a nonexistent `schemas.contracts` module instead of `shared.contracts` — both fixed; see `INTEGRATION_GUIDELINES.md`'s contract-changes log.
- Two near-duplicate copies of the integration guidelines exist at the repo root: `INTEGRATION_GUIDELINES.md` (canonical, linked from this README) and a stray extensionless `integration_guidelines` file that has drifted out of sync with it. Worth deleting the stray copy in a follow-up so the "one source of truth" rule this file itself asks for actually holds.

## Repo layout

```
grid_engine/                 Person 1 — power-flow simulation, forecasting, validation, faults, API
agent_engines/                Person 2 — offers, market clearing, settlement
orchestrator/                    Person 3 — drives the full loop, dashboard, blockchain record (structure TBD)
shared/                          contracts.py — pydantic request/response shapes shared by all three services
INTEGRATION_GUIDELINES.md        cross-service source of truth — read this first
```

Each service currently lives on its own branch (`grid-infra`, `agent-engine`, and the orchestrator's) pending a merge into `main`.
