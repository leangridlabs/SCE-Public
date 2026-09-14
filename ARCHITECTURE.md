# SCE Architecture

This document describes the system design and architectural guarantees of the
SCE Semantic Cache Engine. It is written for engineers evaluating whether SCE
fits their infrastructure. Proprietary scoring weights, distance thresholds,
and internal algorithm implementations are not included.

---

## System Overview

```
┌─────────────────────────────────────────────────────────────┐
│  Client (IDE / MCP host / agent / script)                   │
└──────────────────────────┬──────────────────────────────────┘
                           │  HTTP  (x-api-key header)
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  SCE Server  (Rust/Axum, :35100)                            │
│                                                               │
│   POST /mcp      ──► MCP adapter ──► Resolver / Ingest       │
│   POST /v1/chat/completions ──► Gateway ──► Resolver, or     │
│                                  real upstream + auto-commit │
│   POST /resolve  ──► Resolver cascade                        │
│   POST /ingest   ──► Chunker ──► Single-writer channel       │
│   POST /commit   ──► Single-writer channel                   │
│   POST /purge    ──► Single-writer channel + LRU flush       │
│   GET  /stats    ──► Read-only pool                          │
└──────────────────────────┬────────────────────────────────────┘
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
   ┌─────────────────┐      ┌──────────────────────┐
   │  Resolver       │      │  Single-writer task   │
   │  ├─ LRU-256     │      │  (mpsc channel)       │
   │  ├─ Lexical     │      │  Serialises all DB    │
   │  ├─ Semantic    │      │  writes; reads use    │
   │  └─ Graph       │      │  r2d2 pool            │
   └────────┬────────┘      └──────────┬────────────┘
            └──────────┬───────────────┘
                       ▼
           ┌───────────────────────┐
           │  SQLite (WAL mode)    │
           │  reasoning_cards      │
           │  cards_fts  (FTS5)    │
           │  graph_edges          │
           │  section_facts        │
           │  vec0 (sqlite-vec)    │
           └───────────────────────┘
```

---

## The Resolver Cascade

Incoming questions pass through a multi-tier cascade. Faster and cheaper
stages run first. A question exits the cascade as soon as a tier produces
a confident enough result; it never runs all tiers unnecessarily.

### Tier 1 — In-process LRU (exact repeat)

A 256-entry in-process LRU cache catches exact-string repeats before any
database access occurs. Latency: sub-millisecond.

### Tier 2 — Lexical search (FTS5/BM25)

Full-text search over committed card content using SQLite's built-in FTS5
engine with BM25 ranking. This stage acts as a binary gate: did anything
relevant exist at all? Cards clearing this stage proceed to the next tier;
everything else is classified as a miss immediately.

Latency on a warm 10,000-card store: ~5–10 ms.

### Tier 3 — Semantic similarity (dense embeddings)

Questions that pass the lexical gate are scored against card embeddings using
cosine similarity. Embeddings are produced by a locally-loaded MiniLM model
(384-dimensional, ONNX/FastEmbed) and stored in `sqlite-vec` virtual tables
statically linked into the binary. SIMD acceleration (AVX on x86, NEON on ARM)
is used automatically based on the host CPU.

No external vector database. No network call. No cloud sidecar.

Latency on a warm 10,000-card store: ~12–18 ms including tiers 1–3.

### Tier 4 — Semantic gate scoring

Cards that pass the lexical gate are scored against the query embedding using
cosine similarity. Two configurable precision gates apply at this tier,
evaluated via proprietary cosine-density thresholds calibrated across
enterprise document domains:

- **Cosine precision gate** — the primary similarity floor. Cards below the
  threshold are rejected.
- **Agent scope gate (SSS)** — measures alignment between the query and the
  centroid of cards already committed by this agent. Prevents recall from
  answering questions that are outside the established scope of the current
  session or agent. Configurable per deployment (disableable).
- **Domain knowledge gate (DKSA)** — measures alignment between the query and
  the centroid of the ingest corpus. Rejects queries that fall outside the
  document domain entirely. Configurable per deployment (disableable).

