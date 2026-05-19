> **PATENT PENDING** — Technology and methodology patent pending. All rights reserved.

[![Live API](https://img.shields.io/badge/Live%20API-Swagger%20UI-b48cff?style=for-the-badge)](http://37.27.97.75:8200/docs)
[![Dashboard](https://img.shields.io/badge/Dashboard-Mnemos%20UI-5fc77a?style=for-the-badge)](http://37.27.97.75:3002/)
[![Grafana](https://img.shields.io/badge/Observability-Grafana-f59e0b?style=for-the-badge)](http://37.27.97.75:10210/d/mnemos-overview/)
[![MCP](https://img.shields.io/badge/MCP-27%20tools-0ea5e9?style=for-the-badge)](docs/MCP_TOOLS.md)

# Mnemos

**Persistent agentic memory with provenance.**

Mnemos is an AI-accessible memory system that treats every stored fact as a
positioned, typed unit of knowledge — not an opaque embedded chunk. Documents
keep their **page tree** so retrieval results carry `Doc > §3 > p.5` style
breadcrumbs. Codebases are ingested as a **5-layer semantic graph**
(structural / AST / runtime-ops / semantic / relationship), not as flat
RAG. The system surfaces over REST (FastAPI), MCP (for Claude Desktop,
Cursor, and any MCP-aware agent), and a Next.js dashboard.

---

## Live access — no installation

| Surface | URL | Notes |
|---|---|---|
| **Dashboard** | [http://37.27.97.75:3002/](http://37.27.97.75:3002/) | Memories · Search · Graph · Codebases · Uploads · Imports |
| **REST API (Swagger)** | [http://37.27.97.75:8200/docs](http://37.27.97.75:8200/docs) | 64 endpoints, interactive try-it-out |
| **OpenAPI spec** | [http://37.27.97.75:8200/openapi.json](http://37.27.97.75:8200/openapi.json) | Machine-readable schema |
| **Health probe** | [http://37.27.97.75:8200/health](http://37.27.97.75:8200/health) | JSON `{"status":"healthy", ...}` |
| **Prometheus** | [http://37.27.97.75:9091/](http://37.27.97.75:9091/) | 36 custom series + Python runtime |
| **Grafana** | [http://37.27.97.75:10210/d/mnemos-overview/](http://37.27.97.75:10210/d/mnemos-overview/) | 13-panel "Mnemos — Overview" dashboard (anonymous read; admin `admin/mnemos`) |

> Hosted on a single-node bare-metal deployment for the public demo. Production
> deployment patterns (multi-replica API, Postgres HA, S3-backed media, multi-tenant
> scoping) are in the design docs.

---

## Quick API tour

```bash
# 1. Capture a memory (auto-embedded, auto-tagged)
curl -s -X POST http://37.27.97.75:8200/api/memories \
  -H 'Content-Type: application/json' -H 'X-Tenant-ID: demo' \
  -d '{"content":"Mnemos uses pgvector for similarity search.",
       "tags":["mnemos","architecture"],
       "title":"vector substrate"}'

# 2. Semantic search — returns breadcrumb + anchor on page-tree memories
curl -s "http://37.27.97.75:8200/api/search?query=vector+similarity&limit=3" \
  -H 'X-Tenant-ID: demo' | jq .

# 3. Browse the executable code graph
curl -s "http://37.27.97.75:8200/api/codebases" | jq '.codebases[].name'
curl -s "http://37.27.97.75:8200/api/symbols?prefix=get_conn&limit=3" | jq .
curl -s "http://37.27.97.75:8200/api/symbols/<id>/called-by" | jq .

# 4. List operational/infrastructure resources extracted from a codebase
curl -s "http://37.27.97.75:8200/api/codebases/<id>/infra" | jq .

# 5. Job queue + observability
curl -s "http://37.27.97.75:8200/admin/embedding/queue"
curl -s "http://37.27.97.75:8200/metrics" | head -20   # Prometheus format
```

---

## What's distinctive

| | RAG-over-files | **Mnemos** |
|---|---|---|
| Provenance | snippet only | **Breadcrumb tree** (Doc → Section → Page → Anchor) |
| Code representation | embedded text chunks | **5-layer semantic graph** (symbols + callgraph + infra + relationships) |
| Retrieval substrates | one (vector) | **three** — symbol trie · vector · graph traversal |
| Agent surface | none | **27 MCP tools** wired for Claude Desktop, Cursor, any MCP client |
| Observability | none | **Prometheus + Grafana** — request rate, search latency, embedding queue lag, DB pool, ingest throughput |
| Background reflection | none | **Smriti** workers (deferred) — contradictions, derived concepts, importance, long-term summaries |

---

## Architecture

![Mnemos architecture](docs/architecture.svg)

Surfaces (Dashboard / API / MCP) hit a FastAPI gateway. Ingest dispatches
by source type to per-source **map builders** (`pdf`, `markdown`, `code`,
plus runtime-ops parsers for `terraform`, `kubernetes`, `github-actions`,
`dockerfile`). Writers persist into typed substrates: `memories` for
content, `document_nodes`/`document_terms` for the page tree,
`code_symbols`/`code_relationships` for the executable code graph,
`infra_resources` for operational nodes. Three retrieval substrates
(symbol-trie via pg_trgm, vector via pgvector HNSW, graph via recursive
CTE) are fused at the retrieval boundary.

Read more in **[`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md)**.

---

## Documentation

- 📐 **[`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md)** — components, data flow, retrieval funnel, observability
- 📄 **[`docs/PRODUCT_WRITEUP.md`](docs/PRODUCT_WRITEUP.md)** — problem, solution, positioning, use cases
- 🧩 **[`docs/MEMORY_TAXONOMY.md`](docs/MEMORY_TAXONOMY.md)** — typed memory model (semantic / episodic / procedural / decision / document / image / audio / video)
- 🌳 **[`docs/CODE_INGESTION_5_LAYERS.md`](docs/CODE_INGESTION_5_LAYERS.md)** — structural / AST / runtime-ops / semantic / relationship
- 🔌 **[`docs/API_REFERENCE.md`](docs/API_REFERENCE.md)** — curated REST endpoint map
- 🛠 **[`docs/MCP_TOOLS.md`](docs/MCP_TOOLS.md)** — 27 MCP tools (find_symbol, who_calls, get_infra_resources, semantic_search, …)
- 📊 **[`docs/MONITORING.md`](docs/MONITORING.md)** — Prometheus + Grafana, what's instrumented and why

---

## License

See [LICENSE](LICENSE). Patent pending; all rights reserved. The hosted
demo above is free for evaluation. Contact the author before any
redistribution, derivative work, or commercial use.
