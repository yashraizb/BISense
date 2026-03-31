# BISense — Backend Low-Level Design

**Stack:** FastAPI · LangChain · DuckDB gRPC Service · SQLite · WebSocket
**Deployment:** Railway (Main Backend) · Railway (DuckDB Service)
**Author:** yashraizb

---

## 1. Component Map

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        Main Backend  (FastAPI · Railway)                        │
│                                                                                 │
│  ┌──────────────────┐   ┌───────────────────┐   ┌──────────────────────────┐   │
│  │   AUTH LAYER     │   │   SESSION LAYER   │   │     PROCESSING LAYER     │   │
│  │                  │   │                   │   │                          │   │
│  │  AuthHandler     │   │  SessionManager   │   │  LangChain               │   │
│  │  [/auth/google]  │   │  [Repository]     │   │  AgentExecutor           │   │
│  │                  │   │                   │   │  [Facade consumer]       │   │
│  │  JWTMiddleware   │   │  APScheduler      │   │                          │   │
│  │  [CoR]           │   │  [cleanup job]    │   │  AgentToolsBuilder       │   │
│  └──────────────────┘   └───────────────────┘   │  [Builder]               │   │
│                                                  │                          │   │
│  ┌──────────────────────────────────────────┐   │  DBStrategy              │   │
│  │   WEBSOCKET + FILE LAYER                 │   │  [Strategy]              │   │
│  │                                          │   │                          │   │
│  │  WebSocketManager   FileHandler          │   │  QueryEventBus           │   │
│  │  [Facade consumer]  [Facade consumer]    │   │  [Observer]              │   │
│  │                                          │   └──────────────────────────┘   │
│  │  FileIngestionAdapter                    │                                   │
│  │  [Adapter + Factory]                     │   ┌──────────────────────────┐   │
│  └──────────────────────────────────────────┘   │   PERSISTENCE LAYER      │   │
│                                                  │                          │   │
│  ┌──────────────────────────────────────────┐   │  SessionRepository       │   │
│  │   FACADE                                 │   │  QueryRepository         │   │
│  │                                          │   │  FileRefRepository       │   │
│  │  BackendServiceFacade                    │   │  [Repository Pattern]    │   │
│  │  ┌──────────────────────────────────┐    │   │                          │   │
│  │  │ get_or_create_session()          │    │   │  SQLite (embedded)       │   │
│  │  │ run_query()  → event stream      │    │   │  sessions | queries      │   │
│  │  │ upload_file()                    │    │   │  steps   | file_refs     │   │
│  │  │ generate_download_url()          │    │   └──────────────────────────┘   │
│  │  └──────────────────────────────────┘    │                                   │
│  └──────────────────────────────────────────┘                                   │
│                                                                                 │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  WSMessageFactory [Factory]  ·  QueryResponseBuilder [Builder]           │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
          │ gRPC (RunSQL / GetSchema / LoadFile)            │ HTTPS
          ▼                                                 ▼
┌──────────────────────────┐                   ┌──────────────────────┐
│  DuckDB gRPC Service     │                   │  External Services   │
│  (Railway)               │                   │                      │
│                          │                   │  Google OAuth 2.0    │
│  grpc.aio Server         │                   │  Anthropic API       │
│  In-Memory DuckDB        │                   │  (claude-sonnet-4-6) │
│  (per-user instance)     │                   └──────────────────────┘
│  60s idle timeout        │
└──────────────────────────┘
```

---

## 2. Class-Level Diagram

Legend:
  «abstract»  = ABC (cannot instantiate directly)
  △───        = inherits from (hollow triangle points to parent)
  ◆───        = owns / composed of (filled diamond)
  ─ ─►        = depends on / uses
  + public    - private    # protected

---

### 2.1 Strategy Pattern — DB Backend

```
                    ┌─────────────────────────────────────────────┐
                    │              «abstract»                      │
                    │              DBStrategy                      │
                    ├─────────────────────────────────────────────┤
                    │ + run_sql(session_id: str,                   │
                    │           sql: str) → QueryResult            │
                    │ + get_schema(session_id: str) → SchemaResult │
                    │ + load_file(session_id: str,                 │
                    │             file_path: str,                  │
                    │             table_name: str) → bool          │
                    └─────────────────────────────────────────────┘
                                          △
                                          │
                    ┌─────────────────────┴──────────────────────┐
                    │                                            │
     ┌──────────────┴──────────────┐       ┌─────────────────────┴───────┐
     │     DuckDBGRPCStrategy      │       │     DuckDBLocalStrategy      │
     ├─────────────────────────────┤       ├─────────────────────────────┤
     │ - channel: grpc.aio.Channel │       │ - conn: duckdb.Connection    │
     │ - stub: DuckDBServiceStub   │       ├─────────────────────────────┤
     ├─────────────────────────────┤       │ + run_sql(...)               │
     │ + run_sql(...)              │       │ + get_schema(...)            │
     │ + get_schema(...)           │       │ + load_file(...)             │
     │ + load_file(...)            │       │   (reads file in-process,    │
     │   (calls LoadFile RPC)      │       │    no network — dev/test)    │
     └─────────────────────────────┘       └─────────────────────────────┘

