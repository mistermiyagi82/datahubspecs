# DataPraat — Product Vision

## What is DataPraat?

DataPraat is a data governance platform for Dutch accountants and data consultants. It takes messy source files from clients — CSV exports, XML feeds, Excel spreadsheets — and turns them into trusted, traceable, auditable data that powers dashboards and AI-driven insights.

The core problem: companies deliver financial data in various formats. Today, a data consultant (Thom) manually opens each file, eyeballs it for obvious problems, asks the client questions, fixes issues, and merges everything into a master file for Power BI. This process is time-consuming, error-prone, and leaves no audit trail. When someone asks "where does this number come from?", the answer lives in Thom's head.

DataPraat makes this process structured, automated, and fully traceable.

---

## The Three Parts

### Part 1 — Validate & Understand

> *"What's in this file and can we trust it?"*

This is where the validation framework lives. A file arrives, and the system:

- **Parses it** — can we open it? Is it CSV, XML, Excel? Is it corrupted? What encoding, what delimiter? (Phase 1-2 checks)
- **Profiles it** — what's in each column? Fill rates, data types, unique values, distributions, date ranges. (Phase 3)
- **Validates it** — are the values correct? Valid dates, consistent formats, no impossible values, duplicates detected, Dutch number notation handled. (Phase 4-5)
- **Flags anomalies** — statistical outliers, suspiciously homogeneous columns, AI-powered semantic interpretation of what each column likely contains. (Phase 6-7)

The output is a **validation report** — a report card for the file. A human (Thom) reviews it, sees the red/yellow/green flags, and decides: is this good enough, or do I need to call the client?

**The file itself is never modified here.** This part only observes and documents. It answers questions that Thom would otherwise spend 30-60 minutes figuring out manually:

- Is the row count what we expected?
- Are there columns that are mostly empty?
- Did Excel eat the leading zeros from postcodes again?
- Are the totals in the right order of magnitude?
- Are there dates in the future or numbers that make no sense?

This is Thom's eyeball check, automated. It is the first rung of Harm Nico's 5-step validation ladder: **internal consistency** — checking the data against itself.

**76 checks defined, 48 in MVP.** See VALIDATION_FRAMEWORK_COMPLETE_EN.md for the full list.

---

### Part 2 — Transform & Trace

> *"Turn validated sources into one trusted master file, and remember everything."*

This is where the actual work happens:

- **Multiple validated files come together** — different sources, different clients, different periods, different formats. A municipality might deliver invoicing data as CSV, care trajectory data as XML, and budget data as Excel.
- **A human maps columns, resolves conflicts, applies transformations** — Dutch number format to standard, rename columns to match the schema, merge product categories, handle overlapping date ranges.
- **Every action is logged** — who changed what, when, why. Not just "column renamed" but "column 'afdeling_code' renamed to 'department_code' by Thom on 2026-04-09, reason: client changed their export system."
- **Lineage is tracked** — every value in the master file points back to its source file, source row, source column. Click a number → see exactly where it came from.
- **The result is the master file** — the single source of truth that Power BI dashboards are built on.
- **Context expectations live here too** — "client said total should be ~5.2M (source: email 2026-03-15)", "they renamed 'afdeling' to 'dept' this quarter", "Q3 data was delivered late and may be incomplete."

This is where Harm Nico's validation ladder extends beyond Step 1:

| Step | What it does | Example |
|------|-------------|---------|
| Step 1 | Internal consistency (Part 1) | "Are the values in this file valid?" |
| Step 2 | Compare against previous delivery | "Last quarter had 4,200 rows, this one has 412 — 90% drop" |
| Step 3 | Compare against peer organisations | "This municipality spends 2.3x the average for its size class" |
| Step 4 | Cross-check with other data from the same source | "The jaarrekening says €5.2M but our data says €13.5M" |
| Step 5 | Validate against independent external data | "CBS population data says 48,000 inhabitants — does client count match?" |

Each comparison adds a **confidence layer** to the master file. A number that passed all 5 steps is highly trustworthy. A number that only passed Step 1 should be treated with caution.

**This part is the audit trail.** If someone looks at a number in the dashboard and asks "where does this come from?", Part 2 has the complete answer: source file, source row, who uploaded it, what checks it passed, who approved it, what transformations were applied, and why.

---

### Part 3 — Talk to Your Data

> *"Ask questions, get answers with context."*

An AI layer on top of the master file. But unlike generic "chat with CSV" tools, this AI has **context**:

- **It knows the validation results from Part 1** — data quality flags, known gaps, columns that were mostly empty, dates that were inconsistent.
- **It knows the lineage from Part 2** — where values came from, what was changed, what the audit trail looks like.
- **It surfaces deviations from norms** — not just "what happened" but "is this normal compared to peers, compared to history, compared to the budget?"
- **It acknowledges uncertainty** — if validation flagged something, the AI mentions it. "Revenue in Q3 was €5.2M, but note that 12% of invoices in the source file had date inconsistencies that may affect this total."
- **It never pretends to know more than the data supports** — per Harm Nico's philosophy, the tool provides data-driven input for human decision-making. It does not prescribe policy.

This is the difference between DataPraat and any generic AI data tool: **the context chain**. A raw LLM on raw data hallucinates confidently. An LLM that knows what the data quality looks like, where each number came from, and what the norms are — that's actually useful.

---

## The Flow

```
Companies deliver files (CSV, XML, Excel)
        │
        ▼
  ┌─────────────┐
  │   PART 1    │  Validate & Understand
  │  (read-only)│  → validation report, metadata, quality flags
  │             │  → 76 automated checks (48 MVP)
  │             │  → "can we trust this file?"
  └──────┬──────┘
         │ human reviews report, approves or rejects
         ▼
  ┌─────────────┐
  │   PART 2    │  Transform & Trace
  │  (edits OK) │  → master file with full lineage
  │             │  → audit trail (who, what, when, why)
  │             │  → validation ladder steps 2-5
  │             │  → context & expectations captured
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │   PART 3    │  Talk to Your Data
  │   (AI/LLM)  │  → answers grounded in context & confidence
  │             │  → deviations from norms surfaced
  │             │  → uncertainty acknowledged
  └─────────────┘
         │
         ▼
  Power BI / Dashboards
  (built on the trusted master file)
```

---

## What We're Building First

**Part 1 is the foundation.** Without validated data, Part 2 and Part 3 are built on sand. The implementation plan (see IMPLEMENTATION_PLAN.md) covers the backend validation engine: a Python + FastAPI service with 48 pluggable checks across 7 phases.

Part 2 and Part 3 come after Part 1 is solid.

---

## The Philosophy Behind It

> *"In norm zit afwijking en in afwijking zit betekenis."*
> (In norms sit deviations, and in deviations sits meaning.)
> — Harm Nico Plomp

Data on its own says nothing. Data only becomes information when measured against a norm. The same analysis that validates data quality **also** produces the norms needed for diagnosis. Large deviations = probably a data issue. Small deviations = probably real information. They are the same process.

DataPraat makes this chain visible at every step: **data → norm → deviation → meaning → action.**

See HARM-NICO-DATA-PHILOSOPHY.md for the full framework.

---

## Key Principles

1. **Financial data first.** Billing/invoicing data is the most reliable because there's money at stake. Always anchor analyses here. Layer quality/process data on top with explicit confidence indicators.

2. **Validate before you present.** No data enters the system without passing through the validation pipeline. Unvalidated data is never shown without a flag.

3. **Make the norm explicit.** Never present a number without context. "€13.5M" means nothing. "€13.5M — 2.3x the average for municipalities of this size" means everything.

4. **Provide proof at every step.** Users must be able to drill down to source rows, see the formula, verify independently. Trust is built through transparency, not claims.

5. **Acknowledge the gap.** Data can tell you what is efficient. It cannot tell you what is just. The tool supports the human who makes the decision — it doesn't make the decision.

6. **Read-only first, edit later.** Part 1 never modifies the source file. It describes, profiles, and validates. Modifications happen in Part 2 with full audit trail.

---

## Target Users

- **Data consultants** (like Thom) who receive client files and need to quickly assess quality, identify problems, and prepare data for analysis.
- **Accountants** in the Netherlands working with financial data from multiple clients and sources.
- **Municipal data teams** working with youth care (Jeugdzorg), social services, and financial reporting data.

---

---

## What's Been Built — Part 3 Frontend

> This section describes the existing codebase in `vibathon-knowledgegraph/`. It covers the **frontend half** of Part 3 — the conversational UI, streaming AI integration, and data visualization layer. The backend/MCP server and database that feed it are described separately.

### Overview

We have a working Next.js application that provides:

- A **chat interface** where users ask questions in natural language and get streaming AI responses
- **Interactive charts** (bar, area, pie, treemap, composed, table) that the AI generates on the fly from tool call results
- **MCP integration** — the AI calls tools on an external MCP server (VVD Dashboard) to query real data, rather than hallucinating answers
- **Multi-model support** — Claude (default), GPT-4o, Groq Llama, selectable per conversation
- **Persistent chat history** — conversations are stored on disk with full metadata, tool invocations, token usage, and cost tracking
- **File upload** — users can attach CSV, Excel, PDF, etc. to a conversation; the AI reads the contents as context

