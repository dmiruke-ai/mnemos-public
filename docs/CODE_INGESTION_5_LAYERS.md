# Code Ingestion as a 5-Layer Semantic Graph

> **The differentiator.** Most "code memory" systems vectorize files and
> search them. Mnemos treats a repository as a graph of executable
> semantics, dependencies, contracts, behaviours, flows, and operational
> intent — extracted in five concurrent layers from a single ingest.
>
> See also [`PRODUCT_WRITEUP.md`](PRODUCT_WRITEUP.md) and
> [`ARCHITECTURE.md`](ARCHITECTURE.md).

## Why not just RAG over source files?

Source code is a graph of:

- **executable semantics** — what does this function actually do?
- **dependencies** — what calls what; what imports what
- **contracts** — type signatures, decorators, abstract base classes
- **behaviors** — side effects, retry semantics, error handling
- **flows** — request paths through services
- **operational intent** — what deploys this, what permissions it needs

Embedding the source text and searching by cosine similarity captures
**none of these structurally**. You get fuzzy matches by text overlap
— useful for "find code about X" but useless for "what breaks if I
change this", "what permissions does this service need", "show me the
deploy DAG".

## The five layers

### Layer 1 — Structural

Repository topology: codebases, modules, packages, folders, manifests.

```sql
CREATE TABLE codebases (
    id UUID PRIMARY KEY,
    name TEXT, repo_url TEXT, branch TEXT, commit_hash TEXT,
    file_count INT, total_lines INT, languages JSONB,
    tenant_id TEXT, created_at TIMESTAMPTZ, last_indexed_at TIMESTAMPTZ
);

CREATE TABLE code_files (
    id UUID PRIMARY KEY,
    codebase_id UUID, file_path TEXT, language TEXT,
    line_count INT, file_size INT,
    imports JSONB,        -- array of imported module strings
    definitions JSONB,    -- {classes: [...], functions: [...]}
    memory_id UUID, content_hash TEXT
);
```

Sourced from: git metadata + filesystem walk. Read manifests
(`package.json`, `go.mod`, `requirements.txt`, `Cargo.toml`, etc.)
inline.

This layer is **shipped**.

### Layer 2 — AST / Symbol

Every function / class / method / interface / trait / module is a
typed node:

```sql
CREATE TABLE code_symbols (
    id UUID PRIMARY KEY,
    codebase_id UUID, file_id UUID, parent_symbol_id UUID,
    kind TEXT,           -- function | method | class | interface | module | type | trait | struct | enum | constant
    name TEXT,           -- 'authorize_payment'
    qualified_name TEXT, -- 'PaymentService.authorize_payment'
    ast_path TEXT,       -- 'Module/ClassDef:PaymentService/FunctionDef:authorize_payment'
    signature TEXT,      -- 'async def authorize_payment(self, amount: int) -> bool'
    docstring TEXT,
    file_path TEXT, language TEXT,
    line_start INT, line_end INT,
    memory_id UUID,      -- → memories (the content chunk)
    metadata JSONB       -- decorators, is_async, modifiers, ...
);

CREATE INDEX code_symbols_name_trgm  ON code_symbols USING gin (name gin_trgm_ops);
CREATE INDEX code_symbols_qname_trgm ON code_symbols USING gin (qualified_name gin_trgm_ops);
```

Sourced from:
- **Python:** stdlib `ast` module — full nested walk
- **JS / TS / TSX / Go / Rust / Java:** per-language regex pattern bank
  (placeholder for tree-sitter; the regex catches top-level decls)
- Future: tree-sitter for everything

This layer is **shipped**.

### Layer 3 — Runtime / Operational

Terraform, Kubernetes, GitHub Actions, Helm, Dockerfiles. The
*runtime* of your code, not the source itself.

```sql
CREATE TABLE infra_resources (
    id UUID PRIMARY KEY,
    codebase_id UUID, file_path TEXT,
    source_kind TEXT,        -- terraform | kubernetes | github-actions | dockerfile | helm | argocd
    resource_kind TEXT,      -- 'aws_instance', 'Deployment', 'job', 'workflow', 'stage', ...
    name TEXT,
    qualified_name TEXT,     -- 'aws_instance.web', 'prod/Deployment/api', 'workflow.CI/job.deploy'
    properties JSONB,        -- the parsed body
    depends_on TEXT[],       -- upstream qualified_names
    line_start INT, line_end INT,
    memory_id UUID
);
```

