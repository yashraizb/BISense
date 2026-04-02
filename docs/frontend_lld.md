# BISense — Frontend Low-Level Design

**Stack:** React 18 · TypeScript · Zustand · WebSocket · Recharts · Tailwind CSS · Vite · Vercel
**Scope:** GitHub Issue #3 — Component architecture, State management, WebSocket integration, File upload flow, Chat UI, DataTable & Chart design
**Author:** yashraizb

---

## 1. Page / Route Structure

```
/                  → redirect to /chat (if authenticated) or /login
/login             → LoginPage   — Google OAuth entry point
/auth/callback     → AuthCallbackPage — handles redirect from GET /auth/google
/chat              → ChatPage    — full app shell (protected route)
```

All routes below `/chat` are protected. If no JWT is found in `localStorage`, the router redirects to `/login`.

### Route Component Map

```
App
├── <RouterProvider>
│   ├── / (index)       → <Navigate to="/chat" />   (or /login if !authed)
│   ├── /login          → <LoginPage />
│   ├── /auth/callback  → <AuthCallbackPage />
│   └── /chat           → <ProtectedRoute> → <ChatPage />
```

---

## 2. Component Tree

### 2.1 Full Tree

```
<App>
└── <RouterProvider>
    ├── <LoginPage>
    │   └── <GoogleSignInButton />           — renders "Sign in with Google" button
    │
    ├── <AuthCallbackPage>                   — reads ?token= from URL, stores JWT,
    │                                          redirects to /chat
    │
    └── <ChatPage>                           — main app shell, full-viewport
        ├── <Sidebar>
        │   ├── <AppLogo />
        │   ├── <FilePanel>
        │   │   ├── <FileUploadZone />       — drag-drop + file picker
        │   │   └── <UploadedFileList>
        │   │       └── <FileChip />         — one chip per uploaded table
        │   └── <SessionStatus />            — "Session active · 28 min left"
        │
        └── <ChatPane>
            ├── <ChatWindow>                 — scrollable message list
            │   └── <ChatMessage />          — one per message (see §2.2)
            │       ├── <UserBubble />       — plain text prompt
            │       ├── <AssistantBubble>
            │       │   ├── <StatusBadge />  — planning / running / done
            │       │   ├── <QueryPlanSteps />  — numbered step list
            │       │   ├── <StepResult />   — one per partial_result (collapsible)
            │       │   │   ├── <SqlCodeBlock />  — syntax-highlighted SQL
            │       │   │   └── <DataTable />     — rows preview
            │       │   ├── <FinalResultPanel>
            │       │   │   ├── <DataTable />
            │       │   │   ├── <ChartPanel />
            │       │   │   └── <DownloadButton />
            │       │   └── <ErrorBanner />
            │       └── <ThinkingIndicator /> — animated dots while streaming
            │
            └── <ChatInputBar>
                ├── <PromptTextarea />        — auto-resize, Enter to send
                └── <SendButton />
```

### 2.2 ChatMessage Props

`ChatMessage` is a discriminated-union renderer. The message shape from the store determines which sub-component renders:

```typescript
// Discriminated union — see §4.1 for full store type
type MessageKind =
  | "user"
  | "status"
  | "query_plan"
  | "partial_result"
  | "final_result"
  | "error";
```

| Sub-component | Rendered for kind | Key props |
|---|---|---|
| `UserBubble` | `"user"` | `text: string` |
| `StatusBadge` + `ThinkingIndicator` | `"status"` | `message: string` |
| `QueryPlanSteps` | `"query_plan"` | `steps: string[]` |
| `StepResult` | `"partial_result"` | `step: number`, `sql: string`, `rows: Row[]` |
| `FinalResultPanel` | `"final_result"` | `queryId: string`, `rows: Row[]` |
| `ErrorBanner` | `"error"` | `code: string`, `message: string` |

### 2.3 Component Prop Signatures