Precision is tunable on an integer scale — higher values are more
conservative. The shipped defaults are calibrated for a mixed general corpus;
enterprise deployments tune to their specific document set.

### Tier 5 — Graph-assisted context hydration

If the semantic tier returns a partial match (above a relevance floor but
below the recall confidence threshold), the graph layer fetches all
`depends_on` and `same-source` edges connected to the matching card. The
related cards and section facts are returned alongside the primary match as
structured context. The calling agent uses this to compress its prompt rather
than sending raw document content.

Token savings on a graph-assisted hit: 40–60% versus an uncached call.

---

## Route Decisions

The resolver outputs exactly one of four routes per query:

| Route | Condition | Agent action |
|---|---|---|
| `recall` | High-confidence match — lexical + semantic gates both cleared | Serve answer directly; no model call needed |
| `graph_assisted` | Partial match — lexical gate cleared, semantic below recall threshold | Use returned context to compress prompt |
| `agent_handoff` | No card match but structural index exists for the source path | Use section index as a document map |
| `generic` | No local signal | Call model normally |

The cascade always resolves. A cold miss on a 10,000-card store adds
~12–18 ms before forwarding to the model — negligible relative to LLM
API latency.

---

## Card Invalidation

### Content-hash invalidation

Every ingested document is fingerprinted with BLAKE3. On re-ingest, the hash
is compared chunk-by-chunk. Only sections whose content has changed are
re-embedded and re-committed. Unchanged sections remain warm. This prevents
mass eviction on minor document edits.

### Model and prompt-template drift

Card fingerprints are composite — they incorporate source content, the
producing model's identity, and the prompt template in effect at commit time.
If the underlying LLM or system prompt template changes, existing cards whose
source content is unchanged are still detected as stale and demoted to
`graph_assisted` mode on next hit. The old reasoning structure is used to
hydrate the new model run once; the resulting card is auto-committed with the
updated signature. No manual purge is required when upgrading models.

### Explicit purge

`POST /purge` with a `source_path` hard-deletes all cards and section facts
for that document and flushes the LRU cache. Used when a document is deleted
or replaced wholesale.

---

## Concurrency Model

**Writes** are serialised through a single Rust `mpsc` channel to a dedicated
writer task. All write paths (ingest, commit, purge, annotate) enqueue through
this channel. There is never more than one concurrent writer to SQLite, which
eliminates `SQLITE_BUSY` errors entirely under concurrent load.

**Reads** use a bounded `r2d2` connection pool. Read queries (`/resolve`,
`/cards`, `/stats`) are fully concurrent and never block on write operations.
SQLite WAL mode ensures readers and the writer do not contend for the same lock.

**Write batching** is used during bulk ingest. Large document ingestion is
processed in bounded chunks so that the write queue does not stall read traffic.

Validated under 50 concurrent agent simulations with zero errors and ≥ 95%
recall rate.

---

## Named API Key Model

SCE uses a named API key model in which each key carries a stable label, a
rotatable secret, a bound namespace, and an optional permission list.

```json
"compliance_admin": {
    "secret":      "...",
    "namespace":   "compliance",
    "permissions": ["supersession_approve"]
}
```

- **Label** — stable audit identity. Stored in every audit record. Does not
  change when the secret is rotated.
- **Secret** — the value sent in the `x-api-key` header. Rotate by updating
  config and restarting; no audit records are invalidated.
- **Namespace** — the data silo. All cards committed with a key go to its
  namespace. All recalls are filtered to that namespace.
- **Permissions** — optional capability grants. Currently defined:
  `supersession_approve` (required to confirm or reject supersession candidates).

The namespace filter is applied by the authentication middleware before any
handler code executes. There is no caller-provided parameter that can override
the namespace assigned by the key.

---

## Multi-Tenant Isolation

Namespace isolation is enforced at the query planner level, before any vector
or lexical execution occurs. Every query receives a mandatory namespace filter
derived from the authenticated API key's configured namespace.

A vector search issued under the `compliance` namespace executes only against
cards committed under that namespace. Cross-namespace semantic bleed is
prevented at the SQL layer, not by application logic that could be bypassed.