Parser dispatch (`ingest/infra_parsers/dispatch.py`):

| File pattern | Parser | What it extracts |
|---|---|---|
| `*.tf`, `*.tfvars` | `terraform.py` | `resource "TYPE" "NAME" { ... }` + `module`/`variable`/`output` blocks. Dependency edges from `<TYPE>.<NAME>` interpolations (literal-filtered to skip strings like `t3.micro`). |
| `*.yaml`, `*.yml` under `.github/workflows/` | `github_actions.py` | workflow + per-job nodes. `depends_on` from `needs:`. Captures `uses:` (third-party actions). |
| Other `*.yaml`, `*.yml` | `kubernetes.py` | Each `kind`+`metadata.name`+`metadata.namespace`. Deps from `configMapRef`, `secretRef`, `serviceAccountName`, `volumes.*.name`, `persistentVolumeClaim.claimName`. Multi-doc files supported. |
| `Dockerfile`, `*.Dockerfile`, `Dockerfile.*` | `dockerfile.py` | Each `FROM image AS stage` → image + stage nodes with depends_on edge. Multi-stage builds preserved. |

This layer is **shipped (terraform / k8s / GH-Actions / Dockerfile).
Helm + ArgoCD + IAM parsers are de-scoped from v1; the schema
accommodates them.**

### Layer 4 — Semantic

LLM-generated per-symbol summaries. Replaces low-quality centroid
embeddings (mean of chunks) with **summary embeddings** that capture
intent, not surface form.

```sql
CREATE TABLE code_semantic_summaries (
    id UUID PRIMARY KEY,
    symbol_id UUID,                  -- → code_symbols
    summary_text TEXT,               -- "This function validates JWT tokens against the JWKS keys"
    summary_kind TEXT DEFAULT 'description',
    -- description | behavior | side_effects | preconditions
    model TEXT,                       -- 'claude-sonnet-4-7'
    embedding_id UUID,                -- → embeddings (the summary embedding)
    generated_at TIMESTAMPTZ,
    metadata JSONB
);
```

Produced by the **Smriti reflection worker** (Sprint H, terminal).
Schema is in place; population deferred.

Why summaries? Because:

- Raw code embedding for `def get_user(id): ...` matches anything with
  similar variable names. The summary `"fetch a user record by ID
  from the users table with a 5-second timeout"` matches on behaviour.
- An LLM pass per symbol is ~$0.001 at current pricing. A 10k-symbol
  repo costs ~$10 to summarise, then queries are free forever.

### Layer 5 — Relationship

Typed edges between symbols (or between a symbol and an external
import target):

```sql
CREATE TABLE code_relationships (
    id UUID PRIMARY KEY,
    codebase_id UUID,
    src_symbol_id UUID,        -- caller
    dst_symbol_id UUID,        -- callee (NULL if external)
    dst_external TEXT,         -- 'os.path.join' for external targets
    kind TEXT,                 -- calls | imports | extends | implements | decorated-by | references | uses | instantiates | reads | writes
    properties JSONB           -- {line: 42, via: "method", target: "..."}
);
```

The graph substrate. Walked via recursive CTE for impact analysis
("who calls this?") and dependency analysis ("what does this call?").

**Today's resolution:** intra-file calls are resolved by name match
(best-effort). Cross-file calls become external edges with `dst_external`
set to the called name. Tree-sitter pass for full cross-file resolution
is planned.

This layer is **shipped (intra-file resolution; cross-file deferred).**

## Three retrieval substrates

Concurrently with the five storage layers, three retrieval substrates
operate on top:

### Substrate A — Symbol lookup

```sql
SELECT * FROM code_symbols
WHERE name ILIKE 'authorize_payment%' OR qualified_name ILIKE '%authorize_payment%'
ORDER BY (qualified_name = 'authorize_payment') DESC
LIMIT 10;
```

`pg_trgm` GIN indexes make this ~1ms even on 100k+ symbols. This is the
"I know the name I'm after" substrate.

Exposed via:
- REST: `GET /api/symbols?prefix=authorize_payment&codebase_id=...`
- MCP: `find_symbol(prefix="authorize_payment")`

### Substrate B — Vector

Standard pgvector HNSW over per-chunk embeddings. Works for "find code
conceptually similar to this" queries. Sprint H's summary embeddings
(Layer 4) will dramatically improve this substrate.