Context that holds the strategy (dependency-injected, never instantiates directly):

     ┌─────────────────────────────────────────────────────────────────────┐
     │                      BackendServiceFacade                           │
     ├─────────────────────────────────────────────────────────────────────┤
     │ - db: DBStrategy   ◄── injected at startup via DB_BACKEND env flag  │
     │ ...                                                                 │
     └─────────────────────────────────────────────────────────────────────┘
```

---

### 2.2 Repository Pattern — SQLite Data Access

```
                    ┌──────────────────────────────────────────────┐
                    │               «abstract»                      │
                    │               BaseRepository                  │
                    ├──────────────────────────────────────────────┤
                    │ # conn: sqlite3.Connection                    │
                    ├──────────────────────────────────────────────┤
                    │ # execute(sql: str, params: tuple) → Cursor   │
                    │ # fetch_one(sql: str, params) → Row | None    │
                    │ # fetch_all(sql: str, params) → list[Row]     │
                    └──────────────────────────────────────────────┘
                                          △
              ┌───────────────────────────┼───────────────────────────┐
              │                           │                           │
┌─────────────┴────────────┐ ┌───────────┴────────────┐ ┌───────────┴────────────┐
│    SessionRepository     │ │    QueryRepository      │ │   FileRefRepository    │
├──────────────────────────┤ ├────────────────────────┤ ├────────────────────────┤
│ + create(user_id,        │ │ + create(session_id,   │ │ + insert(session_id,   │
│     session_id,          │ │     query_id,          │ │     table_name,        │
│     ttl=30) → Session    │ │     prompt) → Query    │ │     file_path)→FileRef │
│ + get(session_id)        │ │ + update_status(       │ │ + get_by_session(sid)  │
│     → Session | None     │ │     query_id,          │ │     → list[FileRef]    │
│ + touch(session_id)      │ │     status)            │ │ + delete_by_session(   │
│ + expire(session_id)     │ │ + get_with_steps(qid)  │ │     session_id)        │
│ + list_expired()         │ │     → Query+list[Step] │ └────────────────────────┘
│     → list[Session]      │ └────────────────────────┘
└──────────────────────────┘

                    ┌──────────────────────────────┐
                    │       StepRepository          │
                    ├──────────────────────────────┤
                    │ + insert(query_id, step_n,    │
                    │     sql, csv_path) → Step     │
                    │ + get_by_query(query_id)      │
                    │     → list[Step]              │
                    └──────────────────────────────┘

All repositories share one sqlite3.Connection (Singleton — see §2.9).
No raw SQL is written outside of a Repository class.
```

---

### 2.3 Adapter Pattern — File Ingestion

```
                    ┌──────────────────────────────────────────────┐
                    │               «abstract»                      │
                    │           FileIngestionAdapter                │
                    ├──────────────────────────────────────────────┤
                    │ + supported_extensions: list[str]  «classvar» │
                    ├──────────────────────────────────────────────┤
                    │ + ingest(file: UploadFile,                    │
                    │          dest_dir: Path) → IngestedFile       │
                    └──────────────────────────────────────────────┘
                                          △
                    ┌─────────────────────┼──────────────────────┐
                    │                     │                      │
         ┌──────────┴───────┐  ┌──────────┴────────┐  ┌─────────┴────────────┐
         │    CSVAdapter    │  │   ExcelAdapter     │  │   ParquetAdapter     │
         ├──────────────────┤  ├───────────────────┤  ├──────────────────────┤
         │ supported =      │  │ supported =        │  │ supported =          │
         │ [".csv"]         │  │ [".xlsx", ".xls"]  │  │ [".parquet"]         │
         ├──────────────────┤  ├───────────────────┤  │  «future»            │
         │ + ingest(...)    │  │ + ingest(...)      │  ├──────────────────────┤
         │   pd.read_csv()  │  │   pd.read_excel()  │  │ + ingest(...)        │
         │   → IngestedFile │  │   → IngestedFile   │  │   pd.read_parquet()  │
         └──────────────────┘  └───────────────────┘  └──────────────────────┘