This is a standalone deployable app (Railway or Vercel), not a library. It runs as a Next.js server with API routes handling the AI orchestration and a React frontend rendering the conversation.

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Framework | Next.js 14 (App Router) |
| UI | React 18 + Tailwind CSS + shadcn/ui |
| AI orchestration | Vercel AI SDK (`streamText`, `experimental_createMCPClient`) |
| LLM providers | `@ai-sdk/anthropic`, `@ai-sdk/openai`, Groq (OpenAI-compatible) |
| Charts | Recharts 3.8 |
| Markdown | react-markdown + remark-gfm |
| Storage | Filesystem (JSON + Markdown) + SQLite (embeddings, facts) |
| Embeddings | Voyage AI (text-embedding-3) |
| Language | TypeScript 5 |

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Browser                                                │
│  ┌────────────┐  ┌──────────────┐  ┌────────────────┐  │
│  │  ChatView   │  │ MessageBubble│  │   VVDChart     │  │
│  │  (streaming │  │ (markdown +  │  │ (recharts,     │  │
│  │   useChat)  │  │  tool results│  │  chart-type    │  │
│  │             │  │  + charts)   │  │  switching)    │  │
│  └──────┬──────┘  └──────────────┘  └────────────────┘  │
└─────────┼───────────────────────────────────────────────┘
          │ POST /api/chats/[chatId]/messages (SSE stream)
          ▼
┌─────────────────────────────────────────────────────────┐
│  Next.js API Route (messages/route.ts)                  │
│                                                         │
│  1. Load chat metadata + system prompt                  │
│  2. Attach uploaded file contents to message             │
│  3. Connect to MCP server (VVD Dashboard)               │
│  4. streamText(Claude, { tools: mcpTools + memoryTools })│
│  5. AI decides which tools to call (up to 5 steps)      │
│  6. Stream response tokens back to browser              │
│  7. On finish: persist messages, index embeddings,      │
│     extract entities (async, fire-and-forget)           │
└────────────┬────────────────────────────────────────────┘
             │ MCP tool calls (streamable HTTP)
             ▼
