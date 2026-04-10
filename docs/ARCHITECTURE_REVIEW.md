# Architecture Review: Does the Original Strategy Still Hold?

After revisiting all spec documents, meeting transcripts, and the storage analysis, here's what still works, what needs updating, and what was wrong.

---

## What still holds

### Python backend + Next.js frontend = correct

The two-service split is right. Python for data processing (validation, transformation, profiling) and Next.js for the UI (chat, reports, upload). The ecosystem argument in PRODUCT_VISION.md is solid — pandas, scipy, polars, openpyxl have no TypeScript equivalents worth using.

### FastAPI + FastMCP on one Starlette process = correct

One backend process serving REST (Parts 1 & 2) and MCP (Part 3) is the right call. It's proven in vvd-viewer, avoids inter-service complexity, and keeps the deployment simple. No reason to change this.

### Pluggable check architecture = correct

The IMPLEMENTATION_PLAN.md design (BaseCheck ABC, auto-discovery via pkgutil, `c{phase}{check}_{name}.py` naming) is clean and extensible. 48 MVP checks across 7 phases, growing to 76. This design is independent of the database choice and doesn't need changing.

### Monorepo = correct (for now)

At 1-2 devs building an MVP, a monorepo with `packages/backend/` and `packages/frontend/` is right. Microservices would be premature.

### MCP as the boundary between data and AI = correct

The frontend never touches the database. All data access goes through MCP tools or REST endpoints. This decoupling is one of the best decisions in the architecture.

### Reuse strategy = correct

Adapting vvd-viewer (MCP server pattern) and vibathon-knowledgegraph (chat UI) rather than rebuilding from scratch is the fastest path. The patterns are proven.

---

## What needs updating

### 1. DuckDB as the single database -> PostgreSQL

**Original decision:** "One DuckDB file shared between all three Parts."

**Why it was chosen:** Zero deployment (embedded), proven in vvd-viewer, fast analytical queries.

**Why it no longer fits:**

DuckDB was the right choice for vvd-viewer — a read-only analytical dashboard with static data baked into Docker. DataPraat is fundamentally different:

| Requirement | DuckDB fit | PostgreSQL fit |
|---|---|---|
| Validation results (frequent small writes) | Poor (columnar, OLAP-optimized) | Good (row-oriented, OLTP) |
| Audit trail (append-only, write-heavy) | Poor | Good |
| Lineage tracking (foreign keys, joins by ID) | Awkward | Natural |
| Context annotations (per-column, per-file) | Workable | Better |
| Concurrent users (Thom + analysts + clients) | Single writer | Full concurrency |
| Analytical queries for Part 3 / dashboards | Excellent | Good enough at this scale |
| Master file data (financial, exact) | Good | Good |
| Deployment | Zero (embedded file) | Needs managed service (~$0-7/mo) |

The single-writer limitation is the dealbreaker. DataPraat writes constantly: file uploads, validation results, context from meetings, transformation logs, audit events. Even with a small team, two people uploading files simultaneously would block each other.

**Recommendation:** PostgreSQL as the single database. Use a managed service (Supabase free tier, Neon free tier, or Railway at ~$7/mo). All three storage layers (hard data, validation metadata, context + audit trail) live in one database with proper schemas.

**Migration cost:** Low. The backend doesn't exist yet — no code to rewrite. The only existing DuckDB code is in vvd-viewer, which stays separate. The FastAPI backend in IMPLEMENTATION_PLAN.md has no database code yet; it's just a validation pipeline that returns JSON.

