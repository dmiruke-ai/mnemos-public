# Monitoring

Mnemos ships with a **Prometheus + Grafana** stack as first-class
infrastructure, not as an afterthought. This doc covers what's
instrumented, why, what the overhead is, and how to read the dashboards.

## Live

- **Prometheus:** [http://37.27.97.75:9091/](http://37.27.97.75:9091/)
- **Grafana:** [http://37.27.97.75:10210/d/mnemos-overview/](http://37.27.97.75:10210/d/mnemos-overview/)
- **Credentials:** `admin / mnemos` (admin); anonymous viewer role enabled (no login needed for read-only).

The Grafana dashboard "Mnemos — Overview" has 13 panels covering API
throughput, request latency, search QPS, embedding queue, ingest
throughput, DB pool, and process memory.

## What's instrumented

### HTTP tier — via Starlette middleware (`api/metrics.py`)

| Metric | Type | Labels |
|---|---|---|
| `mnemos_http_requests_total` | counter | method, route, status |
| `mnemos_http_request_duration_seconds` | histogram (11 buckets) | method, route |
| `mnemos_http_requests_in_flight` | gauge | method |

**Cardinality discipline:** the `route` label is the FastAPI **route
template** (`/api/memories/{memory_id}`), not the raw path. Otherwise
UUIDs would explode the series count.

### Search tier — via `HybridSearchService` (`retrieval/hybrid_search.py`)

| Metric | Type | Labels |
|---|---|---|
| `mnemos_search_queries_total` | counter | tool (`semantic` | `hybrid`) |
| `mnemos_search_duration_seconds` | histogram (9 buckets) | tool |
| `mnemos_search_results_count` | histogram | tool |

The label is derived from the config weights at call time — pure-vector
calls are tagged `semantic`, weighted calls are `hybrid`.

### Ingest tier — via `code_writer` + `writer` (`ingest/map_builders/`)

| Metric | Type | Labels |
|---|---|---|
| `mnemos_ingest_nodes_total` | counter | source (`pdf` | `markdown` | `code` | `kubernetes` | ...) |
| `mnemos_ingest_chunks_total` | counter | source |
| `mnemos_ingest_terms_total` | counter | source |
| `mnemos_ingest_duration_seconds` | histogram | source |

Source labels are pulled from the doc-root node's `anchor.source` field
at write time. Unknown sources become `source="unknown"`.

### DB pool — via `mnemos_db_pool_*` gauges

| Metric | Type | Refresh |
|---|---|---|
| `mnemos_db_pool_size` | gauge | per scrape |
| `mnemos_db_pool_in_use` | gauge | per scrape |
| `mnemos_db_pool_available` | gauge | per scrape |

Sampled inside the `/metrics` handler — no background thread, no
cardinality, ~1 µs of work per scrape.

### Embedding dispatcher — via `embedding_*` gauges + counters

| Metric | Type | Notes |
|---|---|---|
| `embedding_queue_depth` | gauge | in-process queue size |
| `embedding_queue_lag_seconds` | gauge | age of oldest pending DB row |
| `embedding_successes_total` | counter | persisted jobs |
| `embedding_failures_total` | counter | failed attempts (not "failed jobs") |

### Runtime — stock `prometheus_client` defaults

`python_gc_*`, `process_resident_memory_bytes`, `process_open_fds`,
`process_cpu_seconds_total`, etc. — useful for "is the API healthy"
panels.

## What's on the dashboard

13 panels, structured in 5 rows:

| Row | Panels |
|---|---|
| **Status** | API up · in-flight requests · embed queue depth · embed queue lag · DB pool in-use · DB pool size |
| **Traffic** | Requests/sec by route · request latency p50/p95/p99 |
| **Mix** | HTTP status mix (rate) · Search QPS by tool |
| **Quality** | Search latency p50/p95 by tool · Search result-count distribution p50/p95 |
| **Pipelines** | Ingest rate by source (chunks/sec) · Embedding success vs failure rate |
| **Resources** | DB pool in-use vs available · Process RSS |

All panels share a `now-1h` time range by default; the dashboard
refreshes every 30s.

## Overhead

Measured on the live system:

| Component | RAM | CPU | Disk | Network | Added API latency |
|---|---|---|---|---|---|
| Prometheus (15s scrape, 15d retention) | ~150 MB | <1% idle | **~50 MB/day** | ~5 KB/scrape | none |
| Grafana (no live dashboards open) | ~150 MB | <1% idle | ~20 MB total | trivial | none |
| `/metrics` endpoint (current 36 series) | <1 MB | ~10-20 µs/scrape | none | ~5 KB/response | none |
| HTTP middleware | ~5-10 MB | **~1-3 µs/request** | none | none | **<1% p99** |
| Search instrumentation | negligible | ~50 ns per `.inc()` / `.observe()` | none | none | none |
| Ingest counters | negligible | same | none | none | none |
| DB pool gauges | negligible | ~1 µs/scrape | none | none | none |

**Total:** ~300 MB extra RAM, ~50 MB disk/day, **<1% added p99
latency on the API**. Dominant cost is Prometheus's idle RAM, which is
a fixed line, not per-request.

## Configuration

The whole stack lives under the `monitoring` profile in
`docker-compose.yml`:

```bash
docker compose --profile monitoring up -d prometheus grafana
```

Configuration files:

```
ops/
├── prometheus/
│   └── prometheus.yml                       # 15s scrape, 15d retention
└── grafana/
    ├── provisioning/
    │   ├── datasources/prometheus.yml        # auto-provisioned datasource
    │   └── dashboards/dashboards.yml         # dashboard auto-load
    └── dashboards/
        └── mnemos-overview.json              # the 13-panel dashboard
```

Both services join the `mnemos_default` docker network and address the
API as `api:8000` internally.

## Adding a new metric

1. Define it in `api/metrics.py`:
   ```python
   from prometheus_client import Counter
   MY_COUNTER = Counter(
       "mnemos_my_counter_total",
       "Description",
       labelnames=("dimension",),
   )
   ```
2. Increment from anywhere: `from api.metrics import MY_COUNTER; MY_COUNTER.labels(dimension="...").inc()`
3. Add a panel to `ops/grafana/dashboards/mnemos-overview.json` (Grafana
   reloads provisioning every 30s — no restart needed).

**Label cardinality rule:** keep the cartesian product of all label
values under 1000. The HTTP middleware's `route` × `status` × `method`
combo is bounded because routes use templates not raw paths.

## What's NOT instrumented (yet)

- **Per-tenant breakdown.** Tenant labels would 10×+ cardinality;
  added as a separate `mnemos_tenant_*` series family when multi-tenant
  authz lands.
- **Long-running async jobs.** Worker process metrics are in scope but
  not yet exposed; arq has its own queue metrics that need bridging.
- **External API calls.** Ollama / Cohere outbound timing — easy to add
  with a 3-line decorator; deferred until volume justifies.
- **Storage S3.** Media upload size + duration distributions. Designed
  in `api/metrics.py` schema; pull when MinIO/S3 ships.

## Alerting

Prometheus has the rule engine but no rules yet — production setup
would add Alertmanager + rules for:

- API up == 0 for 1m
- Embed queue lag > 5min for 10m
- HTTP p95 > 1s for 5m
- Embed failure rate > 1/min for 5m
- DB pool exhausted (in-use == size) for 1m

The rules file goes in `ops/prometheus/rules/`; Alertmanager would be
a fourth compose service. Not in the public demo.
