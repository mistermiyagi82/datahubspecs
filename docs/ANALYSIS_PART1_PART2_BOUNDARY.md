# Analysis: Part 1 & Part 2 Boundary + Data Storage Strategy

Based on meeting transcripts with Thom (10 Apr 2026) and Marloes (9 Apr 2026), and the existing spec documents.

---

## What was discussed

### Meeting with Thom (10 Apr, Part 1&2 discussion)

**The 7-step framework debate:** Daan showed Thom the validation framework (7 phases, 76 checks). Thom's key insight is that the boundary between Part 1 and Part 2 is blurry. Specifically:

- **Steps 1-2** (File Check + Structure Recognition) are clearly Part 1 — pure read-only observation. "Can we open it? Is it structured correctly?"
- **Step 3** (Descriptive Statistics) starts profiling — still read-only, still Part 1.
- **Steps 4-7** (Content Validation, Consistency, Statistical, AI Analysis) — this is where Thom pushes back. He argues that to meaningfully validate content, you already need **context from the client**. A column with mixed numeric/text values looks wrong to a machine, but a domain expert knows product codes can be alphanumeric. That context-gathering conversation is essentially Part 2 work (building the master file's metadata layer).

**The context capture problem:** Daan and Thom agree on the core issue — the data itself is "worthless" without context (echoing Harm Nico). The key discussion was *how* to capture that context:

1. **Knowledge graph** — Daan proposed this. Thom liked the Obsidian-style linked knowledge idea. Unstructured data goes in, domain-specific knowledge base builds up iteratively over time.
2. **Recording client conversations** — Daan proposed recording meetings where Thom walks through the validation report with the client. Thom pushed back slightly — said clients would accept a chat-style interface (ChatGPT-like) before they'd accept voice recording. His counter-proposal: **AI generates the data quality analysis, Thom discusses it with client in a meeting, that meeting gets recorded, and the transcript feeds context back into the system.** Daan loved this — it gives structured context because Thom prepares discussion points from the report.
3. **Data must be relational, context can be fuzzy** — Critical distinction they both made. The actual data values (a 2 must be a 2, not "close to 2") need relational storage. But context and metadata (why a column is empty, what a product code means) can live in embeddings or knowledge graphs. You can't use embeddings for financial data. You can for context.

**VDD's existing product:** Thom revealed VDD already sells a "data quality scan" product — essentially what Part 1 does manually. This validates the commercial angle for Part 1 standalone.

### Meeting with Marloes (9 Apr, End user perspective)

**User personas:** Marloes confirmed four user types for the Jeugdzorg domain:
- **Wethouders** (aldermen) — want explanations and talking points for council meetings
- **Controllers** — want hard numbers, KPIs, cost tracking
- **Beleidsmedewerkers** (policy staff) — want to predict and explain trends
- **Data-analisten** — want detailed data access

**Two steering axes:** Marloes distinguished:
1. **Costs & care usage** — measurable with KPIs, relatively straightforward
2. **Societal impact / citizen behavior** — hard to measure, hypothetical, long-term

**Scenario analysis:** Clients want "what-if" analysis — "if we turn this policy knob, what happens?" This is currently tied to Thom's forecast model. This confirms Part 3 value but also means Parts 1 & 2 need to capture enough context for meaningful scenarios.

**CBS data caveat:** Marloes warned that CBS data rarely matches internal municipal data — a critical insight for validation ladder Step 5.

---

## Analysis: Part 1 vs Part 2 Boundary

Thom is right, and the current 7-step framework conflates two distinct activities:

**Part 1 should be: "What can we learn from this file without any human input?"**

That's Steps 1-3 of the framework, plus the purely mechanical parts of Steps 4-6 (format validation, type checking, statistical outlier detection). Everything the machine can do by staring at the file alone. The output is a **validation report** — observations, flags, questions to discuss with the client.

**Part 2 should be: "What do we learn by discussing this file with a human?"**

This is where context enters. The validation report from Part 1 generates **discussion points**. Thom reviews them with the client. That conversation (recorded, transcribed, or entered via chat) produces **context** that resolves the ambiguities: "Yes, that column is intentionally sparse", "No, that 90% drop is a data error", "Product codes are alphanumeric, that's normal."

Part 2 then also covers the actual transformation: column mapping, merging sources, building the master file — but now *informed by* the context gathered through the Part 1 report discussion.

**The clean boundary:**

| | Part 1 | Part 2 |
|---|---|---|
| **Input** | Raw file | File + validation report + human context |
| **Activity** | Automated checks, profiling | Context capture, transformation, lineage |
| **Output** | Validation report + discussion points | Master file + audit trail + knowledge base |
| **Human role** | Reviews report | Provides context, approves transformations |
| **Data touched?** | Never (read-only) | Yes (transforms, merges) |

This means the 7-step framework is actually a **Part 1 pipeline** — but the *interpretation* of results (Steps 4-7 flags becoming resolved context) is Part 2 work.

---

## Analysis: Data Storage Strategy

Three categories of information with fundamentally different storage needs:

### 1. Hard data (the actual values) — Relational / DuckDB

As Daan said: "a 2 must be a 2." Financial data, client counts, costs, dates — these go into a proper schema. Star schema in DuckDB is perfect. This is what dashboards and Part 3 query. No embeddings, no fuzziness.

### 2. Validation metadata (what the checks found) — Structured JSON / relational

Check results, column profiles, data types, fill rates, outlier scores. This is machine-generated, deterministic, and structured. Store as JSON reports attached to each file upload, or as rows in a `validation_results` table. This feeds the Part 1 report UI and Part 3's AI context.

### 3. Context (what humans told us about the data) — Layered approach

Rather than picking one storage paradigm, use layers:

**Layer A: Structured context (relational)**
Things that map cleanly to the data schema:
- Column annotations: "column X is alphanumeric product codes, not mixed types"
- Known exceptions: "client Y always delivers late, Q3 data may be incomplete"
- Approval status: "Thom approved this file on 2026-04-09"
- Lineage: source file -> source row -> transformation -> master file row

This goes in DuckDB alongside the hard data. It's queryable, exact, and versioned.

**Layer B: Unstructured context (knowledge graph + embeddings)**
Things that are conversational, domain-specific, cross-client:
- Meeting transcripts and summaries
- Domain rules: "in Jeugdzorg, a 'traject' can span multiple 'producten'"
- Client-specific knowledge: "Terneuzen changed their export system in Q2 2026"
- Norms: "municipalities this size typically spend X EUR per capita"

This is where a **knowledge graph** makes sense. Not a complex graph database — SQLite with a `facts` table (entity, relationship, entity, source, timestamp) is enough. The vibathon-knowledgegraph codebase already has this pattern. Combined with embeddings for semantic search.

**Layer C: Audit trail (append-only log)**
Everything that happened, in order:
- File uploaded at T1 by user X
- Validation ran, 3 warnings, 1 error
- Meeting with client at T2, transcript attached
- Context added: "column Y is intentionally sparse (reason: ...)"
- Transformation applied: rename column A -> B

This is an append-only event log. Simple table: `(id, timestamp, actor, action, target, details_json)`. Never deleted, never modified. This is the "where does this number come from?" answer chain.

### Why not just a knowledge graph for everything?

Knowledge graphs are great for fuzzy, interconnected context. They're terrible for exact financial data. And they're overkill for structured validation results. Use the right tool for each layer.

### Why not a canonical database for everything?

Because some context is inherently unstructured (meeting transcripts, domain knowledge) and you need semantic search to retrieve it. A relational schema can't handle "find me everything we know about how Terneuzen handles their product codes."

---

*Generated: 2026-04-10*
*Sources: Meeting Thom & Daan (10 Apr), Meeting Marloes & Daan (9 Apr), PRODUCT_VISION.md, VALIDATION_FRAMEWORK_COMPLETE_EN.md, HARM-NICO-DATA-PHILOSOPHY.md*