```
<FileUploadZone>
  onFilesSelected: (files: File[]) => void
  disabled: boolean
  acceptedTypes: string[]           // [".csv", ".xlsx", ".xls"]

<FileChip>
  tableName: string
  fileName: string
  rowCount: number
  colCount: number

<DataTable>
  columns: ColumnDef[]              // [{ key, label, dtype }]
  rows: Row[]                       // Row = Record<string, string>
  pageSize?: number                 // default 10
  sortable?: boolean                // default true

<ChartPanel>
  columns: ColumnDef[]
  rows: Row[]
  initialChartType?: ChartType      // auto-detected (see §8.2)

<SqlCodeBlock>
  sql: string
  collapsible?: boolean             // default true

<DownloadButton>
  queryId: string
  disabled: boolean
```

---

## 3. Zustand Store Design

### 3.1 Store Slices Overview

The single `useAppStore` is divided into four logical slices composed in one `create()` call:

```
useAppStore
├── authSlice      — JWT, user identity
├── sessionSlice   — session ID, connection status
├── chatSlice      — message list, streaming state
└── uploadSlice    — upload queue, file refs
```

### 3.2 `authSlice`

```typescript
interface AuthState {
  jwt: string | null;
  userId: string | null;
  isAuthenticated: boolean;
}

interface AuthActions {
  setAuth: (jwt: string, userId: string) => void;
  clearAuth: () => void;
}
```

- `setAuth` writes JWT to `localStorage` and updates store.
- `clearAuth` removes JWT from `localStorage`, resets slice, triggers WS disconnect.
- `jwt` is initialised by reading `localStorage` on store creation (hydration).

### 3.3 `sessionSlice`

```typescript
interface SessionState {
  sessionId: string | null;
  wsStatus: "disconnected" | "connecting" | "connected" | "reconnecting";
  sessionTtlMinutes: number;       // remaining minutes display
}

interface SessionActions {
  setSessionId: (id: string) => void;
  setWsStatus: (s: WsStatus) => void;
  setTtl: (minutes: number) => void;
}
```

- `sessionId` is persisted to `localStorage` so reconnect can reuse the same session.
- `wsStatus` drives the `SessionStatus` indicator and disables the `SendButton`.

### 3.4 `chatSlice`

```typescript
interface ChatMessage {
  id: string;                         // client-generated UUID
  kind: MessageKind;                  // discriminated union key
  createdAt: number;                  // Date.now()

  // kind === "user"
  text?: string;

  // kind === "status"
  message?: string;

  // kind === "query_plan"
  steps?: string[];

  // kind === "partial_result"
  step?: number;
  sql?: string;
  rows?: Row[];
  csvPath?: string;

  // kind === "final_result"
  queryId?: string;
  finalRows?: Row[];
  finalCsvPath?: string;

  // kind === "error"
  code?: string;
  errorMessage?: string;
}

interface ChatState {
  messages: ChatMessage[];
  isStreaming: boolean;              // true while WS messages are arriving
  activeQueryId: string | null;     // query being executed right now
}

interface ChatActions {
  appendMessage: (msg: ChatMessage) => void;
  setStreaming: (v: boolean) => void;
  setActiveQueryId: (id: string | null) => void;
  clearMessages: () => void;
}
```

- `appendMessage` is called by the `useWebSocket` hook on each incoming WS event.
- `isStreaming` disables the chat input to prevent concurrent queries.
- Messages are **not** persisted to localStorage — they live only for the browser session (history is not a V1 requirement).

### 3.5 `uploadSlice`

```typescript
interface FileRef {
  tableName: string;
  fileName: string;
  rowCount: number;
  columns: ColumnDef[];
  uploadedAt: number;
}

interface UploadItem {
  id: string;                         // client UUID
  file: File;
  status: "pending" | "uploading" | "done" | "error";
  errorMessage?: string;
  result?: FileRef;
}

interface UploadState {
  queue: UploadItem[];
  uploadedFiles: FileRef[];           // successfully ingested tables
}

interface UploadActions {
  enqueueFiles: (files: File[]) => void;
  setItemStatus: (id: string, status: UploadItem["status"], detail?: Partial<UploadItem>) => void;
  addFileRef: (ref: FileRef) => void;
  clearQueue: () => void;
}
```

---

## 4. WebSocket Integration

### 4.1 Connection Lifecycle

