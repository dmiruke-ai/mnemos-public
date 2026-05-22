# MCP Tools

Mnemos exposes **27 typed tools** over the Model Context Protocol (MCP).
Any MCP-aware client — Claude Desktop, Cursor, Continue, custom agents —
can call them. The tools cover memory CRUD, hybrid retrieval, knowledge
graph navigation, code-graph traversal, and ingestion.

This document lists every tool with its input schema and a one-line
description of what it returns. For the live drift-verification matrix
(which tools currently pass against the running deployment), see the
private repo's `mcp/TOOL_STATUS.md`.

## Configuration

### Claude Desktop

Add to `~/Library/Application Support/Claude/claude_desktop_config.json`
(macOS) or `%APPDATA%\Claude\claude_desktop_config.json` (Windows):

```json
{
  "mcpServers": {
    "mnemos": {
      "command": "python",
      "args": ["-m", "mcp.server"],
      "cwd": "/path/to/mnemos",
      "env": {
        "DATABASE_URL": "postgresql://mnemos:mnemos@mnemos.dmiruke.dev:5434/mnemos",
        "OLLAMA_URL": "http://mnemos.dmiruke.dev:11434"
      }
    }
  }
}
```

For the hosted demo, contact the author for a remote MCP endpoint URL
(SSE mode).

### Cursor

Add to `~/.cursor/mcp.json` with the same shape.

---

## Memory & retrieval

### `semantic_search`
**Use when:** finding conceptually-related memories.
**Args:** `query` (required), `limit` (default 10), `memory_type`, `tags`.
**Returns:** list of matches with `Breadcrumb: <doc> > <section> > p.<N>` and `Anchor: {...}` for page-tree memories.

### `hybrid_search`
**Use when:** you want vector + keyword + recency weighting tuned.
**Args:** `query`, `vector_weight` (default 0.5), `keyword_weight` (0.3), `recency_weight` (0.2), `memory_type`, `tags`, `limit`.
**Returns:** ranked results with score decomposition + breadcrumb + anchor.

### `get_context_bundle`
**Use when:** assembling the full retrieval context for an LLM prompt.
**Args:** `query` (required), `max_memories` (default 10), `max_entities` (20), `include_relationships` (true).
**Returns:** memories + entities + relationships in one envelope, deduplicated and ranked.

### `list_memories`
**Use when:** browsing a typed slice.
**Args:** `memory_type`, `tags`, `limit`, `offset`.

### `list_recent`
**Use when:** "what happened in the last 24h?"
**Args:** `timeframe` (`1h | 24h | 7d | 30d`), `memory_type`, `limit`.

### `get_stats`
**Use when:** operational rollup.
**Args:** `period_days` (default 7).
**Returns:** total memories, success rate, failed, duplicates, avg latency, retrieval stats.

### `decision_trace`
**Use when:** reconstructing a decision with alternatives + rationale.
**Args:** `decision_id` or `memory_id` (one required).

### `weekly_summary`
**Use when:** auto-generated weekly digest.
**Args:** `period_days` (default 7), `generate_new` (default false).
**Returns:** themes, action items, key decisions, executive summary.

---

## Ingest

### `ingest_memory`
**Use when:** adding a raw memory from inside an agent.
**Args:** `content` (required), `memory_type` (default `episodic`), `title`, `tags`, `source`.

### `ingest_pdf`
**Use when:** adding a PDF document — routed through the **page-tree builder** so chunks land with `document_nodes` populated (Breadcrumb + Anchor become available on the first semantic_search after embedding).
**Args:** `file_path` (server-side; required), `tags`, `per_page` (default true).
**Returns:** node count, chunk count, term count, page count.

### `ingest_image`
**Use when:** adding an image with OCR (requires tesseract on the host).
**Args:** `file_path`, `tags`, `language` (default `eng`).

---

## Knowledge graph (concepts + entities)

### `get_entity_graph`
**Use when:** navigating concepts, people, organisations as nodes.
**Args:** `action` (`get_entity | list_entities | get_relationships | search`; **optional** — auto-inferred from other args), `entity_id`, `entity_type`, `query`, `limit`.

