# BISense Backend Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the full BISense backend — FastAPI main service + DuckDB gRPC service — as designed in `docs/backend_lld.md`.

**Architecture:** Two Railway services communicate over gRPC. The main FastAPI backend handles auth, WebSocket chat, file upload, and LangChain orchestration. The DuckDB gRPC service manages per-user in-memory DuckDB instances and executes SQL with guardrails.

**Tech Stack:** FastAPI · LangChain · grpcio · DuckDB · SQLite · APScheduler · Google OAuth 2.0 · Anthropic claude-sonnet-4-6

---

## Implementation Order (dependency-safe)

```
DuckDB gRPC Service (independent)
  → Issue A: Proto + server scaffold
  → Issue B: LoadFile RPC + instance manager
  → Issue C: RunSQL + GetSchema RPCs
  → Issue D: ListTables + DropTable + ExportTable + idle timeout

Main Backend (builds in this order)
  → Issue 1:  Core infrastructure (config, singletons, logging, JWT, SQLite schema)
  → Issue 2:  Persistence layer (BaseRepository + 4 concrete repos)
  → Issue 3:  DB Strategy layer (ABC + Local + gRPC strategies)
  → Issue 4:  File ingestion adapters + factory
  → Issue 5:  Events system (bus, types, WSMessageFactory, WSStreamer, SQLiteLogger)
  → Issue 6:  Auth layer (Google OAuth + JWT middleware)
  → Issue 7:  WebSocket pipeline (CoR chain + WebSocketManager)
  → Issue 8:  LangChain agent (AgentToolsBuilder + tools + QueryResponseBuilder + AgentExecutor)
  → Issue 9:  BackendServiceFacade
  → Issue 10: API routes + main.py wiring
  → Issue 11: Session cleanup scheduler
  → Issue 12: Test suite
```

## GitHub Issues Index

Issues are tracked in the BISense GitHub repo. Each issue contains full implementation steps.

| # | Title | Depends On |
|---|-------|------------|
| A | DuckDB gRPC Service: Proto + server scaffold | — |
| B | DuckDB gRPC Service: LoadFile RPC + instance manager | A |
| C | DuckDB gRPC Service: RunSQL + GetSchema RPCs | B |
| D | DuckDB gRPC Service: ListTables, DropTable, ExportTable, idle timeout | C |
| 1 | Backend: Core infrastructure | — |
| 2 | Backend: Persistence layer | 1 |
| 3 | Backend: DB Strategy layer | A (for gRPC proto stubs) |
| 4 | Backend: File ingestion adapters | 1 |
| 5 | Backend: Events system | 4 |
| 6 | Backend: Auth layer | 1 |
| 7 | Backend: WebSocket pipeline | 6 |
| 8 | Backend: LangChain agent | 3, 5 |
| 9 | Backend: BackendServiceFacade | 2, 3, 4, 5, 8 |
| 10 | Backend: API routes + main.py | 6, 7, 9 |
| 11 | Backend: Session cleanup scheduler | 2, 7 |
| 12 | Backend: Test suite | 2, 3, 4, 7, 9 |

## Key Design Decisions (from LLD)

- `DB_BACKEND` env var selects `local` (DuckDBLocalStrategy, used in tests) or `grpc` (DuckDBGRPCStrategy, production)
- All repositories share a single `sqlite3.Connection` (Singleton)
- gRPC channel is a Singleton — one persistent connection, not one per request
- No raw SQL outside Repository classes
- SQL guardrails enforced in DuckDB service RunSQL RPC: DROP / DELETE / ALTER / TRUNCATE are banned
- Top 10 rows returned in chat; full download via HMAC-signed URL
- WebSocket CoR chain: JWTValidator → SessionGatekeeper → MessageTypeRouter
- AgentExecutor emits events via QueryEventBus — zero imports from ws/ or repositories/
- Files on disk NOT deleted on session expiry (MVP)