```
useWebSocket hook (called once from <ChatPage>)
  │
  ├── on mount
  │   ├── read sessionId + jwt from store
  │   ├── ws = new WebSocket(`${WS_BASE_URL}/ws/${sessionId}?token=${jwt}`)
  │   └── attach onopen / onmessage / onerror / onclose handlers
  │
  ├── onopen → store.setWsStatus("connected")
  │
  ├── onmessage → dispatch(msg)         (see §4.2)
  │
  ├── onclose
  │   ├── store.setWsStatus("reconnecting")
  │   └── schedule exponential-backoff reconnect (max 5 attempts, cap 30s)
  │
  └── on unmount → ws.close()
```

The hook exposes a `sendQuery(prompt: string)` function used by `<ChatInputBar>`.

```typescript
// Outbound message shape (matches backend WSRequestContext)
interface QueryMessage {
  type: "query";
  prompt: string;
}
```

### 4.2 Inbound Message Dispatch Table

Each WS frame is a JSON object. The `type` field routes to the correct store action:

| `type` value | Backend source | Frontend action |
|---|---|---|
| `"status"` | `WSMessageFactory.status()` | `appendMessage({ kind:"status", message })` |
| `"query_plan"` | `WSMessageFactory.query_plan()` | `appendMessage({ kind:"query_plan", steps })` |
| `"partial_result"` | `WSMessageFactory.partial_result()` | `appendMessage({ kind:"partial_result", step, sql, rows, csvPath })` |
| `"final_result"` | `WSMessageFactory.final_result()` | `appendMessage({ kind:"final_result", queryId, finalRows })` then `setStreaming(false)` |
| `"error"` | `WSMessageFactory.error()` | `appendMessage({ kind:"error", code, errorMessage })` then `setStreaming(false)` |

```typescript
// Full inbound message union (mirrors WSMessageFactory output)
type InboundWsMessage =
  | { type: "status";          message: string }
  | { type: "query_plan";      steps: string[] }
  | { type: "partial_result";  step: number; sql: string; rows: Row[]; csv_path: string }
  | { type: "final_result";    query_id: string; rows: Row[]; csv_path: string }
  | { type: "error";           code: string; message: string };
```

### 4.3 Sending a Query

```
User presses Enter / Send
  │
  ├── appendMessage({ kind:"user", text: prompt })
  ├── store.setStreaming(true)
  ├── ws.send(JSON.stringify({ type:"query", prompt }))
  └── (WS messages arrive asynchronously — see §4.2)
```

### 4.4 Reconnect Strategy

```typescript
const BACKOFF_BASE_MS = 1000;
const BACKOFF_CAP_MS  = 30_000;
const MAX_ATTEMPTS    = 5;

// delay = min(base * 2^attempt + jitter, cap)
```

On reconnect the same `sessionId` is reused. The backend's `SessionGatekeeper` validates it, calls `Facade.get_or_create_session()`, and reloads `FileRefs` — so the DuckDB table state is restored transparently.

---

## 5. File Upload Flow

### 5.1 Step-by-Step

```
1. User drops file(s) on <FileUploadZone> or clicks to browse
   │
   └── onFilesSelected(files) called

2. useFileUpload hook receives files
   ├── validate extension (.csv / .xlsx / .xls)
   ├── for each valid file: store.enqueueFiles([file])
   └── for each invalid file: show inline validation error

3. Hook iterates queue, picks next "pending" item, sets status → "uploading"

4. POST /files/upload  (multipart/form-data)
   Headers: Authorization: Bearer <jwt>
   Body:    file=<binary>  session_id=<sessionId>

5. On 200 { table_name, row_count, columns }
   ├── store.setItemStatus(id, "done", { result: FileRef })
   ├── store.addFileRef(FileRef)
   └── <FileChip> appears in <UploadedFileList>

6. On error (4xx / 5xx / network)
   └── store.setItemStatus(id, "error", { errorMessage })
       → <FileChip> shows red error state with retry button

7. Files are processed serially (one at a time) to avoid
   saturating the backend upload endpoint.
```

### 5.2 Validation Rules (client-side, pre-upload)

