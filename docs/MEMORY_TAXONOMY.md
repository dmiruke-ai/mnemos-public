# Memory Taxonomy

Mnemos stores every fact as a typed **Atomic Knowledge Unit (AKU)** —
not an opaque embedded chunk. The type drives downstream behaviour:
which fields are required, which background workers process it, how it
shows up in retrieval, whether it's eligible for contradiction
detection, etc.

This document is the canonical list.

## The eight memory types

```sql
CREATE TYPE memory_type AS ENUM (
  'semantic',
  'episodic',
  'procedural',
  'decision',
  'note',
  'image',
  'audio',
  'document',
  'video'
);
```

### `semantic` — concepts, facts, definitions
The default. "Postgres uses MVCC for concurrency control." Embedded for
similarity search; participates in concept extraction; eligible for
contradiction detection.

### `episodic` — events with a timestamp
"Deploy v2.3.0 to prod at 14:32 UTC, failed health check at 14:35."
Carries `occurred_at` separately from `captured_at`. Used in
`list_recent`; participates in timeline reconstruction.

### `procedural` — how-to knowledge
"To rotate the production API key: 1) generate new key, 2) update
secret, 3) deploy, 4) revoke old key after 5 min." Code chunks
default to this type. Eligible for `find_implementations` lookup;
displayed with code-formatting in the dashboard.

### `decision` — choices made with rationale
Has a companion `decision_logs` row with structured alternatives
considered and the rationale. "Chose JWT over session cookies because
agent traffic is stateless." Surfaced via the `decision_trace` MCP
tool; eligible for contradiction detection against future decisions on
the same topic.

### `note` — free-form jottings
The "I'll figure this out later" bucket. Useful but un-typed; not
prioritised in retrieval.

### `image` / `audio` / `video` — media
The content is opaque (binary), but extracted text (OCR / ASR /
transcript) becomes searchable. Metadata carries width/height/duration/
codec/etc. Stored in `media/` (S3 in production, local FS in the demo).

### `document` — uploaded files larger than one chunk
The root memory for a PDF or large text file. Per-page chunks become
`semantic` memories linked back to the `document` parent via
`document_nodes`.

## What's on every memory

```sql
CREATE TABLE memories (
  id UUID PRIMARY KEY,
  memory_type memory_type NOT NULL,
  content TEXT NOT NULL,                    -- the textual body (or extracted text for media)

  -- Provenance + identity
  source VARCHAR(512),                       -- 'pdf:open-brain-architecture.pdf', 'mcp_tool', 'web_clipper'
  source_id VARCHAR(255),                    -- source's own id if any
  idempotency_key VARCHAR(255) UNIQUE,
  tenant_id VARCHAR(255) NOT NULL,
  user_id VARCHAR(255),

  -- Display
  title VARCHAR(512),
  summary TEXT,                              -- LLM-generated or hand-written

  -- Indexing
  tags TEXT[],
  metadata JSONB,
  search_vector TSVECTOR,                    -- auto-maintained via trigger
  classification data_classification DEFAULT 'internal',

  -- Lifecycle
  captured_at TIMESTAMPTZ DEFAULT now(),     -- when we received it
  occurred_at TIMESTAMPTZ,                   -- when the event happened (episodic)
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),

  -- Embedding pipeline
  embedding_status VARCHAR(50) DEFAULT 'pending',  -- pending|completed|failed
  embedding_attempts INT DEFAULT 0,
  embedding_error TEXT,
  embedding_queued_at TIMESTAMPTZ,
  processing_status TEXT DEFAULT 'done',

  -- Linkage
  document_node_id UUID,                     -- → document_nodes (page-tree)
  codebase_id UUID                            -- → codebases (for code memories)
);
```

## How types route through the system