**What about DuckDB's analytical strengths?** At DataPraat's scale (13 municipalities, thousands of rows per delivery, maybe tens of thousands in the master file), PostgreSQL handles analytical queries fine. If analytical performance ever becomes a bottleneck (it won't for years), you can add DuckDB as a read-only analytical layer that syncs from PostgreSQL. Don't optimize for that now.

### 2. The Part 1 / Part 2 boundary needs redrawing

**Original model (from PRODUCT_VISION.md):**
- Part 1 = Validate & Understand (7-phase validation pipeline, 76 checks)
- Part 2 = Transform & Trace (ETL, lineage, audit trail, master file)
- Part 3 = Talk to Your Data (AI + MCP)

**Thom's challenge:** The 7-step validation framework mixes pure automated observation (Steps 1-3) with checks that require human context to interpret (Steps 4-7). A column with mixed types gets flagged as an error in Step 4, but Thom knows product codes are alphanumeric. Without that context, the validation report generates false positives that erode trust.

**Revised model:**

| | Part 1: Observe | Part 2: Contextualize & Transform |
|---|---|---|
| **What happens** | Automated checks run on the raw file. No human input needed. | Human reviews report, provides context. Transformations applied. |
| **7-step mapping** | Steps 1-3 fully. Steps 4-7 run but results are _questions_, not _verdicts_. | Steps 4-7 findings get resolved: confirmed, dismissed, or annotated with context. |
| **Output** | Validation report + discussion points | Master file + audit trail + knowledge base |
| **Data touched?** | Never (read-only) | Yes |
| **Who acts** | System | Thom (with client) |

**Key insight:** Steps 4-7 still _run_ in Part 1 — the statistical outlier detection, the AI semantic analysis, etc. But their output is framed as **questions to discuss with the client**, not as pass/fail verdicts. Part 2 is where those questions get answered and the answers become context.

This doesn't change the validation engine architecture (IMPLEMENTATION_PLAN.md is still correct). It changes how the _output_ is presented and what happens next.

### 3. Storage needs a proper schema design, not "just DuckDB"

**Original:** "One DuckDB file. Part 1 writes, Part 2 writes, Part 3 reads."

**Revised:** PostgreSQL with distinct schemas for different concerns:

```
datapraat (PostgreSQL database)
├── uploads/              -- file metadata, raw storage reference
│   ├── files             -- id, filename, format, size, uploaded_by, uploaded_at
│   └── file_contents     -- raw content or S3/disk reference
│
├── validation/           -- Part 1 output
│   ├── reports           -- id, file_id, status, created_at
│   ├── phase_results     -- report_id, phase, status, summary
│   └── findings          -- phase_result_id, check_id, severity, message, details (JSONB)
│
├── context/              -- Part 2: human-provided context
│   ├── annotations       -- file_id, column_name, note, author, created_at
│   ├── meeting_notes     -- file_id, transcript, summary, created_at
│   ├── resolutions       -- finding_id, status (confirmed/dismissed/annotated), reason, resolved_by
│   └── domain_knowledge  -- entity, fact, source, created_at (knowledge graph as rows)
│
├── master/               -- Part 2: transformed data
│   ├── datasets          -- id, name, description, created_at
│   ├── columns           -- dataset_id, name, type, source_file, source_column
│   ├── rows              -- dataset_id, data (JSONB), source_file_id, source_row_number
│   └── transformations   -- id, dataset_id, type, from_state, to_state, reason, applied_by
│
├── lineage/              -- Part 2: traceability
│   ├── cell_lineage      -- master_dataset_id, row_id, column_name, source_file_id, source_row, source_column
│   └── confidence        -- master_dataset_id, row_id, column_name, validation_steps_passed, score
│
└── audit/                -- Layer C: append-only event log
    └── events            -- id, timestamp, actor, action, target, details (JSONB), session_id
```

This is **one database** with clear boundaries. Every table serves a specific purpose. Lineage is tracked via foreign keys (click a number -> follow the chain). The audit log captures everything.

### 4. Knowledge graph approach needs simplification

**Original proposal (ANALYSIS_PART1_PART2_BOUNDARY.md):** SQLite knowledge graph with entity/relationship/entity triples + embeddings for semantic search.

**Revised:** The `context.domain_knowledge` table in PostgreSQL handles the structured facts. For semantic search across meeting transcripts and domain knowledge, use **pgvector** (PostgreSQL extension) instead of a separate SQLite + embedding store. One database, one query interface.

This eliminates the "three databases" problem (DuckDB + SQLite + knowledge graph) and replaces it with one PostgreSQL instance that handles everything. pgvector is mature, widely supported on managed services (Supabase and Neon both include it), and avoids the operational complexity of a separate embedding store.

For the knowledge graph specifically: simple rows in `domain_knowledge` (entity, relationship, target_entity, source, timestamp) are sufficient. No need for Neo4j or a graph database. Graph queries ("what do we know about Terneuzen's product codes?") become simple SQL filters. If you need traversal later, PostgreSQL's recursive CTEs handle it.

### 5. Monorepo structure needs a minor update

**Original:**
```
datapraat/
├── packages/
│   ├── backend/          ← Python: FastAPI + FastMCP + DuckDB
│   └── frontend/         ← Next.js
├── docker-compose.yml
└── README.md
```

**Revised:**
```
datapraat/
├── packages/
│   ├── backend/          ← Python: FastAPI + FastMCP + PostgreSQL
│   │   ├── src/
│   │   │   ├── validation/    ← Part 1: checks, profiling, reports
│   │   │   ├── context/       ← Part 2a: context capture, annotations, resolutions
│   │   │   ├── transform/     ← Part 2b: merge, lineage, master file
│   │   │   ├── mcp/           ← Part 3: MCP tools for AI queries
│   │   │   ├── db/            ← SQLAlchemy/asyncpg, migrations (Alembic)
│   │   │   └── server.py      ← Single ASGI entrypoint
│   │   ├── migrations/        ← Alembic migration scripts
│   │   ├── pyproject.toml
│   │   └── Dockerfile
│   └── frontend/         ← Next.js: chat, reports, upload UI
│       ├── src/
│       ├── package.json
│       └── Dockerfile
├── docker-compose.yml     ← backend + frontend + postgres
└── README.md
```

Key changes:
- `DuckDB` -> `PostgreSQL` in the backend
- Added `context/` module (Part 2a: the context capture layer, separate from transformation)
- Added `db/` module with migrations (Alembic) — proper schema management
- Added `migrations/` directory
- docker-compose now includes a PostgreSQL service

---

## What was wrong

### "One DuckDB file" was a false simplification

The original architecture sold DuckDB's embedded nature as a feature: "no database server to deploy." But this traded deployment simplicity for architectural limitations. A managed PostgreSQL instance on Supabase/Neon is just as easy to deploy (one URL in an env var) and removes the single-writer bottleneck, the OLTP performance concern, and the "how do we do migrations" question.

### The IMPLEMENTATION_PLAN.md is isolated from the storage question

The validation engine plan (IMPLEMENTATION_PLAN.md) correctly treats the database as an afterthought — the pipeline produces a JSON report, and where it gets stored is a separate concern. This is actually good: the check architecture is database-agnostic. But it means the plan doesn't address where validation results, context, and lineage live. That gap needs filling.

### The Part 1/2/3 split implied sequential development

The original vision says "Part 1 first, then Part 2, then Part 3." But from Thom's meeting, it's clear that Part 1's validation report is only valuable if Part 2's context capture mechanism exists to act on it. A validation report full of unresolved questions is just noise. The implication: **Part 1 and Part 2a (context capture) should be built together.** The transformation/lineage parts of Part 2 can come later.

Suggested build order:
1. Part 1 validation engine + Part 2a context capture (the discussion/annotation layer)
2. Part 2b transformation pipeline + lineage tracking
3. Part 3 AI layer (adapt existing vvd-viewer + vibathon-knowledgegraph)

---

## Summary of changes

| Decision | Original | Revised | Why |
|---|---|---|---|
| Database | DuckDB (embedded) | PostgreSQL (managed) | Write-heavy workload, concurrency, lineage needs FK/joins |
| Storage layers | DuckDB + SQLite + knowledge graph | PostgreSQL only (+ pgvector for embeddings) | One database, one query interface, less operational complexity |
| Part 1/2 boundary | 7 steps = Part 1, transform = Part 2 | Steps 1-3 + flagging = Part 1, context + transform = Part 2 | Thom's feedback: interpretation requires human context |
| Build order | Part 1 -> Part 2 -> Part 3 | Part 1 + Part 2a -> Part 2b -> Part 3 | Validation report needs context capture to be useful |
| Monorepo structure | validation/ + transform/ + mcp/ | validation/ + context/ + transform/ + mcp/ + db/ | Context capture is its own concern |
| Unchanged | Python + FastAPI + FastMCP, Next.js frontend, MCP boundary, pluggable checks, monorepo, reuse strategy | | These decisions are sound |

---

*Generated: 2026-04-10*
*Based on: PRODUCT_VISION.md, IMPLEMENTATION_PLAN.md, VALIDATION_FRAMEWORK_COMPLETE_EN.md, HARM-NICO-DATA-PHILOSOPHY.md, harm-nico-vision-to-features.md, 20260804-initial-file-sanity-check-thom.md, Meeting Thom & Daan, Meeting Marloes & Daan, ANALYSIS_PART1_PART2_BOUNDARY.md*