| Rule | Detail |
|---|---|
| Accepted extensions | `.csv`, `.xlsx`, `.xls` |
| Max file size | 50 MB (configurable via `VITE_MAX_UPLOAD_MB`) |
| Duplicate table name | Warn user if `tableName` already in `uploadedFiles` |
| Empty file | Reject with inline error before POST |

### 5.3 useFileUpload Hook Interface

```typescript
interface UseFileUploadReturn {
  queue: UploadItem[];
  uploadedFiles: FileRef[];
  handleFiles: (files: File[]) => void;        // entry point from <FileUploadZone>
  retryItem: (id: string) => void;             // retry a failed upload
}
```

---

## 6. Chat UI Design

### 6.1 Message Rendering Rules

Messages are rendered top-to-bottom in `<ChatWindow>`. `ChatWindow` auto-scrolls to the bottom on each new message while `isStreaming` is true.

| ChatMessage kind | Visual | Alignment |
|---|---|---|
| `"user"` | Rounded pill, accent bg | Right |
| `"status"` | Small pill, muted, animated pulse | Left, indented |
| `"query_plan"` | Numbered ordered list, monospace | Left |
| `"partial_result"` | Collapsible card: SQL block + mini DataTable (top 5 rows) | Left |
| `"final_result"` | Full DataTable + ChartPanel + DownloadButton | Left |
| `"error"` | Red alert banner with `code` badge | Left |

### 6.2 Streaming UX

While `isStreaming === true`:
- `<ThinkingIndicator>` (three animated dots) renders at the bottom of the message list.
- `<ChatInputBar>` is disabled (textarea + button).
- Each `partial_result` message animates in as it arrives (fade-in).
- `query_plan` steps tick in one-by-one as the agent emits them (not all at once).

### 6.3 Query Plan Steps Display

`QueryPlanSteps` renders a numbered list. As `partial_result` events arrive for step N, the Nth item gets a checkmark icon and the step's DataTable is inserted below it (collapsible). This gives a visual progress indicator.

```
Query plan:
  1. ✓ Get schema                  ← completed, chevron to expand SQL + rows
  2. ✓ SELECT revenue BY month     ← completed
  3.   Aggregate by region         ← pending (no result yet)
```

### 6.4 Final Result Panel

`<FinalResultPanel>` renders after `final_result` arrives:

```
┌─────────────────────────────────────────────────────────┐
│  Results  (124 rows · showing 10)          [Download CSV]│
│  ┌──────────────────────────────────────────────────┐   │
│  │  <DataTable rows=finalRows columns=... />        │   │
│  └──────────────────────────────────────────────────┘   │
│  [Bar]  [Line]  [Pie]  [Scatter]  ← chart type tabs      │
│  ┌──────────────────────────────────────────────────┐   │
│  │  <ChartPanel rows=finalRows />                   │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## 7. DataTable Component Design

### 7.1 Column Definition

```typescript
interface ColumnDef {
  key: string;            // matches Row key
  label: string;          // display header
  dtype: string;          // from backend schema: "VARCHAR" | "INTEGER" | "FLOAT" | ...
  numeric: boolean;       // derived: dtype in ["INTEGER","FLOAT","DOUBLE","BIGINT"]
}
```

### 7.2 Props and Behaviour

```typescript
interface DataTableProps {
  columns: ColumnDef[];
  rows: Row[];
  pageSize?: number;         // default: 10
  sortable?: boolean;        // default: true
  compact?: boolean;         // default: false (true inside StepResult)
  className?: string;
}
```

| Feature | Detail |
|---|---|
| Pagination | Client-side. Controls: prev / next / page-number pills. `pageSize` configurable, presets: 10, 25, 50. |
| Sorting | Click column header to sort asc, click again for desc, third click resets. Sort is client-side over current `rows` prop. |
| Numeric alignment | Numeric columns right-aligned. Non-numeric left-aligned. |
| Type formatting | `FLOAT` / `DOUBLE` → `toLocaleString(2 decimal)`. `INTEGER` → `toLocaleString()`. Dates pass through. |
| Empty state | Renders "No results" with icon when `rows.length === 0`. |
| Overflow | Horizontal scroll via `overflow-x: auto` wrapper. Cells truncate at 200px with tooltip. |
| Column count | Renders all columns. For >10 columns, horizontal scroll activates. |

### 7.3 State (internal, no Zustand)

```typescript
// local useState inside DataTable
const [page, setPage] = useState(0);
const [sortKey, setSortKey] = useState<string | null>(null);
const [sortDir, setSortDir] = useState<"asc" | "desc">("asc");
```

Sorted + paginated rows are computed with `useMemo` from `rows`, `sortKey`, `sortDir`, and `page`.

---

## 8. Recharts Integration

### 8.1 Chart Types Supported

| Chart type | Recharts component | Best for |
|---|---|---|
| Bar | `<BarChart>` | Categorical comparisons (e.g. revenue by region) |
| Line | `<LineChart>` | Time-series trends |
| Pie | `<PieChart>` | Part-to-whole proportions (≤ 8 categories) |
| Scatter | `<ScatterChart>` | Correlation between two numeric columns |

### 8.2 Auto-Detection Logic

When `final_result` arrives, `ChartPanel` runs `detectChartType(columns, rows)`:

```
1. Count numeric columns (nc) and non-numeric columns (cc)