Exposed via:
- REST: `GET /api/search?query=...` (filters on `memory_type=procedural` + `tags=code`)
- MCP: `search_codebase(query="rate limiter implementation")`

### Substrate C — Graph

Recursive CTE walks over `code_relationships`. Direction-aware (`calls`
= outgoing, `called-by` = incoming).

```sql
WITH RECURSIVE callers AS (
    SELECT src_symbol_id, dst_symbol_id, 1 AS depth
    FROM code_relationships
    WHERE dst_symbol_id = $1 AND kind = 'calls'
  UNION ALL
    SELECT r.src_symbol_id, r.dst_symbol_id, c.depth + 1
    FROM code_relationships r
    JOIN callers c ON r.dst_symbol_id = c.src_symbol_id
    WHERE c.depth < 5 AND r.kind = 'calls'
)
SELECT * FROM callers;
```

Exposed via:
- REST: `GET /api/symbols/{id}/calls`, `GET /api/symbols/{id}/called-by`
- MCP: `who_calls(symbol="...")`, `get_symbol_callgraph(symbol="...")`

## Worked example — the mnemos codebase

Ingesting `mnemos` itself produced (counted from `/api/codebases/{id}/graph-stats`):

| Layer | Count |
|---|---|
| **L1** code_files | 202 |
| **L2** code_symbols | **2,526** (155 functions + 72 classes + 85 methods + 17 modules in the api/ subtree; full repo larger) |
| **L3** infra_resources | 2 (Dockerfile image + stage with depends_on edge) |
| **L5** code_relationships | **10,793** (8,553 calls + 1,612 imports + ~600 decorated-by/extends/etc.) |

A typical query, `get_symbol_callgraph(symbol="get_pool")`, returns:

```
Outgoing edges of: get_pool (function) — /api/db_pool.py:43

  [calls       ] _dsn (function) — /api/db_pool.py:39
  [calls       ] external → logger.info L52
  [calls       ] external → ThreadedConnectionPool L49
```

And the reverse, `who_calls(symbol="get_pool")`:

```
Callers of: get_pool (function) — /api/db_pool.py:43

  [calls       ] get_conn (function) — /api/db_pool.py:68
```

A cross-file edge (`get_pool` in `db_pool.py` is called by `get_conn` in
the same module, which in turn is called by many modules) is resolved
fully through Substrate C.

## Why Postgres for the graph?

We considered Neo4j / KuzuDB / Memgraph. We chose Postgres adjacency
tables because:

1. **One datastore to operate.** Postgres is already the system of
   record for memories, document_nodes, embeddings, audit log,
   API keys. Adding a graph DB doubles the ops surface.
2. **Recursive CTEs are fast enough at our scale.** 10k symbols and
   100k edges traverses in <5ms. We'd need a graph DB at 1M+ edges
   and high-depth traversals — when we hit that we'll add one.
3. **Migrations are cheap.** Adding a new edge type or property is a
   `JSONB` field change, not a graph schema migration.
4. **Joins with other substrates are free.** "Show me all symbols
   called by X that have a `decision` memory attached" is one SQL
   query, not a two-system fan-out.

If Postgres graph queries become the bottleneck, we'll add KuzuDB
(embedded, columnar, fast) as a read replica without changing the
primary write path.

## What's NOT extracted yet

- **Call chains across packages.** Today's intra-file resolution catches
  ~90% of in-process calls. Cross-file requires a name resolver or
  tree-sitter pass.
- **Effects analysis.** Side-effect tagging (`pure`, `reads_db`,
  `writes_s3`, `calls_external_api`) is not extracted. Smriti will
  populate this.
- **Type-flow / dataflow.** Argument types and return types are in the
  signature string but not parsed for downstream join. Sprint E or later.
- **Helm / ArgoCD / IAM.** Schema is ready; parsers de-scoped from v1.

## Future direction

Once Sprint H (Smriti) ships:

- Layer 4 gets populated for every symbol on ingest (or in batches)
- The vector substrate switches from "raw code embedding" to "summary
  embedding", with no API change to callers
- The graph substrate gets a new edge kind: `derived-from`, linking
  consolidated symbols back to their source AKUs
- Contradiction detection: when two symbols' summaries disagree
  ("validates JWT" vs "skips JWT for internal traffic"), flag a
  decision memory for human review

This is what turns "stored code" into "engineering memory" in the
philosophical sense.