**Regulated namespaces** (`regulated: true`, the default) cannot be configured
with a TTL. If both are set, the server refuses to start. This prevents
accidental expiry of cards in regulated contexts.

---

## Supersession Governance

When a new document is ingested, SCE automatically compares it against all
existing documents in the same namespace using multiple independent signals
(structural, naming, and content-based) to estimate whether the new document
likely replaces an older one. No card is modified automatically — a
supersession candidate is advisory only, and requires agreement across more
than one signal before it is even surfaced for review.

A human reviewer with the `supersession_approve` permission then confirms or
rejects the candidate via the governance endpoints. On confirmation, all cards
from the older document are marked `superseded` and permanently excluded from
recall. Superseded cards remain in the database for audit and legal hold.

Every governance decision (confirm or reject) is written to an append-only
audit table before the response is returned. A decision without an audit
record cannot occur.

---

## Optimization Log

When `optimization_log.enabled: true` is set in config, every resolve decision
is written to a JSONL file and a bounded ring buffer. Each trace records the
namespace, a question hash, and the outcome of the resolution — which route
was returned and which card, if any. The log is off by default because it
contains scored query hashes. It is intended for corpus tuning engagements:
operators use it to measure the effect of threshold changes on real traffic
before committing to a new configuration.

---

## Storage Model

All data lives in a single SQLite database file (`sce.db`):

| Table | Contents |
|---|---|
| `reasoning_cards` | Card text, scope fingerprint, content hash, provenance envelope |
| `cards_fts` | FTS5 virtual table — BM25 index over card content |
| `graph_edges` | Directed edges between related cards |
| `section_facts` | Structural anchors returned on `agent_handoff` |
| `vec0` | 384-dim float32 embeddings (sqlite-vec virtual table) |

The database uses WAL mode with `synchronous=NORMAL` and a configured
`busy_timeout`. Background `PRAGMA incremental_vacuum` runs during idle
windows to prevent index fragmentation.

Because the store is plain SQLite, customer data is always accessible via
standard SQL tools regardless of whether SCE is running.

---

## API Reference

### Authentication

All endpoints except `/healthz` and `/ready` require `x-api-key: <key>`.
When named API keys are configured, the key resolves to a bound namespace
and permission set (see Named API Key Model above). The server refuses to
bind on a non-loopback address with no keys configured.

### Probes

```
GET /healthz   → 200 always while process is alive
GET /ready     → 200 when database + model ready; 503 otherwise
```

```json
{ "ready": true, "subsystems": { "database": "ok", "vec_store": "ok", "model": "loaded" } }
```

### Resolve

```
POST /resolve
{ "question": "string", "source_hint": "optional/path" }
```

```json
{
  "route": "recall | graph_assisted | agent_handoff | generic",
  "primary_card": { ... } | null,
  "related_cards": [ ... ],
  "section_facts": [ ... ]
}
```

### Gateway (OpenAI-wire-compatible)

```
POST /v1/chat/completions
{
  "model": "gpt-4o-mini",
  "messages": [ { "role": "user", "content": "string" } ],
  "sce_metadata": {
    "sources_read":     [ { "source_path": "...", "anchor_slug": "..." } ],
    "reasoning_chain":  [ "optional", "steps" ],
    "depends_on":       [ "optional-card-uuid" ],
    "committed_by":     "optional",
    "caller_identity":  "optional — who is asking, for the recall audit log",
    "namespace":        "optional"
  }
}
```

Accepts the same request shape any OpenAI Chat Completions client already sends.
`sce_metadata` is an optional SCE-only extension — stripped before the request is
forwarded upstream, so a real provider never sees it. On a cache hit, a synthesized
response is returned with zero upstream calls:

```json
{
  "choices": [ { "message": { "role": "assistant", "content": "..." } } ],
  "usage": { "prompt_tokens": 0, "completion_tokens": 0, "total_tokens": 0 },
  "sce_cache": { "route": "recall", "cache_hit": true, "card_id": "uuid", "audit_url": "/cards/uuid/audit" }
}
```