2. If nc >= 1 and cc === 1:
   - If first non-numeric column looks like a date → LineChart
   - Else if distinct values in cc column <= 8 → PieChart
   - Else → BarChart

3. If nc >= 2 and cc === 0:
   → ScatterChart (first two numeric cols as x/y)

4. Fallback: BarChart

User can always override via the chart-type tabs in <FinalResultPanel>.
```

### 8.3 Data Flow: SQL Result → Chart

```
final_result WS message
  │
  └── rows: Row[]   (each Row = Record<string, string>)
            │
            ▼
  transformToChartData(rows, columns, chartType)
            │
            ├── BarChart / LineChart / PieChart:
            │   pick xKey (first non-numeric) and yKey (first numeric)
            │   map rows → [{ [xKey]: row[xKey], [yKey]: parseFloat(row[yKey]) }]
            │
            └── ScatterChart:
                pick xKey (first numeric) and yKey (second numeric)
                map rows → [{ x: parseFloat(row[xKey]), y: parseFloat(row[yKey]) }]
            │
            ▼
  <BarChart data={chartData}>
    <XAxis dataKey={xKey} />
    <YAxis />
    <Tooltip />
    <Legend />
    <Bar dataKey={yKey} />
  </BarChart>
```

### 8.4 ChartPanel Props

```typescript
interface ChartPanelProps {
  columns: ColumnDef[];
  rows: Row[];
  initialChartType?: ChartType;     // from auto-detection
}

// Internal state
const [chartType, setChartType] = useState<ChartType>(initialChartType);
```

### 8.5 Recharts Configuration Defaults

```typescript
const CHART_DEFAULTS = {
  width: "100%",           // ResponsiveContainer fills parent
  height: 320,             // px
  colors: [                // Tailwind slate-500 palette progression
    "#6366f1", "#22d3ee", "#f59e0b", "#10b981", "#f43f5e", "#8b5cf6",
  ],
  margin: { top: 8, right: 16, bottom: 8, left: 0 },
  animationDuration: 400,  // ms
};
```

All Recharts charts are wrapped in `<ResponsiveContainer width="100%" height={320}>` so they reflow correctly in the chat pane.

---

## 9. API Service Layer

### 9.1 `api.ts` — HTTP Client

```typescript
// base URL from env
const API_BASE = import.meta.env.VITE_API_BASE_URL;  // e.g. https://bisense.up.railway.app

// Authenticated fetch wrapper
async function apiFetch(path: string, init?: RequestInit): Promise<Response>
  // Injects Authorization: Bearer <jwt> header from store
  // Throws ApiError on non-2xx

// Endpoints
export const api = {
  uploadFile: (sessionId: string, file: File) => FormData POST /files/upload,
  getDownloadUrl: (queryId: string) => GET /files/download/{queryId},
  initiateGoogleAuth: () => GET /auth/google  (redirect),
};
```

### 9.2 Environment Variables

| Variable | Purpose | Example |
|---|---|---|
| `VITE_API_BASE_URL` | Backend HTTP base URL | `https://bisense.up.railway.app` |
| `VITE_WS_BASE_URL` | Backend WebSocket base URL | `wss://bisense.up.railway.app` |
| `VITE_MAX_UPLOAD_MB` | Client-side upload size limit | `50` |