### `extract_concepts`
**Use when:** pulling key concepts from text on-the-fly.
**Args:** `text` (required), `memory_id` (optional; if given, persists).

### `get_concept_clusters`
**Use when:** discovering thematic groupings in the corpus.
**Args:** `n_clusters` (default 5), `format` (`text | mermaid | json`).

---

## Code graph (Sprint D-bis) ⭐

### `find_symbol`
**Use when:** "where is `authorize_payment` defined?"
**Args:** `prefix` (required), `codebase`, `kind` (`function | method | class | module | type | interface | struct | trait`), `language`, `limit` (default 20).
**Returns:** list of symbols with qualified_name, file path, line, signature.

### `who_calls`
**Use when:** impact analysis — "what breaks if I change this?"
**Args:** `symbol` (required — name or qualified_name), `codebase`, `kind` (default `calls`; can be `imports | extends | decorated-by`), `limit` (default 25).
**Returns:** list of callers with file:line.

### `get_symbol_callgraph`
**Use when:** dependency analysis — "what does this function depend on?"
**Args:** `symbol`, `codebase`, `kind`, `limit`.
**Returns:** outgoing edges with target symbol + line.

### `get_infra_resources`
**Use when:** "what AWS resources does this codebase provision?", "show me the GH-Actions workflows", "which k8s deployments use this ConfigMap?"
**Args:** `codebase` (required), `source_kind` (`terraform | kubernetes | github-actions | dockerfile`), `resource_kind` (e.g. `aws_instance`, `Deployment`, `workflow`), `limit` (default 50).
**Returns:** list of resources with qualified_name, file path, depends_on edges.

---

## Code search (Layer 2 + B)

### `search_codebase`
**Use when:** semantic search over code content.
**Args:** `query` (required), `codebase`, `language`, `limit`.

### `find_implementations`
**Use when:** find a function/class by name (no graph walk).
**Args:** `name` (required), `type` (`function | class | method`), `codebase`.

### `find_imports`
**Use when:** "which files import psycopg2?"
**Args:** `import_name` (required), `codebase`.

### `list_codebases`
**Args:** `limit`.
**Returns:** every ingested codebase with file count + languages.

### `get_codebase_structure`
**Use when:** tree-view of a single codebase.
**Args:** `codebase` (required), `path_pattern`.

---

## Programming patterns (pattern library)

A separate corpus of canonical algorithms / data structures / idioms,
ingested separately:

### `search_patterns`
**Args:** `query` (required), `category` (`algorithmic | data_structure | language_construct | concurrency | functional | system_design`), `language`, `limit`.

### `get_pattern`
**Args:** `slug` (required, e.g. `two-pointers`), `language`.

### `get_pattern_path`
**Use when:** learning path with prerequisites + extensions.
**Args:** `slug`, `depth` (default 2).

### `compare_implementations`
**Args:** `slug`, `languages` (array).

---

## Tool count summary

| Bucket | Tools |
|---|---|
| Memory & retrieval | 8 |
| Ingest | 3 |
| Knowledge graph (concepts/entities) | 3 |
| **Code graph (D-bis)** | **4** |
| Code search | 5 |
| Programming patterns | 4 |
| **Total** | **27** |

## Drift-verification

Every tool is verified against the live schema by
`scripts/mcp_drift_verify.py` (in the private repo). Current pass rate:
**25/27** — the 2 fails are environmental (`decision_trace` returns
"Decision not found" with `is_error=True` on a fake UUID — correct
semantics; `ingest_image` needs tesseract on the host).

## What's coming (Sprint E)

7 new tools tied to the Retrieval Planner v1:

- `get_outline(doc_id)` — table of contents from `document_nodes`
- `get_page(doc_id, page_num)` — fetch a specific page
- `get_breadcrumb(memory_id)` — explicit breadcrumb lookup
- `get_neighbors(node_id, depth)` — walk the page-tree neighborhood
- `lookup(term, doc_id?)` — back-of-book index lookup via `document_terms`
- `get_document_index(doc_id)` — full back-of-book for one doc
- `search_within(doc_id, query)` — search scoped to one doc

Brings the surface to **34 tools** total.