┌─────────────────────────────────────────────────────────┐
│  External MCP Server (e.g. VVD Dashboard on Railway)    │
│  Tools: vvd_actuals_summary, cbs_kosten_summary, etc.  │
│  Returns: structured JSON (rows, columns, aggregations) │
└─────────────────────────────────────────────────────────┘
```

### Key Components

**Chat UI** (`src/components/chat/`)

- `ChatView.tsx` — Main chat page. Uses Vercel AI SDK's `useChat` hook for streaming. Handles model selection, sidebar toggles, the message list, and auto-scroll.
- `MessageBubble.tsx` — Renders a single message. Parses markdown, detects tool invocations in the message, and renders `VVDChart` components inline when the AI called a data tool.
- `MessageInput.tsx` — Text input with file upload button. Supports drag-and-drop.

**Charts** (`src/components/charts/`)

- `VVDChart.tsx` — A universal chart renderer. Receives raw tool call results (JSON rows) and renders them as interactive Recharts visualizations. Supports 8 chart types with a switcher UI. Auto-detects whether data is categorical (→ bar/pie), time series (→ area/line), or comparison (→ composed). Shows KPI summary cards below each chart.

**AI Orchestration** (`src/lib/`)

- `anthropic.ts` — System prompt definition (Dutch, tailored for Jeugdzorg data analysis). Defines the AI's personality, tool usage rules, and response formatting expectations.
- `prompt.ts` — Prompt assembly: base prompt + per-chat overrides + global overrides + skill modules.
- `tools.ts` — Builds the memory tool set: `query_memory`, `search_history`, `get_recent_messages`, `grep_history`, `query_graph`, `get_calendar_availability`.

**Memory System** (`src/lib/memory/`)

A three-layer persistent memory that enriches every conversation:

| Layer | How it works | Storage |
|-------|-------------|---------|
| **Recent** | Last N messages from current chat | Filesystem (messages.md) |
| **Semantic** | Voyage AI embeddings + cosine similarity search across all chats | SQLite (`chunks` table) |
| **Graph** | Entity extraction (via Haiku) → fact triples, keyword-searchable | SQLite (`facts` table) + Graphiti service |

After each exchange, the system asynchronously: (1) indexes the exchange as an embedding chunk, (2) extracts entities and relationships via a cheap LLM call, (3) ingests the episode into the knowledge graph. The AI can then query any of these layers as tools during future conversations.

**Storage** (`src/lib/storage/`)

- `chats.ts` — CRUD for chats. Each chat is a directory under `data/chats/{chatId}/` with `meta.json` (metadata, model, agent config) and `messages.md` (full conversation with embedded context as HTML comments).
- `files.ts` — File upload storage under `data/uploads/{chatId}/`. Supports text, CSV, JSON, PDF, images.

### What This Means for DataPraat

This codebase is **Part 3's presentation layer**. In DataPraat terms:

| What it does today | What it becomes in DataPraat |
|-------------------|------------------------------|
| Connects to VVD Dashboard MCP server | Connects to **DataPraat's MCP server** (master file queries, validation results, lineage lookups) |
| System prompt about Jeugdzorg | System prompt about **the client's data domain** — informed by Part 1 validation results and Part 2 lineage |
| VVDChart renders youth care data | Charts render **any validated financial data** from the master file |
| Memory stores conversation facts | Memory stores **data quality context** — "last time we got data from client X, column Y was 40% empty" |
| File upload sends files to chat context | File upload triggers **Part 1 validation pipeline** and shows the report |

The key architectural decision already made: **the AI talks to data through MCP tools, not direct database access.** This means Part 3's frontend doesn't need to know how the data is stored or validated — it just calls tools and renders results. The MCP server (Part 3's backend) is the boundary.

### What's Reusable As-Is

- **Chat UI components** — ChatView, MessageBubble, MessageInput. Generic streaming chat with tool result rendering.
- **VVDChart** — Already supports arbitrary JSON data with multiple chart types. Needs minor adaptation for new data shapes, but the rendering engine is solid.
- **Vercel AI SDK integration** — streamText + MCP client pattern. Swap the MCP server URL and system prompt, and it works with any MCP-compatible backend.
- **Memory system** — The three-layer memory (recent/semantic/graph) is domain-agnostic. Works for any conversational context.
- **Storage layer** — File-based chat persistence. Simple, works for single-instance deployment.
- **Multi-model support** — Claude/GPT/Groq selection per chat. No changes needed.

### What Needs Changing

- **System prompt** — Currently hardcoded for Jeugdzorg/VVD. Needs to become dynamic, driven by the client's data domain and Part 1/2 context.
- **MCP server URL** — Currently points to VVD Dashboard on Railway. Needs to point to DataPraat's own MCP server.
- **Chart data shapes** — VVDChart expects VVD-specific column names (gemeente, productcategorie, Q, P, PxQ). Needs generalization for arbitrary financial data.
- **Landing page** — Currently shows a flat chat list. DataPraat needs a workspace concept: per-client, per-delivery, with validation status visible.
- **File upload flow** — Currently just attaches files to chat context. In DataPraat, uploading a file should trigger Part 1's validation pipeline and show the report before any AI conversation.

### What's Not Part of DataPraat

The following features exist in this codebase but are **not relevant** to DataPraat:

- **Attio CRM integration** (`src/lib/attio.ts`, recruiter mode, contact picker) — recruitment-specific, remove entirely.
- **Google Calendar integration** (`src/lib/calendar.ts`) — scheduling feature, not needed.
- **Graphiti service** (`graphiti_service/`) — external knowledge graph service. The SQLite-based memory is sufficient; Graphiti adds operational complexity without clear value for DataPraat's use case.
- **Recruiter templates and agent configs** — DataPraat will have its own template system based on client/delivery context.

---

## What's Been Built — Part 3 Backend / MCP / Database

> This section describes the existing codebase in `vvd-viewer/`. It covers the **backend half** of Part 3 — the MCP server, database, data access layer, CBS integration, and visualization hints. The frontend that calls into this is described in the previous section.

### Overview

We have a working Python MCP server that provides:

- A **DuckDB analytical database** with 9 tables (4 VVD fact/dimension tables + 5 CBS open data tables) holding ~75K rows of youth care forecasting and actuals data
- **14 MCP tool endpoints** that the AI calls to query data — forecasts, actuals, comparisons, summaries, CBS benchmarks, and cross-analyses
- **2 MCP resources** (schema reference + Dutch/English glossary) that give the AI structural knowledge about the data
- **Chart hint generation** — every tool response includes visualization metadata (chart type, axes, colors, KPIs) so the frontend can auto-render charts without a second round trip
- **CBS open data integration** — an ETL pipeline that fetches 3 CBS OData APIs (municipal costs, client volumes, referral trajectories) covering all 414 Dutch municipalities
- **REST API endpoints** (legacy) for direct HTTP access to actuals data
- **Docker + Railway deployment** — production-ready containerized deployment with health checks

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Server framework | FastMCP + Starlette (ASGI) |
| Runtime | Python 3.12 + Uvicorn |
| Database | DuckDB (embedded OLAP, read-only) |
| MCP transport | Streamable HTTP (stateless) |
| HTTP client | httpx (async, for CBS OData fetching) |
| Validation | Pydantic 2 (tool parameter schemas) |
| Config | python-dotenv |
| Deployment | Docker (python:3.12-slim) + Railway.app |
| Package manager | uv |

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  External Data Sources                                      │
│  ├─ VVD CSV (1.2M rows, youth care actuals + forecasts)     │
│  └─ CBS OData APIs (83454NED, 85099NED, 85097NED)           │
└────────────────────┬────────────────────────────────────────┘
                     │ scripts/import_cbs.py (ETL)
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  DuckDB: data/vddview.duckdb (30.9 MB, read-only)          │
│                                                             │
│  VVD tables:                                                │
│  ├─ FactVoorspelling (25,628 rows) — monthly forecasts      │
│  ├─ FactWerkelijk (18,248 rows) — monthly actuals           │
│  ├─ DimJaarMaand (72 rows) — date dimension 2019–2024      │
│  └─ Instellingen (2 rows) — forecast run metadata           │
│                                                             │
│  CBS tables:                                                │
│  ├─ CbsKosten — municipal youth care costs                  │
│  ├─ CbsKerncijfers — client volumes                         │
│  ├─ CbsTrajecten — referral trajectories                    │
│  ├─ DimGemeente — 414 Dutch municipalities                  │
│  └─ DimZorgvormMapping — CBS↔VVD category mapping (42 rows) │
└────────────────────┬────────────────────────────────────────┘
                     │ asyncio.to_thread(cursor.execute())
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  FastMCP Server (src/vvd_mcp/)                              │
│                                                             │
│  server.py — Starlette ASGI app + lifespan management       │
│  db.py     — DuckDB connection + async query helper          │
│  tools.py  — 14 MCP tool definitions (2060 lines)           │
│  resources.py — schema + glossary resources (287 lines)     │
│  chart_hints.py — visualization hint generator (600 lines)  │
│                                                             │
│  Exposed at:                                                │
│  ├─ / (MCP Streamable HTTP — all tool calls + resources)    │
│  ├─ /health (Railway readiness probe)                       │
│  └─ /api/actuals (legacy REST)                              │
└────────────────────┬────────────────────────────────────────┘
                     │ MCP protocol (Streamable HTTP, stateless)
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  Clients                                                    │
│  ├─ Part 3 Frontend (Next.js, via Vercel AI SDK MCP client) │
│  ├─ Claude Desktop / Claude Code (native MCP support)       │
│  └─ Any MCP-compatible client                               │
└─────────────────────────────────────────────────────────────┘
```