Factory that selects the right adapter (see §2.4 for Factory pattern):

                    ┌──────────────────────────────────────────────┐
                    │         FileIngestionAdapterFactory           │
                    ├──────────────────────────────────────────────┤
                    │ - _registry: dict[str, FileIngestionAdapter]  │
                    ├──────────────────────────────────────────────┤
                    │ + register(adapter: FileIngestionAdapter)     │
                    │ + for_file(filename: str)                     │
                    │       → FileIngestionAdapter                  │
                    │     raises UnsupportedFileTypeError           │
                    └──────────────────────────────────────────────┘

FileHandler depends on FileIngestionAdapterFactory, never on concrete adapters:
    FileHandler ─ ─► FileIngestionAdapterFactory ─ ─► FileIngestionAdapter (ABC)
```

---

### 2.4 Factory Pattern — WebSocket Messages

```
All static methods. No inheritance needed — single class owns all message shapes.
Frontend contract is defined entirely here.

                    ┌──────────────────────────────────────────────┐
                    │           WSMessageFactory                    │
                    ├──────────────────────────────────────────────┤
                    │ + «static» status(                            │
                    │       message: str) → dict                    │
                    │   → { type:"status", message }               │
                    │                                               │
                    │ + «static» query_plan(                        │
                    │       steps: list[str]) → dict               │
                    │   → { type:"query_plan", steps }             │
                    │                                               │
                    │ + «static» partial_result(                    │
                    │       step: int, sql: str,                    │
                    │       rows: list,                             │
                    │       csv_path: str) → dict                   │
                    │   → { type:"partial_result", step,           │
                    │        sql, rows, csv_path }                  │
                    │                                               │
                    │ + «static» final_result(                      │
                    │       query_id: str,                          │
                    │       rows: list,                             │
                    │       csv_path: str) → dict                   │
                    │   → { type:"final_result", ... }             │
                    │                                               │
                    │ + «static» error(                             │
                    │       code: str,                              │
                    │       message: str) → dict                    │
                    │   → { type:"error", code, message }          │
                    └──────────────────────────────────────────────┘

Used by WSStreamer (Observer subscriber) — never built ad-hoc in handler code.
```

---

### 2.5 Observer Pattern — Query Step Events

```
Event hierarchy (value objects, not subclassed beyond these):

                    ┌──────────────────────────────────────────────┐
                    │              «abstract»                       │
                    │              QueryEvent                       │
                    ├──────────────────────────────────────────────┤
                    │ + query_id: str                               │
                    │ + occurred_at: datetime                       │
                    └──────────────────────────────────────────────┘
                                          △
              ┌───────────────────────────┼───────────────────────────┐
              │                           │                           │
  ┌───────────┴──────────┐  ┌────────────┴──────────┐  ┌────────────┴──────────┐
  │  QueryPlanReadyEvent │  │  StepCompletedEvent    │  │  QueryFinalizedEvent  │
  ├──────────────────────┤  ├───────────────────────┤  ├───────────────────────┤
  │ + steps: list[str]   │  │ + step_n: int          │  │ + rows: list          │
  └──────────────────────┘  │ + sql: str             │  │ + csv_path: str       │
                             │ + rows: list           │  └───────────────────────┘
  ┌───────────────────────┐  │ + csv_path: str        │
  │   QueryFailedEvent    │  └───────────────────────┘
  ├───────────────────────┤
  │ + code: str           │
  │ + message: str        │
  └───────────────────────┘