---

## 10. Directory Structure

```
frontend/
├── src/
│   ├── main.tsx                    # React root, RouterProvider
│   ├── App.tsx                     # Route definitions
│   │
│   ├── pages/
│   │   ├── LoginPage.tsx
│   │   ├── AuthCallbackPage.tsx    # reads ?token=, calls store.setAuth()
│   │   └── ChatPage.tsx            # shell: <Sidebar> + <ChatPane>
│   │
│   ├── components/
│   │   ├── chat/
│   │   │   ├── ChatWindow.tsx      # scrollable message list
│   │   │   ├── ChatMessage.tsx     # discriminated-union renderer
│   │   │   ├── ChatInputBar.tsx    # textarea + send button
│   │   │   ├── UserBubble.tsx
│   │   │   ├── AssistantBubble.tsx
│   │   │   ├── QueryPlanSteps.tsx
│   │   │   ├── StepResult.tsx      # collapsible: SQL + DataTable
│   │   │   ├── FinalResultPanel.tsx
│   │   │   ├── ThinkingIndicator.tsx
│   │   │   ├── StatusBadge.tsx
│   │   │   └── ErrorBanner.tsx
│   │   │
│   │   ├── upload/
│   │   │   ├── FileUploadZone.tsx  # drag-drop + file picker
│   │   │   ├── FileChip.tsx
│   │   │   └── UploadedFileList.tsx
│   │   │
│   │   ├── visualization/
│   │   │   ├── DataTable.tsx
│   │   │   ├── ChartPanel.tsx
│   │   │   └── SqlCodeBlock.tsx
│   │   │
│   │   └── ui/
│   │       ├── Sidebar.tsx
│   │       ├── AppLogo.tsx
│   │       ├── SessionStatus.tsx
│   │       ├── GoogleSignInButton.tsx
│   │       ├── DownloadButton.tsx
│   │       └── ProtectedRoute.tsx  # redirects to /login if !authed
│   │
│   ├── hooks/
│   │   ├── useWebSocket.ts         # WS lifecycle + dispatch
│   │   └── useFileUpload.ts        # upload queue processor
│   │
│   ├── store/
│   │   └── appStore.ts             # Zustand: all four slices composed
│   │
│   ├── services/
│   │   └── api.ts                  # HTTP client wrapper
│   │
│   ├── types/
│   │   └── index.ts                # shared TypeScript types
│   │
│   └── index.css                   # Tailwind base + global resets
│
├── public/
│   └── favicon.svg
│
├── index.html
├── vite.config.ts
├── tailwind.config.ts
├── tsconfig.json
└── package.json
```

---

## 11. Key Constraints & Decisions

| Concern | Decision | Reason |
|---|---|---|
| State library | Zustand (single store, slices) | Low boilerplate; no context provider nesting required |
| WS reconnect | Exponential backoff, max 5 attempts, reuse same sessionId | Transparent resume; backend reloads FileRefs on reconnect |
| Message persistence | Messages live in memory only (no localStorage) | History not a V1 requirement; avoids stale data issues |
| Chart auto-detection | Heuristic on column dtypes and cardinality | Good default UX; user override always available via tabs |
| Upload serialisation | One file at a time | Avoids server resource contention; simpler error recovery |
| DataTable sorting/pagination | Client-side (no re-fetch) | All row data already in memory (backend returns top 10; user downloads full CSV) |
| JWT storage | `localStorage` | Simplest approach for SPA; acceptable for MVP |
| Concurrent queries | Blocked at UI level while `isStreaming` | Backend doesn't support concurrent WS messages per session in V1 |
| Row cap in chat | Top 10 rows shown in DataTable (mirrors backend `partial_result` row cap) | Full dataset available via Download CSV |
| Recharts sizing | All charts use `ResponsiveContainer` | Correct reflow in variable-width chat pane |
