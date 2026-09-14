# SCE Test Evidence

This document summarises the test suites shipped with the SCE Semantic Cache
Engine and what each suite proves. All scripts in `scripts/` are black-box
callers against the compiled server binary — they contain no proprietary
implementation details and can be run against any deployed SCE instance.

---

## Test Suite Overview

| Suite | Script | Tests | What it proves |
|---|---|---|---|
| Milestone | `milestone_test.py` | 40 | Core correctness across recall, disambiguation, isolation, invalidation, adversarial stress |
| Acceptance | `acceptance_test.py` | 17 | Real-world behavioral patterns (ingest, update, concurrency, provenance, tampering, graph edges, audit trail completeness) |
| Card audit | `card_audit_test.py` | 9 | Full audit trail lifecycle: provenance round-trip, integrity check, audit_url in every hit, 404/400 edge cases |
| Namespace isolation | `namespace_isolation_test.py` | 6 | Cards committed under one namespace are invisible to queries from another |
| Supersession governance | `supersession_governance_test.py` | 9-step | Full lifecycle: v1 ingest → v2 ingest → candidate detected → agent key 403 → admin confirms → v1 cards superseded → v1 content misses recall |
| SSS/DKSA gate | `sss_dksa_gate_test.py` | 4 phases | Scope and domain gates fire correctly; vague queries blocked; specific queries pass; off-domain queries blocked |
| Shared relay | `shared_relay_test.py` | 10 | Multi-agent shared namespace recall: cards committed by a coordinator agent recalled correctly by a worker agent |
| Vertical Fleet | `vertical_fleet_test.py` | — | Cold/warm transitions across insurance, legal, and regulatory document sets |
| SQLite Stress | `sqlite_stress_test.py` | — | Resolve latency thresholds under 10,000-card store volume |
| Concurrent Write+Purge | `concurrent_write_purge_test.py` | — | Write/delete contention safety under simultaneous load |
| Ingest Coverage | `ingest_coverage_test.py` | 54 | 16 file formats + concurrent burst + idempotency + read/write race |
| Gateway | `gateway_smoke_test.py`, `gateway_tool_chain_live_test.py` | — | OpenAI-wire-compatible endpoint: cache-hit synthesis, cache-miss passthrough + commit, mechanical tool-call reasoning_chain extraction against a real model |
| SDK | `sce-client` test suite | 7 | Python SDK's `ForcedSession` genuinely skips the real model on a cache hit and commits correctly on every other route |

**Total controlled test assertions: 150+**

---

## Milestone Test Results (milestone_test.py)

### Suite breakdown

| Suite | Count | What passes |
|---|---|---|
| Milestone 1 — Orthogonal recall | 5 | Distinct facts recalled correctly; unrelated queries miss cleanly |
| Milestone 2 — Confusable pairs | 4 | Near-identical questions resolve to correct separate cards |
| Milestone 3 — Domain isolation | 5 | Multi-agent domain separation; zero cross-domain bleed |
| Ingest validation | 3 | Small, medium, and large document chunking produces expected card counts |
| Invalidation | 7 | Purge evicts cards; re-ingest restores them; cross-source isolation maintained |
| Structural recall | 2 | Zero wrong-card recalls across BM25 + semantic gate |
| Cross-document | 4 | Same rule applied across 10–20 independent source documents returns correct attribution |
| Semantic delta | 4 | Chunk-level re-ingest: changed sections update; unchanged sections stay warm |
| Adversarial stress | 1 | 50 off-domain probes: zero wrong-card recalls |
| Concurrent fleet | 3 | 50 concurrent agents: ≥ 95% recall rate, zero errors, zero wrong-card recalls |

### Key result: zero wrong-card recalls

Across all 40 milestone tests — including 50 adversarial off-domain probes
and confusable-pair disambiguation — **zero incorrect cards were returned as
`recall` results**. Questions below confidence threshold are demoted to
`graph_assisted` or `generic` rather than serving a potentially wrong answer.

---

## Acceptance Test Results (acceptance_test.py)

Each pattern maps to a documented failure mode and the SCE mechanism that
addresses it.