Event bus (the Observable):

                    ┌──────────────────────────────────────────────┐
                    │              QueryEventBus                    │
                    ├──────────────────────────────────────────────┤
                    │ - _handlers: dict[type, list[EventHandler]]   │
                    ├──────────────────────────────────────────────┤
                    │ + subscribe(event_type: type,                 │
                    │             handler: EventHandler)            │
                    │ + emit(event: QueryEvent)                     │
                    │     calls all handlers subscribed to          │
                    │     event.__class__                           │
                    └──────────────────────────────────────────────┘
                                ◆           ◆
                                │           │ (owns list of handlers)
                                │           │
           ┌────────────────────┘           └───────────────────────┐
           │                                                        │
           │  EventHandler (ABC — the Observer interface)           │
           │  ┌──────────────────────────────────────────────┐     │
           │  │             «abstract»                        │     │
           │  │             EventHandler                      │     │
           │  ├──────────────────────────────────────────────┤     │
           │  │ + handle(event: QueryEvent)                   │     │
           │  └──────────────────────────────────────────────┘     │
           │                      △                                 │
           │           ┌──────────┴──────────┐                     │
           │           │                     │                     │
  ┌────────┴──────┐  ┌─┴────────────────┐  ┌┴────────────────────┐│
  │  WSStreamer   │  │ SQLiteQueryLogger │  │  MetricsCollector   ││
  ├───────────────┤  ├──────────────────┤  │  «future»           ││
  │ - ws_manager  │  │ - query_repo      │  ├─────────────────────┤│
  │               │  │ - step_repo       │  │ + handle(event)     ││
  │ + handle(e)   │  │ + handle(event)   │  │   log tokens/latency││
  │   sends WS    │  │   writes to       │  └─────────────────────┘│
  │   message via │  │   SQLite          │                          │
  │   WSMessage   │  └──────────────────┘                          │
  │   Factory     │                                                 │
  └───────────────┘                                                 │
           └─────────────────────────────────────────────────────── ┘

AgentExecutor calls only: event_bus.emit(StepCompletedEvent(...))
It has zero imports from ws/, repositories/, or factories/.
```

---

### 2.6 Builder Pattern — Query Response

```
Director: AgentExecutor calls add_step() per LangChain step, set_final() at end.

                    ┌──────────────────────────────────────────────┐
                    │           QueryResponseBuilder                │
                    ├──────────────────────────────────────────────┤
                    │ - _query_id: str                              │
                    │ - _prompt: str                                │
                    │ - _steps: list[StepResult]                   │
                    │ - _final_csv: str | None                      │
                    │ - _status: str                                │
                    ├──────────────────────────────────────────────┤
                    │ + set_query_id(qid: str) → Self              │
                    │ + set_prompt(prompt: str) → Self             │
                    │ + add_step(step_n: int,                       │
                    │            sql: str,                          │
                    │            rows: list,                        │
                    │            csv_path: str) → Self             │
                    │ + set_status(status: str) → Self             │
                    │ + set_final(csv_path: str,                    │
                    │             rows: list) → Self               │
                    │ + build() → QueryResponse                     │
                    │     raises if query_id or prompt not set      │
                    └──────────────────────────────────────────────┘
                                          │ builds
                                          ▼
                    ┌──────────────────────────────────────────────┐
                    │           QueryResponse  «dataclass»          │
                    ├──────────────────────────────────────────────┤
                    │ + query_id: str                               │
                    │ + prompt: str                                 │
                    │ + steps: list[StepResult]                    │
                    │ + final_csv: str | None                       │
                    │ + status: str                                 │
                    └──────────────────────────────────────────────┘
```

---

### 2.7 Facade Pattern — BackendServiceFacade

```
Without Facade: WebSocket handler imports and orchestrates 7+ classes itself.
With Facade:    WebSocket handler calls one class; internal complexity is hidden.

                    ┌─────────────────────────────────────────────────────────────┐
                    │                  BackendServiceFacade                        │
                    ├─────────────────────────────────────────────────────────────┤
                    │ - db: DBStrategy                «Strategy»                  │
                    │ - session_repo: SessionRepository  «Repository»             │
                    │ - query_repo: QueryRepository                               │
                    │ - file_ref_repo: FileRefRepository                          │
                    │ - step_repo: StepRepository                                 │
                    │ - event_bus: QueryEventBus          «Observer»              │
                    │ - adapter_factory:                                           │
                    │     FileIngestionAdapterFactory     «Factory»               │
                    ├─────────────────────────────────────────────────────────────┤
                    │ + get_or_create_session(                                     │
                    │       user_id: str,                                          │
                    │       session_id: str | None) → Session                     │
                    │                                                              │
                    │ + run_query(session: Session,                                │
                    │             prompt: str)                                     │
                    │         → AsyncGenerator[QueryEvent, None]                  │
                    │                                                              │
                    │ + upload_file(session: Session,                              │
                    │               file: UploadFile) → IngestedFile              │
                    │                                                              │
                    │ + generate_download_url(                                     │
                    │       query_id: str,                                         │
                    │       user_id: str) → str                                   │
                    └─────────────────────────────────────────────────────────────┘
                    ◆         ◆        ◆          ◆         ◆         ◆
                    │         │        │          │         │         │
               DBStrategy  Session  Query    FileRef    Step    QueryEventBus
                          Repo     Repo      Repo      Repo