On a miss, the request is forwarded to the real provider using the caller's own
`Authorization` header (never stored by SCE), and the real response is committed as a
new card before being returned unmodified. Tool-calling conversations are supported:
the mid-loop round where a model requests a tool call is passed through with no
caching (nothing to cache yet); a final answer in a tool-augmented conversation is
committed with its `reasoning_chain` populated automatically, at no extra token cost
and no extra model call. `stream: true` requests are pass-through only.

### Ingest

```
POST /ingest
{ "path": "/absolute/path/to/document.md" }
```

```json
{ "status": "ok", "cards": 4, "changed": true }
```

`changed: false` — content hash unchanged; no cards re-committed.

### Commit

```
POST /commit
{
  "question": "string",
  "answer": "string",
  "source": "optional",
  "namespace": "optional — defaults to global",
  "committed_by": "optional — e.g. agent:workflow-1",
  "reasoning_chain": ["optional", "steps"],
  "depends_on": ["optional-card-uuid"],
  "tokens_used": 0
}
```

Returns `card_id`, `card_hash` (BLAKE3), `depends_on_edges`, `tokens_used`.

### Purge

```
POST /purge
{ "source_path": "path/to/document.md" }
```

### Stats

```
GET /stats
```

Returns per-card and aggregate token savings, hit rate, and route distribution.
Suitable for dashboard or CFO-level ROI reporting.

### Annotate

```
POST /annotate
{ "card_id": "uuid", "symbol": "MyFunction", "line_start": 42, "line_end": 80, "learned": "string" }
```

Attaches a symbol-level anchor fact to an existing card. Used by MCP session
protocol tools (`annotate_read`) to record function-level context.

### Cards

```
GET /cards
```

List committed cards with provenance fields.

```
GET /cards/{id}/audit
```

Returns full provenance for a single card: `question`, `answer`, `source_path`,
`committed_by`, `agent_model`, `reasoning_chain`, `depends_on` (resolved to full card
summaries, not just IDs), `sources_read` (each flagged `stale` against the source
document's current hash), `card_hash`, `tokens_used`, `recall_count`, `status`
(`active` or `superseded`), and `integrity_verified` (BLAKE3 hash check). Every
cache-hit response from `/resolve` and the gateway includes an `audit_url` field
pointing to this endpoint for that card.

```
GET /cards/{id}/recall-events
```

Every recorded serve of this specific card — timestamp, route, and caller identity
— distinct from the cumulative `recall_count` on the card itself.

```
GET /recall-events?caller=<substring>&since=<timestamp>&limit=<n>
```

Searchable log across every recall event on the server, not scoped to one card.

### Supersession Governance

```
GET  /supersession/candidates          — list pending candidates
POST /supersession/confirm             — mark old document cards superseded (requires supersession_approve)
POST /supersession/reject              — suppress this document pair permanently (requires supersession_approve)
```

Confirm and reject both write an immutable audit record before returning.
A candidate that has already been confirmed or rejected cannot be acted on again.

### Regulatory Acknowledgment

```
POST /acknowledge-regulatory-responsibility
{ "org_name": "Acme Insurance", "namespace": "compliance" }
```

Records a permanent acknowledgment entry (org name, key label, namespace,
timestamp). The hardcoded response message reminds operators that regulatory
compliance obligations are their responsibility, not SCE's.

### Debug

```
GET /debug/triage           — subsystem health detail, gate config, card/centroid counts
GET /debug/scoring-export   — last 500 scoring traces from the optimization log ring buffer
```

### MCP

```
POST /mcp
```

JSON-RPC 2.0. Supports `tools/list` and `tools/call` with tools `resolve`
and `ingest_document`. Compatible with GitHub Copilot, Cursor, Claude Desktop,
and any MCP Streamable HTTP client.

---

## TLS

TLS is optional. Set `tls_cert_path` and `tls_key_path` in config to enable
rustls-backed HTTPS. Both fields must be set together or neither. The server
defaults to plain HTTP on `:35100`.

---

## Air-Gapped Deployment

The binary has zero runtime dependencies on external network services. The
ONNX embedding model and all required runtimes are bundled in the deployment
package. Schema migrations run automatically at binary startup. Air-gapped
VPC deployments receive updates via a self-contained tarball or container image.
