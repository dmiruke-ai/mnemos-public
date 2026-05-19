# Mnemos — Product Writeup

## The problem

Today's "AI memory" looks like RAG: chunk every document, embed the
chunks, search by cosine similarity, paste the top-K into an LLM
prompt. That works for FAQs. It collapses on the workloads that actually
matter:

1. **Citations are noise.** A retrieved chunk has no idea what document
   it came from, what section, what page. Agents that need to *cite*
   the source — or jump to the next paragraph for context — have to
   reconstruct that from substrings.

2. **Code is a graph, not text.** A function isn't a 500-token snippet.
   It's a typed node in a call graph, with dependencies, callers,
   decorators, types, and infrastructure ties. Embedding the source text
   loses 90% of what makes the code meaningful for impact analysis or
   refactor planning.

3. **Single-substrate retrieval flattens everything.** Vector similarity
   is good for "find me something conceptually like X". Exact symbol
   lookup is good for "where is `authorize_payment` defined". Graph
   traversal is good for "what breaks if I change this function". A
   pure-vector retrieval system answers the first one well and the
   other two terribly.

4. **Operational intent disappears.** Your terraform, k8s manifests,
   GitHub Actions workflows, and Helm values describe the *runtime* of
   your system. Embedded as text, they become unsearchable for "what
   IAM permissions does this service have" or "which workflow deploys
   to prod". A memory system that doesn't model this can't help agents
   reason about deployments.

5. **No reflection layer.** Memory grows monotonically. Without a
   background process that consolidates, contradicts, and summarises,
   the system gets slower over time and the signal-to-noise of retrieval
   degrades.

## What Mnemos does

Mnemos restructures memory as a **typed, positioned, graph-aware substrate**:

### 1. Page-tree retrieval (shipped)

Every document — PDF, Markdown, code file — is ingested as a tree:

```
Document
  ├─ Section "Architecture"
  │   ├─ Paragraph (lines 12–28)
  │   └─ Paragraph (lines 29–41)
  ├─ Section "Storage"
  └─ Section "Retrieval"
```

Each leaf carries an **anchor** — `{page_num: 5}` for PDFs,
`{line_start, line_end, heading_path}` for Markdown, `{ast_path, line_start}`
for code. Retrieval returns not just the matched chunk but the full
**breadcrumb** walking back to the doc root, plus the anchor.

Caller-facing: every `semantic_search` and `hybrid_search` result
includes `Breadcrumb: Open Brain Architecture > p.5` and
`Anchor: {"page_num": 5, "char_count": 1240, "source": "pdf"}` —
ready to cite, ready to deep-link.

### 2. Five-layer code semantic graph (shipped — Sprint D-bis)

Codebases are not flat text. Mnemos extracts five concurrent layers from
every ingest:

| Layer | What it captures |
|---|---|
| **1 · Structural** | Repos, modules, packages, folders, ownership, manifests (`package.json`, `go.mod`, `Cargo.toml`, …) |
| **2 · AST / Symbol** | Functions, classes, methods, interfaces, traits, with `qualified_name`, signature, docstring, line range |
| **3 · Runtime / Operational** | Terraform resources, Kubernetes objects, GitHub Actions DAGs, Dockerfile stages — with their `depends_on` edges |
| **4 · Semantic** | (schema in place; Smriti will populate) LLM-generated per-symbol summaries: "this function validates JWT tokens" |
| **5 · Relationship** | Typed edges: `calls`, `imports`, `extends`, `decorated-by`, `references` — across files |

Then three **retrieval substrates** sit on top:

| Substrate | Use case |
|---|---|
| **Symbol trie** (pg_trgm GIN) | "Find `authorize_payment`" — sub-millisecond exact / prefix |
| **Vector** (pgvector HNSW) | "Find code conceptually about JWT validation" |
| **Graph** (recursive CTE over edges) | "What calls this function?", "What does this depend on?" |

See [`CODE_INGESTION_5_LAYERS.md`](CODE_INGESTION_5_LAYERS.md) for the
full model.

### 3. Operational graph (shipped)

Terraform, Kubernetes, GitHub Actions, and Dockerfile sources are parsed
into typed `infra_resources`:

- `aws_instance.web` depends on `aws_subnet.public`
- `prod/Deployment/api` depends on `prod/ConfigMap/api-config` and `prod/ServiceAccount/api-sa`
- `workflow.CI/job.deploy` depends on `workflow.CI/job.test`
- `stage.app` depends on `image.python:3.11-slim`

Retrieval can now answer: "what AWS resources does this codebase
provision?", "what's the GH-Actions DAG for releases?", "which k8s
deployments use this ConfigMap?" — none of which a vector RAG can do.

### 4. Smriti reflection layer (designed, sprint H)

A background worker that runs over `document_nodes` and `code_symbols`,
producing:

- `summarize_memory_cluster` — derived summaries that replace centroid embeddings
- `derive_concepts` — extract higher-order concepts from clusters
- `detect_contradictions` — flag conflicting memories
- `generate_long_term_summary` — compressed historical view
- `build_contextual_profile` — per-tenant cognitive snapshot
- `compress_memory_graph` — prune low-importance branches
- `infer_importance` — automatic salience scoring
- `create_derived_memory` — synthesised AKUs from atomic ones

This is what turns "stored facts" into "memory" in the philosophical
sense.

## Positioning

| Tool | What it's good at | Where Mnemos differs |
|---|---|---|
| **LangChain RAG / LlamaIndex** | One-shot Q&A over embedded chunks | No page tree; no code graph; no operational layer; no reflection |
| **Sourcegraph / Cody** | Code search + LSP-style navigation | Per-language; no cross-source (docs, ops); no MCP; closed-source server |
| **Mem0 / Letta / Zep / Memgpt** | Conversational AI memory layer | Strong on chat memory; thin on document provenance + code graph + operational reasoning |
| **Cognee** | Knowledge-graph memory for agents | Close conceptual neighbour; Mnemos adds the page tree, the runtime-ops layer, and the public MCP surface |
| **Vector DBs (Pinecone, Weaviate, Qdrant)** | One substrate (vector) | Mnemos uses pgvector as one of three substrates, integrated with the others, not as a standalone surface |

## Use cases

### A. AI engineering assistant with grounded code reasoning

An LLM agent with the Mnemos MCP tools can:

1. **`find_symbol(prefix="authorize_payment")`** — locate a function in
   <1ms.
2. **`get_symbol_callgraph(symbol="PaymentService.authorize")`** — see
   every function the target calls and the line where.
3. **`who_calls(symbol="PaymentService.authorize")`** — impact analysis:
   what breaks if I change this?
4. **`get_infra_resources(codebase="checkout", source_kind="kubernetes")`** —
   which Deployments / Services / ConfigMaps does this codebase ship?
5. **`semantic_search(query="how does retry logic work?")`** — returns
   results with `Breadcrumb: Engineering Handbook > §4 Retries > p.5`,
   ready to cite.

### B. Compliance + audit assistant

Every memory has tenant scoping, captured timestamp, and an immutable
audit log entry. Decision memories carry a structured `decision_logs`
row tracing alternatives, rationale, and outcomes. Searches return
provenance, not just snippets — so "show me where we decided to use
JWT instead of session cookies" returns the decision memory with the
exact paragraph + page citation.

### C. Operational troubleshooting

When something breaks in production, an agent can:

1. Identify the symbol in the stack trace: `find_symbol(prefix="...")`.
2. Walk callers (`who_calls`) to find upstream invocations.
3. Pull the runtime resources (`get_infra_resources(source_kind="kubernetes")`)
   to see what's deployed.
4. Cross-reference recent decisions (`decision_trace`) for relevant changes.

All in one substrate, all carrying provenance.

### D. Long-term engineering knowledge

Mnemos preserves the *why* — design decisions, post-mortems, ADRs,
discussions — alongside the *what* — the current state of the code.
Search a year later and you get both, with citations back to the
original documents.

## Why not just stitch this from existing tools?

You could glue together: a vector DB, a separate AST-extractor, a
graph DB, a search index, a documents-as-images parser, and an LLM-
summarisation worker. Each integration is a maintenance burden, each
substrate has its own schema, each retrieval call has its own latency
profile, and the pieces don't share a typed model of "what is a memory".

Mnemos is opinionated about the model — every storage substrate is a
view of the same typed Atomic Knowledge Unit (AKU) — and operationally
boring: one Postgres, one Redis, one API process, one worker. The 5-layer
code graph fits in Postgres adjacency tables; the page tree fits in
`document_nodes`; the vector index is `pgvector`. No new datastore to
operate, until evidence shows pgvector or recursive-CTE can't scale.

## Status

| Sprint | Scope | Status |
|---|---|---|
| Sprint A | Phase 0: partitions, indexes, compose, gunicorn | ✅ shipped |
| Sprint B | Phase 1: Redis + worker + arq + 202 contract | ✅ shipped |
| Sprint C | Phase 0 + 1a: page-tree writer, breadcrumb in MCP, e2e demo | ✅ shipped |
| Sprint D-bis | 5-layer code semantic graph + retrieval substrates | ✅ shipped |
| Sprint D | Multi-resolution embeddings + 7 remaining map builders (chat / email / image / audio / table / log / html) | next up |
| Sprint E | Retrieval Planner v1 + Memory Fusion Layer + 7 new MCP tools | next up after D |
| Sprint F (optional) | BM25 swap, backlinks, `[[wiki-link]]` parser | stretch |
| Sprint G (terminal) | VisDoc adapter (replaces internal PDF/image builders) | terminal |
| Sprint H (terminal) | Smriti reflection worker | terminal |

The hosted demo above runs through Sprint D-bis. Sprint E adds the
explicit retrieval funnel; mnemos.dev wiring follows.

## Contact

This is a working prototype with patent pending. For evaluation use
the hosted demo above. For embedded use, derivative work, or
commercial licensing, contact the author.