| Pattern | Description | Result |
|---|---|---|
| 1 | File-based pre-ingest — ingest pipeline, recall after ingest | PASS |
| 2 | On-demand fetch — commit-then-recall | PASS |
| 3 | Tool-call retrieval — cold miss → commit → deterministic recall | PASS |
| 4 | Shared memory — 50 concurrent agents, zero errors, zero wrong recalls | PASS |
| 5 | Living document updates — delta detection, stale recall prevention | PASS |
| 6 | Cross-document reasoning — source attribution, no answer bleed | PASS |
| 7 | Multi-tenant isolation — per-tenant isolation, ambiguous → generic | PASS |
| 8 | Ambiguous query handling — safe fallback to generic route | PASS |
| 9 | Card provenance (trusted) — committed_by / reasoning_chain round-trip, `integrity_verified: true` | PASS |
| 10 | Card provenance (tampered) — DB-level answer_text mutation → `integrity_verified: false` | PASS |
| 11 | Regulation cascade invalidation — derived answer purged when source regulation changes | PASS |
| 12 | Explicit dependency edges — `depends_on` links create graph edges surfaced on graph-assisted routes | PASS |
| 13 | Smart-read cache gate — `cache_hit` correct on both cold and warm paths | PASS |
| 14 | TurnContext agent signals — `suggested_action`/`should_continue` correct per route | PASS |
| 15 | Graph-assisted related cards — related cards populated via real graph-edge hydration | PASS |
| 16 | Annotate persistence — symbol-level anchor facts persist and round-trip correctly | PASS |
| 17 | Audit trail completeness — `agent_model`, `sources_read` (with staleness), and resolved `depends_on` all present on `/cards/{id}/audit` | PASS |

---

## Concurrent Write + Purge Test (concurrent_write_purge_test.py)

**Setup:** 500 cards committed to a seed source. A purge of that source is
triggered. While the purge is executing, 200 additional cards are committed
to a separate source by 20 concurrent writer threads.

**Assertions verified:**
- Zero 5xx errors from the server under write + purge concurrency
- All purge-target cards are gone after purge completes
- All cards committed during and after the purge are fully readable
- No data corruption detected in post-run verification sample

**What this proves:** SQLite WAL mode with the single-writer `mpsc` channel
handles simultaneous write/delete contention without dropped writes,
corrupted records, or server errors.

---

## SQLite Stress Test (sqlite_stress_test.py)

**Setup:** 10,000 mock cards committed (10,527-card store at time of timing). 500
cards sampled; 1,000 total resolve queries issued (one exact-match and one
paraphrase per card), with the LRU fast-lane bypassed so every query hits SQLite.

**Measured latency (live run):**

| Query type | P50 | P95 | P99 | Threshold (P99) | Result |
|---|---|---|---|---|---|
| Exact match | 8.3 ms | 10.6 ms | 18.9 ms | ≤ 100 ms | PASS |
| Paraphrase (semantic) | 1,190.9 ms | 1,253.1 ms | 1,362.3 ms | ≤ 2,500 ms | PASS |

**What this proves:** paraphrase latency is dominated by embedding-model CPU
inference, not storage — the P50→P99 spread for paraphrase queries is only
172ms, meaning SQLite/hashing overhead stays negligible even at 10,500+ cards.
Write throughput: 10,000 cards committed in 390s (26 cards/s) with 20 concurrent
workers, WAL mode, zero errors.

---

## Ingest Coverage Test (ingest_coverage_test.py)

54 assertions across:
- 16 file formats (Markdown, PDF, DOCX, TXT, CSV, Python, Rust, TypeScript, Go, Java, SQL, YAML, JSON, TOML, XLSX, LaTeX)
- Concurrent burst ingest (multiple documents ingested simultaneously)
- Idempotency (same document ingested twice → `changed: false`, no duplicate cards)
- Read/write race (resolve queries during active ingest → correct results, no errors)

---

## Card Audit Test (card_audit_test.py)

9-step black-box acceptance test against a live server verifying the complete
audit trail lifecycle:

| Step | What it verifies |
|------|------------------|
| 1 | Commit a card with explicit `committed_by` and `reasoning_chain` |
| 2 | Resolve a matching question — `audit_url` present in the cache-hit response |
| 3 | Call `audit_url` — all provenance fields returned |
| 4 | `integrity_verified: true` — BLAKE3 hash passes against stored content |
| 5 | `status: active` — newly committed card is correctly flagged active |
| 6 | `committed_by`, `reasoning_chain`, question, answer, and source all match what was committed |
| 7 | Non-existent but valid UUID → HTTP 404 |
| 8 | Malformed UUID → HTTP 400 |
| 9 | Generic (cache miss) response does not include `audit_url` |

---

## Namespace Isolation Test (namespace_isolation_test.py)

6 assertions against a live server with two named API keys bound to different
namespaces:

- Card committed under namespace A is recalled correctly by namespace A
- Same card returns `generic` when queried from namespace B
- Card committed under namespace B is recalled correctly by namespace B
- Namespace A cannot see namespace B cards (directional)
- Namespace B cannot see namespace A cards (directional)
- Queries to an empty namespace always return `generic`, never a cross-namespace hit