Callers:
    WSManager      ─ ─► BackendServiceFacade   (only import needed)
    api/files.py   ─ ─► BackendServiceFacade
    api/auth.py    ─ ─► BackendServiceFacade
```

---

### 2.8 Chain of Responsibility — WebSocket Request Pipeline

```
Each handler either passes to next or short-circuits with an error message.

                    ┌──────────────────────────────────────────────┐
                    │               «abstract»                      │
                    │               WSRequestHandler                │
                    ├──────────────────────────────────────────────┤
                    │ # next: WSRequestHandler | None               │
                    ├──────────────────────────────────────────────┤
                    │ + set_next(handler: WSRequestHandler) → Self  │
                    │ + handle(ctx: WSRequestContext)               │
                    │     «abstract» — subclass must implement      │
                    └──────────────────────────────────────────────┘
                                          △
              ┌───────────────────────────┼───────────────────────────┐
              │                           │                           │
  ┌───────────┴──────────┐  ┌────────────┴──────────┐  ┌────────────┴──────────┐
  │     JWTValidator     │  │   SessionGatekeeper    │  │   MessageTypeRouter   │
  ├──────────────────────┤  ├───────────────────────┤  ├───────────────────────┤
  │ + handle(ctx)        │  │ + handle(ctx)          │  │ + handle(ctx)         │
  │   decode JWT         │  │   lookup session       │  │   match msg.type      │
  │   invalid →          │  │   expired →            │  │   unknown →           │
  │   error("auth_failed")│  │   error("sess_expired")│  │   error("bad_type")   │
  │   valid →            │  │   active →             │  │   known →             │
  │   next.handle(ctx)   │  │   next.handle(ctx)     │  │   next.handle(ctx)    │
  └──────────────────────┘  └───────────────────────┘  └───────────────────────┘

Chain assembled at startup in main.py:
    JWTValidator → SessionGatekeeper → MessageTypeRouter → Facade.run_query()

WSRequestContext (passed through chain):
    ┌──────────────────────────────────────────────┐
    │          WSRequestContext  «dataclass»         │
    ├──────────────────────────────────────────────┤
    │ + websocket: WebSocket                        │
    │ + raw_message: dict                           │
    │ + user_id: str | None      (set by JWT link)  │
    │ + session: Session | None  (set by Sess link) │
    └──────────────────────────────────────────────┘
```

---

### 2.9 Singleton Pattern — Shared Infrastructure

```
Ensures one instance of expensive resources across the app lifetime.

         ┌─────────────────────────────────────────────────────┐
         │           gRPCChannelSingleton                       │
         ├─────────────────────────────────────────────────────┤
         │ - _instance: grpc.aio.Channel | None   «classvar»   │
         ├─────────────────────────────────────────────────────┤
         │ + «classmethod» get_channel() → grpc.aio.Channel    │
         │     creates channel on first call, reuses after      │
         └─────────────────────────────────────────────────────┘
                    used by: DuckDBGRPCStrategy (injected)

         ┌─────────────────────────────────────────────────────┐
         │           SQLiteConnectionSingleton                  │
         ├─────────────────────────────────────────────────────┤
         │ - _instance: sqlite3.Connection | None  «classvar»  │
         ├─────────────────────────────────────────────────────┤
         │ + «classmethod» get_connection()                     │
         │         → sqlite3.Connection                         │
         │     WAL mode, check_same_thread=False                │
         └─────────────────────────────────────────────────────┘
                    used by: all Repository classes (injected)

         ┌─────────────────────────────────────────────────────┐
         │           APSchedulerSingleton                       │
         ├─────────────────────────────────────────────────────┤
         │ - _instance: AsyncIOScheduler | None   «classvar»   │
         ├─────────────────────────────────────────────────────┤
         │ + «classmethod» get_scheduler()                      │
         │         → AsyncIOScheduler                           │
         │     started once in FastAPI lifespan startup         │
         └─────────────────────────────────────────────────────┘
                    used by: cleanup.py (session expiry job)