```
                          ┌─────────────┐
                  ingest  │ memory_type │  retrieval
                          └──────┬──────┘
                                 │
       ┌─────────────────────────┼─────────────────────────┐
       │                         │                          │
  ┌────▼─────┐            ┌──────▼──────┐           ┌──────▼───────┐
  │ semantic │            │  decision   │           │ procedural   │
  │ episodic │            │             │           │              │
  │  note    │            │ decision_   │           │ Code path:   │
  │ document │            │   logs row  │           │ → code_files │
  │  image   │            │   created   │           │ → code_      │
  │  audio   │            │  + audit    │           │   symbols    │
  │  video   │            │  log entry  │           │   (D-bis)    │
  └──────────┘            └─────────────┘           └──────────────┘
       │                         │                          │
       └──────── all types → embedding queue ────────────────┘
                                 │
                                 ▼
                         pgvector + ts_rank_cd
```

## Tag conventions

Tags are unstructured strings but follow conventions:

| Prefix / pattern | Meaning |
|---|---|
| `project:<slug>` | Belongs to project `<slug>`. The `ingest_project.sh` script tags everything. |
| `source:<tool>` | Source system. `source:ingest_project.sh`, `source:web_clipper`. |
| `type:<kind>` | Type at ingest time. `type:pdf`, `type:code`, `type:metadata`. |
| `lang:<code>` | Programming language. `lang:python`, `lang:typescript`. |
| `page:<N>`, `pages:<total>` | PDF page hints. |
| `category:<slug>` | Optional grouping. |
| `file:<basename>` | Filename hint. |
| `heading:<...>` | Markdown heading the chunk lived under. |

Tags are not enforced; the dashboard's tag autocomplete reads existing
values from the corpus.

## Position via `document_nodes`

For `document`, `semantic`, and `procedural` memories that came from a
structured source (PDF, Markdown, code), `memory.document_node_id`
points at the matching `document_nodes` row, which carries:

- `kind` — `document` | `section` | `page` | `paragraph` | `chunk` | `file` | `code_block` | `figure`
- `parent_id` — the structural parent (for tree walks)
- `anchor` — source-specific position. For PDFs:
  `{page_num, page_width, page_height, char_count, source: "pdf"}`.
  For Markdown: `{line_start, line_end, heading_path, source: "markdown"}`.
  For code: `{file_path, language, ast_path, line_start, line_end, source: "code"}`.

This is what makes the breadcrumb work. See [`ARCHITECTURE.md`](ARCHITECTURE.md)
§3 and the [page-tree retrieval demo](https://github.com/dmiruke-ai/mnemos-public)
for examples.

## What about classification?

Every memory carries a `data_classification` enum: `public`, `internal`,
`confidential`, `restricted`. Defaults to `internal`. Retrieval can
filter on this; multi-tenant deployments enforce row-level access
based on classification × tenant.

## What's NOT a memory type (yet)

- **Conversation turn / chat message** — today these come in via
  importers (Slack, Discord, ChatGPT export) and land as `episodic`
  memories with `metadata.role = "user"|"assistant"`. A dedicated
  `chat` memory_type is on the Sprint D backlog.
- **Knowledge edge** — `concept_relationships` and `code_relationships`
  are edges, not nodes. They're stored in dedicated tables, not in
  `memories`.
- **Anchor / position** — the `document_nodes` row carries position;
  the `memory` carries content. They're linked but separate.

## Background: why typed at all?

Untyped memory ("just a vector store") has one mode: similarity. Every
question is answered the same way: top-K cosine neighbours. That's a
poor fit when:

- You want to find decisions, not facts.
- You want event timelines, not "things conceptually like this".
- You want to filter by language or codebase or media type before
  similarity.
- You want to detect contradictions between facts of the same shape.

Types let retrieval pre-filter (cheap, structured), then apply
similarity (expensive, fuzzy) only on the candidate set. The MCP
`list_recent(timeframe="24h", memory_type="decision")` exercises exactly
this: filter by type + time, then optionally rerank.