---

## Supersession Governance Test (supersession_governance_test.py)

9-step end-to-end workflow:

| Step | What it verifies |
|------|------------------|
| 1 | Ingest v1 document — cards committed |
| 2 | Ingest v2 document — supersession candidate auto-detected |
| 3 | Candidate appears in `/supersession/candidates` with expected confidence and signals |
| 4 | Agent key (no `supersession_approve`) receives HTTP 403 on confirm attempt |
| 5 | Admin key confirms candidate — returns count of cards affected |
| 6 | All v1-source cards now have `status: superseded` |
| 7 | Question that previously recalled a v1 card now returns `generic` |
| 8 | Audit entry present in `/supersession/audit` with correct event, signals, and reviewer |
| 9 | Confirmed candidate no longer appears in pending candidates list |

---

## SSS/DKSA Gate Test (sss_dksa_gate_test.py)

4-phase integration test with a fresh server, seeded corpus, and gate
precision tuned to a representative deployment setting (precision is fully
configurable per deployment — these are test-harness values, not fixed
defaults):

| Phase | What it verifies |
|-------|------------------|
| Phase 0 (6/6 PASS) | Before agent centroid is built, SSS gate is inactive — no false blocks |
| Phase 2 (10/10 PASS) | Vague/out-of-scope questions blocked by SSS gate |
| Phase 3 (6/6 PASS) | Specific in-scope questions pass the SSS gate |
| Phase 4 (10/10 PASS) | Off-domain questions blocked by DKSA gate |

Key property proven: the scope gate is directional — vague/ambient queries
score consistently and measurably closer to the "blocked" side than specific,
document-aligned queries do, and the domain gate blocks queries entirely
outside the ingest corpus regardless of phrasing.

---

## Gateway Test Results (gateway_smoke_test.py, gateway_tool_chain_live_test.py)

Black-box tests against the OpenAI-wire-compatible `/v1/chat/completions`
endpoint, including a run against a real model provider (not a stub).

| Assertion | Result |
|---|---|
| Cache miss forwards to upstream exactly once, response committed | PASS |
| Identical second call served from cache — zero additional upstream calls | PASS |
| `sce_metadata` (`sources_read`, `committed_by`, `reasoning_chain`) round-trips into `/cards/{id}/audit` | PASS |
| Mid-loop tool-call round (model requests a tool, no final answer yet) is never cached | PASS |
| Final answer in a tool-augmented conversation commits with `reasoning_chain` auto-derived from that conversation's own tool-call/tool-result history — zero extra model calls | PASS |

**What this proves:** the gateway integration path works end-to-end against a
real model provider, including the tool-calling case, without any framework-
specific adapter code.

---

## sce-client SDK Test Suite

> 7 / 7 passing (6 unit + 1 live acceptance)

Proves the Python SDK that framework adapters are built on top of.

| Test | Proves |
|---|---|
| Recall hit never calls the real model | `ForcedSession` genuinely skips generation on a cache hit — not just a suggestion |
| Non-recall routes always generate and commit | `graph_assisted`/`agent_handoff`/`generic` are correctly treated as real-generation routes |
| Commit receives all session- and call-level fields | `agent_model`/`reasoning_chain`/`depends_on`/`tokens_used`/`sources_read` all reach `/commit` correctly |
| Live cold-then-warm cycle against a real release binary | The real model is called exactly once across a cold ask followed by a warm ask on the same question |

---

## Running the Tests Yourself

All scripts require `pip install requests` and a running SCE server.

```bash
# Start server first (Docker or binary)
docker compose up -d
# or
./sce-server

# Core suites
python scripts/acceptance_test.py
python scripts/milestone_test.py
python scripts/concurrent_write_purge_test.py
python scripts/sqlite_stress_test.py --cards 10000 --queries 500
python scripts/ingest_coverage_test.py

# Governance and audit suites (require a server with named API keys configured)
python scripts/card_audit_test.py
python scripts/namespace_isolation_test.py
python scripts/supersession_governance_test.py

# Gate correctness suite (requires a server with sss_precision and dksa_precision set)
python scripts/sss_dksa_gate_test.py

# Gateway suite (tool-chain live test requires a real provider API key)
python scripts/gateway_smoke_test.py
python scripts/gateway_tool_chain_live_test.py

# Point at a remote server
SCE_URL=http://192.168.1.10:35100 SCE_API_KEY=your-key python scripts/acceptance_test.py
```

Exit 0 = all assertions passed. Exit 1 = one or more failed. Exit 2 = server unreachable.

The scripts contain no proprietary logic — they are black-box HTTP callers.
You can read every line and verify exactly what is being tested.