```

---

### 2.10 Full Dependency Graph

```
                         main.py (wires everything at startup)
                               │
               ┌───────────────┼───────────────────┐
               │               │                   │
               ▼               ▼                   ▼
    SQLiteConnectionSingleton  gRPCChannelSingleton  APSchedulerSingleton
               │               │
               ▼               ▼
      BaseRepository     DuckDBGRPCStrategy
      (all 4 repos)         (DBStrategy)
               │               │
               └───────┬───────┘
                       │
                       ▼
             BackendServiceFacade ◄── QueryEventBus ◄── [WSStreamer, SQLiteLogger]
             (Facade)                 (Observer)
                       │
               ┌───────┴───────────────────────────┐
               │               │                   │
               ▼               ▼                   ▼
           WSManager      api/files.py          api/auth.py
               │
               ▼
     WSRequestHandler chain
     (CoR: JWT → Session → Router)
               │
               ▼
     AgentExecutor + AgentToolsBuilder
     (tools wrap DBStrategy methods)
```

---

## 3. Directory Structure

```
backend/
├── app/
│   ├── main.py                        # FastAPI app, lifespan, DI wiring
│   │
│   ├── api/
│   │   ├── auth.py                    # GET /auth/google, /auth/callback
│   │   ├── files.py                   # POST /files/upload, GET /files/download
│   │   └── ws.py                      # WS /ws/{session_id}  ← thin, calls Facade
│   │
│   ├── core/
│   │   ├── config.py                  # env vars, DB_BACKEND flag
│   │   ├── jwt.py                     # issue + validate JWT
│   │   ├── middleware.py              # JWTMiddleware [CoR link 1]
│   │   └── singletons.py             # gRPC channel, SQLite conn, scheduler
│   │
│   ├── facade/
│   │   └── backend_service.py         # BackendServiceFacade [Facade]
│   │
│   ├── strategy/
│   │   ├── base.py                    # DBStrategy ABC
│   │   ├── duckdb_grpc.py             # DuckDBGRPCStrategy
│   │   └── duckdb_local.py            # DuckDBLocalStrategy (dev/test)
│   │
│   ├── repositories/
│   │   ├── session.py                 # SessionRepository [Repository]
│   │   ├── query.py                   # QueryRepository
│   │   ├── step.py                    # StepRepository
│   │   └── file_ref.py               # FileRefRepository
│   │
│   ├── adapters/
│   │   ├── base.py                    # FileIngestionAdapter ABC [Adapter]
│   │   ├── csv_adapter.py             # CSVAdapter
│   │   ├── excel_adapter.py           # ExcelAdapter
│   │   └── factory.py                 # FileIngestionAdapterFactory [Factory]
│   │
│   ├── agent/
│   │   ├── executor.py                # LangChain AgentExecutor setup
│   │   ├── tools_builder.py           # AgentToolsBuilder [Builder]
│   │   └── tools/
│   │       ├── get_schema.py          # Tool: wraps DBStrategy.get_schema()
│   │       └── run_sql.py             # Tool: wraps DBStrategy.run_sql()
│   │
│   ├── events/
│   │   ├── bus.py                     # QueryEventBus [Observer]
│   │   ├── types.py                   # QueryEvent subclasses
│   │   ├── ws_streamer.py             # subscriber → WS messages
│   │   └── sqlite_logger.py          # subscriber → SQLite writes
│   │
│   ├── factories/
│   │   └── ws_message.py             # WSMessageFactory [Factory]
│   │
│   ├── builders/
│   │   └── query_response.py         # QueryResponseBuilder [Builder]
│   │
│   ├── ws/
│   │   ├── manager.py                 # WebSocketManager
│   │   └── pipeline.py               # CoR links [2]-[4]
│   │
│   └── scheduler/
│       └── cleanup.py                 # APScheduler job → SessionRepository.list_expired()
│
├── proto/
│   └── duckdb_service.proto           # RunSQL / GetSchema / LoadFile RPCs
│
├── tests/
│   ├── test_strategy_local.py         # uses DuckDBLocalStrategy, no gRPC needed
│   ├── test_repositories.py           # in-memory SQLite
│   ├── test_adapters.py               # CSV/Excel ingestion
│   ├── test_ws_pipeline.py            # CoR chain unit tests
│   └── test_facade.py                 # integration: facade + local strategy
│
└── requirements.txt
```

---

## 4. Flow Walkthroughs

### 4.1 Auth Flow

```
Frontend                 AuthHandler              JWTMiddleware      Google OAuth
    │                        │                        │                   │
    │── GET /auth/google ────►│                        │                   │
    │                        │── redirect ────────────────────────────────►│
    │                        │◄─ id_token ─────────────────────────────────│
    │                        │── verify id_token ─────────────────────────►│
    │                        │◄─ user info ────────────────────────────────│
    │                        │── jwt.issue(user_id) ──►│                   │
    │◄── { jwt_token } ──────│                        │                   │
    │                        │                        │                   │
    │  (all subsequent requests: Authorization: Bearer <jwt>)              │
    │─────────────────────────────────────────────────►│                   │
    │                        │◄── user_id extracted ──│                   │