### Key Components

**MCP Server** (`src/vvd_mcp/server.py`)

The entry point. A Starlette ASGI application that wraps FastMCP with:
- Lifespan management: opens DuckDB on startup, closes on shutdown
- MCP protocol mounted at `/` via `mcp.streamable_http_app()` (stateless — no session state)
- Health check at `/health` for deployment probes
- Two legacy REST routes (`/api/actuals`, `/api/actuals/detail`) for non-MCP consumers

**Database Layer** (`src/vvd_mcp/db.py`)

Minimal but deliberate:
- DuckDB opened in **read-only mode** — prevents accidental mutations
- External access disabled (`enable_external_access=false`) — no file system or network access from SQL
- Memory limit and thread count configurable via env vars (default 256MB, 2 threads)
- Single async helper: `query(ctx, sql, params)` → runs SQL in `asyncio.to_thread()` with a fresh cursor per call, returns `list[dict]`
- Connection shared across all requests (safe because read-only)

**14 MCP Tools** (`src/vvd_mcp/tools.py`)

Each tool is a Pydantic-annotated async function registered with FastMCP. They fall into three groups:

*Discovery:*
- `vvd_list_dimensions` — returns all valid filter values (gemeenten, product categories, forecast versions, date ranges). Call this first.

*VVD Data (7 tools):*
- `vvd_query_forecasts` — forecast time series with confidence intervals
- `vvd_query_actuals` — actuals time series
- `vvd_compare_forecast_actuals` — side-by-side with deviation metrics (verschil, pct, richting, CI status)
- `vvd_actuals_summary` — aggregated actuals (group by gemeente/jaar/kwartaal/maand/productcategorie)
- `vvd_forecast_summary` — aggregated forecasts with CI bounds
- `vvd_realisatie_prognose` — combined timeline: actuals up to cutoff, then forecast forward. No overlap.

