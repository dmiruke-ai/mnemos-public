# API Reference

The live API exposes 64 endpoints. The full OpenAPI 3 spec is at
[`http://mnemos.dmiruke.dev:8200/openapi.json`](http://mnemos.dmiruke.dev:8200/openapi.json);
the interactive Swagger UI is at
[`http://mnemos.dmiruke.dev:8200/docs`](http://mnemos.dmiruke.dev:8200/docs).

This doc curates the **interesting subset** — everything you'd actually
want to call from a script or an agent. All `/api/...` routes accept an
`X-Tenant-ID` header (defaults to `default`); the `/memory/...` and
`/api/keys/...` routes require `Authorization: Bearer mnm-...`.

## Conventions

- **Base URL:** `http://mnemos.dmiruke.dev:8200`
- **Tenant scoping:** every memory + node is tenant-scoped. Pass
  `X-Tenant-ID: <your-tenant>` on every call.
- **202 contract:** ingest paths that take >1s return `HTTP 202 Accepted`
  with `{"job_id": "..."}`. Poll `GET /api/jobs/{job_id}`.
- **Pagination:** `limit` + `offset` query params; max `limit=500` on
  most endpoints.
- **JSON in, JSON out** unless explicitly noted (multipart for uploads,
  Prom text for `/metrics`).

---

## Memory CRUD

### Create / browse / update / delete

```bash
# Create
curl -X POST http://mnemos.dmiruke.dev:8200/api/memories \
  -H 'X-Tenant-ID: demo' -H 'Content-Type: application/json' \
  -d '{
    "content": "Mnemos uses pgvector for similarity search.",
    "memory_type": "semantic",
    "title": "vector substrate",
    "tags": ["mnemos", "architecture"],
    "metadata": {"author": "team"}
  }'
# → {"id":"...","content":"...","node_type":"semantic","metadata":{...},...}

# Browse
curl "http://mnemos.dmiruke.dev:8200/api/memories?limit=10&offset=0" -H 'X-Tenant-ID: demo'

# Single
curl "http://mnemos.dmiruke.dev:8200/api/memories/<id>" -H 'X-Tenant-ID: demo'

# Update
curl -X PUT "http://mnemos.dmiruke.dev:8200/api/memories/<id>" \
  -H 'X-Tenant-ID: demo' -H 'Content-Type: application/json' \
  -d '{"content": "...updated...", "metadata": {"reviewed": true}}'

# Delete
curl -X DELETE "http://mnemos.dmiruke.dev:8200/api/memories/<id>" -H 'X-Tenant-ID: demo'
```

### Recent

```bash
curl "http://mnemos.dmiruke.dev:8200/memory/recent?limit=20" -H 'Authorization: Bearer mnm-...'
```

---

## Search

### Hybrid (vector + keyword + recency)

```bash
curl "http://mnemos.dmiruke.dev:8200/api/search?query=rate+limiter&limit=10" \
  -H 'X-Tenant-ID: demo'
```

Page-tree memories return with full `Breadcrumb` and `Anchor` enrichment
when invoked via MCP. The REST `/api/search` returns the raw hybrid
result; for breadcrumbs, use the MCP `semantic_search` tool or the
`/memory/search` endpoint.

### Vector-only / hybrid with custom weights

```bash
curl -X POST http://mnemos.dmiruke.dev:8200/memory/search \
  -H 'Authorization: Bearer mnm-...' -H 'Content-Type: application/json' \
  -d '{
    "query": "JWT validation",
    "limit": 10,
    "mode": "hybrid",
    "vector_weight": 0.6,
    "keyword_weight": 0.3,
    "recency_weight": 0.1,
    "filters": {"memory_type": "procedural", "tags": ["lang:python"]}
  }'
```

### Context bundle (memories + entities + relationships in one shot)

```bash
curl -X POST http://mnemos.dmiruke.dev:8200/memory/context \
  -H 'Authorization: Bearer mnm-...' -H 'Content-Type: application/json' \
  -d '{
    "query": "how do we authenticate API requests?",
    "max_memories": 10,
    "max_entities": 20,
    "include_relationships": true
  }'
```

---

## Code Graph (Sprint D-bis)

### List codebases

```bash
curl http://mnemos.dmiruke.dev:8200/api/codebases
# → {
#     "codebases": [
#       {"id": "...", "name": "mnemos", "repo_url": "...", "branch": "main",
#        "commit_hash": "...", "file_count": 202, "total_lines": 41544,
#        "language_mix": {"python": 66, "javascript": 1, ...}},
#       ...
#     ]
#   }
```

### Codebase detail + graph stats

```bash
curl "http://mnemos.dmiruke.dev:8200/api/codebases/<id>"
curl "http://mnemos.dmiruke.dev:8200/api/codebases/<id>/graph-stats"
# → {"symbols": 2526, "functions": 155, "classes": 72, "methods": 85,
#    "relationships": 10793, "calls": 1827, "imports": 246, "infra": 2}
```

### Filterable file list

```bash
curl "http://mnemos.dmiruke.dev:8200/api/codebases/<id>/files?language=python&imports=psycopg2&limit=10"
```

`imports=X` matches files whose `imports[]` jsonb array contains the exact
string `X` (jsonb `?` operator). `path=substr` does ILIKE.

### Symbol prefix search

```bash
curl "http://mnemos.dmiruke.dev:8200/api/symbols?prefix=authorize_payment&codebase_id=<id>&kind=function&limit=10"
```

### Call graph traversal

```bash
# Who does this symbol call (outgoing edges)?
curl "http://mnemos.dmiruke.dev:8200/api/symbols/<symbol_id>/calls?limit=50"

# Who calls this symbol (incoming edges)?
curl "http://mnemos.dmiruke.dev:8200/api/symbols/<symbol_id>/called-by?limit=50"

# Filter by relationship kind
curl "http://mnemos.dmiruke.dev:8200/api/symbols/<symbol_id>/calls?kind=imports"
```

### Infra resources

```bash
curl "http://mnemos.dmiruke.dev:8200/api/codebases/<id>/infra?source_kind=kubernetes"
curl "http://mnemos.dmiruke.dev:8200/api/codebases/<id>/infra?resource_kind=Deployment"
```

---

## Duplicates

```bash
# Cluster-wide pairs above similarity threshold
curl "http://mnemos.dmiruke.dev:8200/api/duplicates?threshold=0.95&limit=50"

# Find duplicates of one memory
curl "http://mnemos.dmiruke.dev:8200/memory/<id>/duplicates" -H 'Authorization: Bearer mnm-...'

# Start a dedup scan job
curl -X POST http://mnemos.dmiruke.dev:8200/memory/dedup/scan \
  -H 'Authorization: Bearer mnm-...' -H 'Content-Type: application/json' \
  -d '{"threshold": 0.92, "scope": "tenant"}'

# Poll
curl "http://mnemos.dmiruke.dev:8200/memory/dedup/scan/<job_id>" -H 'Authorization: Bearer mnm-...'
```

---

## Concept / Entity graph

```bash
# List entities (people / orgs / concepts) with filters
curl "http://mnemos.dmiruke.dev:8200/api/graph/nodes?type=concept&limit=100"

# All edges
curl "http://mnemos.dmiruke.dev:8200/api/graph/edges?limit=500"

# Walk a node's neighborhood
curl "http://mnemos.dmiruke.dev:8200/api/graph/nodes/<node_id>/neighbors?depth=2"

# Resolve a specific entity
curl "http://mnemos.dmiruke.dev:8200/graph/entity/<entity_id>"
```

---

## Uploads + Imports

### File upload (auto-routes by MIME)

```bash
curl -X POST http://mnemos.dmiruke.dev:8200/api/upload \
  -H 'X-Tenant-ID: demo' \
  -F 'files=@./paper.pdf' \
  -F 'files=@./README.md'
# PDFs + Markdown auto-build the page tree (document_nodes populated)
# .txt falls back to flat chunking
```

```bash
# Images / audio / video → media endpoint (handles binary properly)
curl -X POST http://mnemos.dmiruke.dev:8200/api/media/upload \
  -H 'X-Tenant-ID: demo' \
  -F 'file=@./diagram.png'
```

### Code repo ingest

```bash
# Sync (small repos)
curl -X POST http://mnemos.dmiruke.dev:8200/api/ingest/code/repo \
  -H 'X-Tenant-ID: demo' -H 'Content-Type: application/json' \
  -d '{"url": "https://github.com/your/repo", "branch": "main"}'

# Async (returns 202 + job_id)
curl -X POST http://mnemos.dmiruke.dev:8200/api/async/ingest/code/repo \
  -H 'X-Tenant-ID: demo' -H 'Content-Type: application/json' \
  -d '{"url": "https://github.com/your/repo", "branch": "main"}'
```

Code ingest automatically populates **all 5 layers** of the code graph
(symbols, relationships, imports, definitions, infra resources from
terraform/k8s/GH-Actions/Dockerfile files in the repo).

### Source-specific importers

`POST /api/import/{source}` (sync) and `POST /api/async/import/{source}`
(202 + job_id) for `chatgpt`, `discord`, `gmail`, `google`, `notion`,
`obsidian`, `slack`, `twitter`.

```bash
curl -X POST http://mnemos.dmiruke.dev:8200/api/async/import/notion \
  -H 'X-Tenant-ID: demo' -H 'Content-Type: application/json' \
  -d '{"token": "secret_...", "workspace_id": "..."}'
# → {"job_id": "...", "status": "queued"}
```

---

## Job status

```bash
curl http://mnemos.dmiruke.dev:8200/api/jobs?limit=20
curl http://mnemos.dmiruke.dev:8200/api/jobs/<job_id>
```

---

## Export

```bash
curl "http://mnemos.dmiruke.dev:8200/api/export/memories?format=ndjson&since=2026-01-01" \
  -H 'X-Tenant-ID: demo' > memories.ndjson

curl "http://mnemos.dmiruke.dev:8200/api/export/index"
curl "http://mnemos.dmiruke.dev:8200/api/export/stats"
```

---

## Operational

| Endpoint | What |
|---|---|
| `GET /health` | `{"status":"healthy","version":"2.0.0", ...}` |
| `GET /info` | API capabilities + features |
| `GET /api/stats` | Dashboard stats (total memories, recent count, duplicates pending) |
| `GET /api/metrics?period_days=7` | Operational metrics JSON (ingest / retrieval / latency rollup) |
| `GET /metrics` | **Prometheus** text format scrape endpoint |
| `GET /admin/embedding/queue` | Queue status (depth, lag, workers, totals) |
| `POST /admin/embedding/retry/{memory_id}` | Re-enqueue a stuck embedding |
| `GET /api/admin/dead-letters?limit=20` | Failed ingests with payload + error |
| `POST /embed` | Direct text → vector (for testing) |

---

## API key management

```bash
# List
curl http://mnemos.dmiruke.dev:8200/api/keys/ -H 'Authorization: Bearer mnm-master-...'

# Create
curl -X POST http://mnemos.dmiruke.dev:8200/api/keys/ \
  -H 'Authorization: Bearer mnm-master-...' -H 'Content-Type: application/json' \
  -d '{"name": "demo-agent", "scopes": ["read", "write"], "rate_limit_per_minute": 60}'
# → {"key": "mnm-xxxx", "key_id": "...", ...}   # secret returned once

# Lifecycle
curl -X POST "http://mnemos.dmiruke.dev:8200/api/keys/<key_id>/rotate" -H 'Authorization: Bearer mnm-master-...'
curl -X POST "http://mnemos.dmiruke.dev:8200/api/keys/<key_id>/deactivate" -H 'Authorization: Bearer mnm-master-...'

# Analytics
curl "http://mnemos.dmiruke.dev:8200/api/keys/analytics/usage?days=7" -H 'Authorization: Bearer mnm-master-...'
curl "http://mnemos.dmiruke.dev:8200/api/keys/analytics/endpoints?days=7&limit=20" -H 'Authorization: Bearer mnm-master-...'
curl "http://mnemos.dmiruke.dev:8200/api/keys/analytics/errors?days=7" -H 'Authorization: Bearer mnm-master-...'
```

---

## SDKs

A Python SDK lives at `sdk/python/` in the private repo (not on PyPI
yet). Reference contract:

```python
from mnemos_sdk import MnemosClient, MemoryType

client = MnemosClient(base_url="http://mnemos.dmiruke.dev:8200", api_key="mnm-...")

# Create
m = client.memories.create(
    content="...", memory_type=MemoryType.SEMANTIC, tags=["..."]
)

# Search
results = client.memories.search(query="...", limit=10)

# Context bundle
bundle = client.memories.context(query="...", max_memories=10)
```

For now, plain HTTP/curl is the supported integration surface.

---

## What's NOT in the public demo

- **Authentication enforcement.** The demo accepts any `Authorization`
  header; production deployments enforce API-key validation.
- **Rate limiting.** Production has slowapi-backed per-key + per-IP
  limits; demo doesn't.
- **Multi-tenant authz.** Demo respects `X-Tenant-ID` for scoping but
  doesn't enforce tenant authn; production wires API keys to tenants.
- **Sensitive operations** (DELETE on production data, mass exports) —
  blocked or rate-limited via dashboard role checks.

If you want to plug Mnemos into your own infrastructure with these
enforced, contact the author.