```

---

### 4.2 File Upload Flow

```
Frontend          FileHandler         FileIngestionAdapterFactory
    │                 │                          │
    │─ POST /files/upload (multipart) ──────────►│
    │  [JWT validated by middleware]              │
    │                 │── for_file("data.csv") ──►│
    │                 │◄── CSVAdapter ────────────│
    │                 │
    │                 │── adapter.ingest(file, dest_dir)
    │                 │   saves → uploads/{user_id}/data/data.csv
    │                 │   returns IngestedFile(table_name, path, rows, cols)
    │                 │
    │                 │── DBStrategy.load_file(session_id, path, table_name)
    │                 │   [DuckDBGRPCStrategy → LoadFile RPC → DuckDB svc]
    │                 │
    │                 │── FileRefRepository.insert(session_id, table_name, path)
    │                 │── QueryEventBus (no event here, simple REST response)
    │◄── 200 { table_name, row_count, columns } ─│
```

---

### 4.3 Query Execution Flow (hero flow)

```
Frontend      WSManager      Pipeline (CoR)     Facade         AgentExecutor     EventBus
    │              │                │               │                │               │
    │─ WS connect /ws/{sid}?token ─►│               │               │               │
    │              │─[1] validate JWT               │               │               │
    │              │─[2] check session ────────────►│               │               │
    │              │     Facade.get_or_create_session()             │               │
    │              │     → if resumed: reload FileRefs via DBStrategy               │
    │              │─[3] route message type         │               │               │
    │              │                                │               │               │
    │─ { type:"query", prompt } ───────────────────►│               │               │
    │              │                                │               │               │
    │              │        Facade.run_query(session, prompt)        │               │
    │              │                                │               │               │
    │              │                   QueryRepository.create(...)  │               │
    │              │                   AgentToolsBuilder.build(...)  │               │
    │              │                                │── astream(prompt, tools) ────►│
    │              │                                │               │               │
    │              │                                │    [Agent calls get_schema()] │
    │              │                                │    DBStrategy.get_schema() ───►[DuckDB svc]
    │              │                                │               │               │
    │              │                                │    [Agent generates SQL]      │
    │              │                                │    DBStrategy.run_sql() ──────►[DuckDB svc]
    │              │                                │               │◄── rows ──────│
    │              │                                │               │               │
    │              │                         emit StepCompletedEvent────────────────►│
    │              │                                │               │    WSStreamer: │
    │◄─ { type:"partial_result", step:1, rows } ───◄│◄──────────────│───────────────│
    │              │                                │               │    SQLiteLogger:
    │              │                                │               │    StepRepo.insert()
    │              │                                │               │               │
    │              │           [repeat per step]    │               │               │
    │              │                                │               │               │
    │              │                         emit QueryFinalizedEvent───────────────►│
    │◄─ { type:"final_result", query_id, rows } ───◄│◄──────────────│───────────────│
    │              │                                │    QueryResponseBuilder.build()│
    │              │                                │    QueryRepo.update_status(done)
```

---

### 4.4 Disconnect / Reconnect Flow

```
Frontend          WSManager          SessionRepository
    │                 │                      │
    │── WS close ────►│                      │
    │                 │  session stays alive  │
    │                 │  (TTL 30 min timer)   │
    │                 │                      │
    │── WS connect (same session_id) ────────►│
    │                 │── get(session_id) ───►│ → Session (not expired)
    │                 │── touch(session_id) ──►│ → resets last_active_at
    │                 │                      │
    │                 │  Facade: reload FileRefs → DBStrategy.load_file() per ref
    │                 │  (eager reload: DuckDB in-memory state rebuilt)
    │◄── WS ready ────│