*CBS Cross-Analysis (6 tools):*
- `cbs_kosten_summary` — municipal costs from 83454NED
- `cbs_kerncijfers_summary` — client volumes from 85099NED
- `cbs_trajecten_summary` — trajectories by referrer from 85097NED
- `cbs_vvd_vergelijk` — VVD actuals vs CBS realized costs side-by-side
- `cbs_vvd_intensiteit` — declarations per unique CBS child (complexity trend)
- `cbs_vvd_prijs_benchmark` — VVD price vs national average (index > 100 = more expensive)
- `cbs_vvd_prognose_vs_begroting` — triple comparison: VVD forecast vs CBS budget vs CBS realized

All tools enforce token-budget-aware row limits (100–500 depending on tool) with a truncation confirmation pattern: if results exceed the limit, the tool warns and the AI can re-call with `confirm_truncation=True`.

**MCP Resources** (`src/vvd_mcp/resources.py`)

Two static Markdown resources the AI can read for structural knowledge:
- `vvd://schema` — full table definitions, column types, constraints, sample rows
- `vvd://glossary` — bilingual Dutch/English glossary of all domain terms (Q, P, PxQ, gemeente, productcategorie, prognoselabel, etc.)

**Chart Hints** (`src/vvd_mcp/chart_hints.py`)

Attached to every summary and cross-analysis tool response in the `meta.chart` field. Each hint is a JSON spec:
```json
{
  "type": "bar",
  "title": "Kosten per gemeente in 2022",
  "x_axis": {"key": "Gemeente", "label": "Gemeente"},
  "y_axis": {"key": "PxQ", "label": "Totale kosten (EUR)"},
  "data_series": [{"key": "PxQ", "label": "Totale kosten", "color": "#3b82f6"}],
  "sort": {"by": "value", "order": "descending"},
  "kpis": [
    {"label": "Totaal", "value": "€45M", "sub": "13 gemeenten"},
    {"label": "Hoogste", "value": "Achterhoek", "sub": "€5.2M"}
  ]
}
```
Chart types: ranked bar, time bar, line, stacked area, combined (multi-axis), grouped bar. Color palette: red→cyan gradient for ranked charts, semantic colors (blue=werkelijk, orange=prognose, cyan=CBS, grey=begroot) for comparisons.

The frontend's `VVDChart` component consumes these hints to auto-render charts without needing domain logic on the client side.

**CBS ETL Pipeline** (`scripts/import_cbs.py`)

Fetches 3 CBS OData APIs with:
- Paginated fetching (follows `odata.nextLink`)
- 2D chunking (period × dimension) to stay under CBS's 10K-row API limit
- Retry logic (3 attempts, exponential backoff)
- Idempotent: drops and recreates CBS tables each run
- Builds `DimGemeente` (414 municipalities) and `DimZorgvormMapping` (42-row CBS→VVD category mapping)

Run with: `uv run scripts/import_cbs.py`

**Excel Export** (`scripts/export_to_excel.py`)

Exports DuckDB to a formatted multi-tab Excel workbook (Actuals, Forecasts, DimJaarMaand, Instellingen) with currency formatting, structured references, and alternating row shading. For offline analysis and client delivery.

### Design Decisions That Matter for DataPraat

1. **MCP as the boundary.** The frontend never touches the database. All data access goes through MCP tool calls. This means swapping the data domain (from Jeugdzorg to any financial data) is a backend-only change — the frontend just calls tools and renders results.

2. **Read-only database.** DuckDB in read-only mode with external access disabled. For DataPraat, Part 1 (validate) and Part 2 (transform) write to the database; Part 3 (talk to data) only reads. This separation is already built.