```

---

### 4.5 Download Flow

```
Frontend            FileHandler           StepRepository
    │                    │                      │
    │── GET /files/download/{query_id} ─────────►│
    │   [JWT middleware: user_id extracted]       │
    │                    │── get_by_query(qid) ──►│ → list[Step]
    │                    │   verify step.user_id == requesting user_id
    │                    │
    │                    │── sign(file_path, user_id, ttl=60s)
    │                    │   HMAC token, not persisted
    │◄── 200 { download_url } ───────────────────│
    │
    │── GET {download_url} [direct to FastAPI]
    │   [middleware validates HMAC token + expiry]
    │◄── 200 CSV file stream
```

---

### 4.6 Session Cleanup Flow

```
APScheduler (every 5 min)
    │
    │── SessionRepository.list_expired()   → list[Session] past 30 min TTL
    │
    │   for each expired session:
    │   ├── WSManager.disconnect(session_id)       if WS still open
    │   ├── FileRefRepository.delete_by_session()  update SQLite record
    │   └── SessionRepository.expire(session_id)   mark expired
    │
    │   Note: files on disk are NOT deleted (manual cleanup only, MVP)
    │   Note: DuckDB in-memory state auto-drops after 60s idle on DuckDB svc
```

---

## 5. SQLite Schema

```sql
CREATE TABLE sessions (
    id            TEXT PRIMARY KEY,       -- UUID
    user_id       TEXT NOT NULL,
    created_at    INTEGER NOT NULL,       -- unix timestamp
    last_active_at INTEGER NOT NULL,
    ttl_minutes   INTEGER DEFAULT 30,
    status        TEXT DEFAULT 'active'   -- active | expired
);

CREATE TABLE queries (
    id            TEXT PRIMARY KEY,       -- UUID
    session_id    TEXT NOT NULL REFERENCES sessions(id),
    prompt        TEXT NOT NULL,
    status        TEXT DEFAULT 'planning', -- planning | running | done | error
    created_at    INTEGER NOT NULL,
    completed_at  INTEGER
);

CREATE TABLE steps (
    id            TEXT PRIMARY KEY,
    query_id      TEXT NOT NULL REFERENCES queries(id),
    step_n        INTEGER NOT NULL,
    sql           TEXT NOT NULL,
    csv_path      TEXT,                   -- results/{query_id}/step_{n}.csv
    row_count     INTEGER,
    created_at    INTEGER NOT NULL
);

CREATE TABLE file_refs (
    id            TEXT PRIMARY KEY,
    session_id    TEXT NOT NULL REFERENCES sessions(id),
    table_name    TEXT NOT NULL,
    file_path     TEXT NOT NULL,          -- uploads/{user_id}/data/{filename}
    uploaded_at   INTEGER NOT NULL
);
```

---

## 6. gRPC Proto (outline)

```protobuf
service DuckDBService {
    rpc LoadFile   (LoadFileRequest)   returns (LoadFileResponse);
    rpc RunSQL     (RunSQLRequest)     returns (RunSQLResponse);
    rpc GetSchema  (GetSchemaRequest)  returns (GetSchemaResponse);
}

message LoadFileRequest  { string session_id = 1; string file_path = 2; string table_name = 3; }
message LoadFileResponse { bool success = 1; string error = 2; }

message RunSQLRequest    { string session_id = 1; string sql = 2; }
message RunSQLResponse   { repeated Row rows = 1; string csv_path = 2; string error = 3; }

message GetSchemaRequest  { string session_id = 1; }
message GetSchemaResponse { repeated TableSchema tables = 1; }

message TableSchema { string table_name = 1; repeated ColumnSchema columns = 2; }
message ColumnSchema { string name = 1; string dtype = 2; }
message Row          { map<string, string> fields = 1; }
```

---

## 7. Key Constraints & Decisions

| Concern | Decision | Reason |
|---|---|---|
| Workers | Single Uvicorn worker | In-memory WS state; scale via Railway replicas later |
| DB backend selection | `DB_BACKEND` env var → Strategy factory | Zero code change to swap local ↔ gRPC |
| Session storage | SQLite (embedded) | No Redis dependency in MVP |
| File deletion | Manual only (MVP) | Files persist across sessions; cleanup deferred |
| gRPC channel | Singleton | One persistent connection; not one per request |
| DuckDB idle | 60s timeout on DuckDB svc side | Free up memory for inactive users |
| Download auth | HMAC signed URL (low TTL, no DB) | Simple, stateless, no extra table |
| SQL safety | Banned ops: DROP / DELETE / ALTER / TRUNCATE | Enforced in RunSQL RPC before execution |
| Test isolation | DuckDBLocalStrategy in all tests | No gRPC service needed to run test suite |