3. **Stateless HTTP.** No session state on the server. Each MCP tool call is independent. This means horizontal scaling is trivial and the server can restart without losing state (state lives in the frontend's chat persistence).

4. **Chart hints as metadata.** The backend tells the frontend *how* to visualize, not just *what* the data is. This means the frontend's chart component stays generic — it doesn't need domain-specific rendering logic.

5. **Token-aware limits.** The server manages response size to prevent LLM context overflow. This is essential for DataPraat where master files could be large — the MCP layer acts as a smart gateway.

6. **Async everything.** DuckDB queries run in `asyncio.to_thread()` to keep the event loop responsive. The ASGI stack (Uvicorn + Starlette) handles concurrent MCP requests cleanly.

### What's Reusable As-Is

- **Server scaffold** — FastMCP + Starlette + lifespan pattern. Swap tools, keep the frame.
- **Database layer** — `db.py`'s async query helper works with any DuckDB schema. The read-only + external-access-disabled pattern is exactly right for Part 3.
- **Tool structure** — Pydantic-annotated async functions with row limits, truncation, and case-insensitive filtering. The pattern repeats: build SQL from filters, execute, attach chart hint, return.
- **Chart hint system** — The hint generator and color palette are reusable. Need new chart builders for DataPraat's data shapes, but the framework (`build_chart_hint` dispatch, KPI formatting, palette system) carries over.
- **MCP resources pattern** — Static schema + glossary resources. DataPraat will have its own, but the pattern (Markdown resources the AI reads for context) is proven.
- **Deployment setup** — Dockerfile, Railway config, health check. Works for any Python MCP server.
- **CBS ETL pattern** — The paginated OData fetcher with chunking and retry is reusable for any CBS dataset. DataPraat's validation ladder Step 5 ("validate against independent external data") could use this exact pipeline.

### What Needs Changing

- **Tool definitions** — Currently hardcoded for Jeugdzorg tables (FactVoorspelling, FactWerkelijk, etc.). DataPraat needs tools that query the master file, validation results, and lineage. New tools like `query_master_file`, `get_validation_report`, `trace_lineage`, `compare_deliveries`.
- **Database schema** — The star schema (fact + dimension tables) is Jeugdzorg-specific. DataPraat needs a schema driven by Part 2's master file structure, plus tables for validation results (Part 1) and lineage/audit trail (Part 2).
- **Chart hints** — The hint builders assume VVD column names (gemeente, productcategorie, Q, P, PxQ). Need generalization: detect column semantics from the master file schema and generate appropriate chart specs.
- **MCP resources** — `vvd://schema` and `vvd://glossary` are domain-specific. DataPraat needs dynamic resources that reflect the current client's data domain — generated from Part 1's profiling results and Part 2's column mappings.
- **CBS integration** — The CBS tools are Jeugdzorg-specific cross-analyses. Some CBS datasets (population, municipal finances) remain useful for DataPraat's validation ladder Step 5, but the specific tools need redesigning.

### What's Not Part of DataPraat

- **VVD-specific domain logic** — The 14 Jeugdzorg product categories, forecast versioning (Prognoselabel), indexation factors, confidence intervals. These are domain data, not platform features.
- **Legacy REST endpoints** — `/api/actuals` and `/api/actuals/detail`. MCP is the interface; REST endpoints were a stepping stone.
- **Baked-in DuckDB file in Docker** — DataPraat's database will be dynamic (Part 1/2 write to it), not a static file bundled at build time.
- **Excel export script** — DataPraat will need export functionality, but it should be an MCP tool or API endpoint, not a standalone script.

---

## What's Been Built — Part 1 (Validate & Understand)

> *To be written — covers the Python + FastAPI validation engine, the 48 MVP checks, and the report output format.*

---

## What's Been Built — Part 2 (Transform & Trace)

> *To be written — covers the transformation pipeline, lineage tracking, audit trail, and master file generation.*

---

## Documents in This Repository

| Document | Purpose |
|----------|---------|
| **PRODUCT_VISION.md** | This file — what DataPraat is and why |
| **VALIDATION_FRAMEWORK_COMPLETE_EN.md** | All 76 checks, MVP/Later markings, JSON report structure (English) |
| **VALIDATIE_FRAMEWORK_COMPLEET.md** | Same as above, in Dutch |
| **IMPLEMENTATION_PLAN.md** | Technical architecture and build plan for Part 1 backend |
| **HARM-NICO-DATA-PHILOSOPHY.md** | Data philosophy: validation ladder, norms, deviations |
| **harm-nico-vision-to-features.md** | Vision → feature mapping |
| **20260804-initial-file-sanity-check-thom (1).md** | Thom's real-world workflow |

---

## Architecture Decision: Language, Repo Structure, and Deployment

### Why Python for the backend

DataPraat is a data validation and transformation platform. The core work — parsing CSV/XML/Excel, profiling columns, detecting outliers, running 48+ validation checks, merging sources — lives squarely in Python's ecosystem. There is no comparable ecosystem in TypeScript:

| Task | Python | TypeScript |
|------|--------|-----------|
| CSV/Excel parsing | pandas, polars, openpyxl | SheetJS (weaker) |
| Column profiling | pandas `.describe()`, scipy | manual, painful |
| Statistical outliers | scipy, sklearn | basically nothing |
| Dutch number formats | easy string ops + locale | same effort |
| DuckDB | native, fast, mature | exists but less mature |
| XML parsing | lxml (battle-tested) | fast-xml-parser (fine) |
| MCP server | FastMCP (Python-native) | MCP SDK (exists, fine) |

Building 48 data validation checks in TypeScript would take roughly 3x as long and fight the ecosystem the whole way. Python is the only sane choice for Part 1 and Part 2.

The frontend (Part 3's UI) is already built in Next.js/TypeScript, and that's the right tool for that job. No reason to rewrite it.

### Why not all TypeScript?

- **Gain:** one language, simpler CI, Next.js API routes handle everything
- **Lose:** the entire Python data ecosystem; weeks rebuilding what pandas/scipy give you for free
- **Verdict:** not worth it for a data validation platform

### Why not microservices (3+ repos)?

- **Gain:** independent deployability per service
- **Lose:** shared types, shared DB layer, simple local dev
- **Verdict:** at current scale (1-2 devs, MVP), unnecessary overhead. One backend process is simpler and faster to ship.

### Target architecture: 2 services, 1 monorepo

DataPraat needs exactly **two services** — a Python backend and a Next.js frontend — sharing one DuckDB database file:

```
┌─────────────────────────────────┐   ┌──────────────────────────┐
│  Python backend (one process)   │   │  Next.js frontend        │
│                                 │   │  (already built)         │
│  FastAPI + FastMCP on Starlette │   │                          │
│                                 │   │  Chat UI ──── MCP ──────┤──→ /mcp
│  Part 1: /api/validate          │   │  Reports UI ── REST ────┤──→ /api/validate
│  Part 2: /api/transform         │   │  Upload UI ── REST ─────┤──→ /api/upload
│  Part 3: /mcp (tools for AI)   │   │  Charts ← chart hints   │
│                                 │   │                          │
│  Shared: DuckDB, models, schema │   │                          │
└─────────────────────────────────┘   └──────────────────────────┘
```

**One Python process** handles all three Parts on the backend:
- **Part 1** — FastAPI REST endpoints for file upload, validation, profiling, report generation
- **Part 2** — FastAPI REST endpoints for transformation, lineage tracking, master file management
- **Part 3** — FastMCP endpoints for AI tool calls against the master file, validation results, and lineage

All three share the same DuckDB database, the same Pydantic models, and the same schema definitions. FastAPI and FastMCP both run on Starlette, so they compose into a single ASGI app — **this is already how vvd-viewer works** (REST endpoints + MCP on the same Starlette process).

**One Next.js frontend** handles all UI:
- Chat interface (Part 3) — talks to `/mcp` via Vercel AI SDK's MCP client
- Validation reports (Part 1) — talks to `/api/validate` via REST
- Transformation UI (Part 2) — talks to `/api/transform` via REST
- File upload — triggers Part 1 validation pipeline, shows report

### Monorepo structure

```
datapraat/
├── packages/
│   ├── backend/          ← Python: FastAPI + FastMCP + DuckDB
│   │   ├── src/
│   │   │   ├── validation/    ← Part 1: 48 checks, profiling, reports
│   │   │   ├── transform/     ← Part 2: merge, lineage, audit trail
│   │   │   ├── mcp/           ← Part 3: MCP tools for AI queries
│   │   │   ├── shared/        ← DB layer, models, schema, chart hints
│   │   │   └── server.py      ← Single ASGI entrypoint
│   │   ├── scripts/           ← CBS ETL, export utilities
│   │   ├── pyproject.toml
│   │   └── Dockerfile
│   └── frontend/         ← Next.js: chat, reports, upload UI
│       ├── src/
│       ├── package.json
│       └── Dockerfile
├── docker-compose.yml     ← Local dev: both services + shared volume
└── README.md
```

### Why this is the fastest path

1. **Reuse, don't rebuild.** The MCP server pattern (vvd-viewer) and the chat frontend (vibathon-knowledgegraph) are proven. Adapt, don't rewrite.
2. **One database.** DuckDB is embedded — no database server to deploy. Part 1 writes validation results, Part 2 writes the master file, Part 3 reads it all. One file.
3. **One backend process.** No inter-service communication, no message queues, no API gateways. FastAPI + FastMCP on one Starlette app.
4. **Two deployment targets.** Python backend on Railway (or any Docker host). Next.js frontend on Vercel (or same host). That's it.
