# SCE Security Model

This document covers the security boundaries, data handling guarantees, and
enterprise deployment considerations for the SCE Semantic Cache Engine.

---

## Data Sovereignty

All customer data — reasoning cards, embeddings, graph edges, and section
facts — lives in a single SQLite file on the operator's own hardware.

- No data is transmitted to LeanGrid Labs servers, ever, under any configuration.
- No telemetry, analytics, or phone-home requests are made to LeanGrid Labs or any
  third party we control. Optional distributed tracing (OTLP) is available for
  enterprise observability — off by default, and when enabled it points only at a
  collector the operator configures themselves, never at LeanGrid Labs.
- SCE is a reasoning **cache and router in front of LLM cost**, not a reasoning
  generator itself — it has no model of its own to run, so a cache miss has to go
  somewhere. That destination is entirely operator-configured: an in-house or fully
  air-gapped model keeps the whole deployment air-gapped; a cloud provider receives
  only what the operator's own orchestrator would have sent it directly anyway. SCE
  itself never introduces a cloud dependency, and never routes anything through us.
- All data remains accessible via standard SQLite tooling regardless of
  whether SCE is running.

If a customer stops using SCE, their data is not locked behind a cloud API.
It is a plain `.db` file they already own.

---

## Authentication and Named API Keys

SCE uses a named API key model. Each key has a stable label (stored in audit
records), a rotatable secret (sent in the `x-api-key` header), a bound
namespace (the data silo the key can access), and an optional permission list.

```json
"compliance_admin": {
    "secret":      "...",
    "namespace":   "compliance",
    "permissions": ["supersession_approve"]
}
```

Key secrets are never logged or returned in any API response. The label is
stored; the secret is not. Rotating a secret requires updating the config
and restarting the server — no audit records are invalidated.

**The server refuses to start** if `bind_address` is non-loopback and no
keys are configured. Accidental unauthenticated exposure on a shared or
cloud host is a startup error, not a runtime condition.

---

## Namespace Isolation

Namespace scoping is enforced by the authentication middleware before any
handler code runs. Every API call resolves to the namespace bound to the
presented key. There is no caller-provided parameter that overrides the key's
namespace. Cross-namespace data access is structurally impossible through the
normal API surface.

For multi-environment deployments, the recommended pattern gives each
environment its own key:

| Key | Namespace | Used by |
|-----|-----------|--------|
| `compliance_dev` | `compliance:dev` | Dev agents, test harnesses |
| `compliance_test` | `compliance:test` | CI pipelines |
| `compliance_prod` | `compliance` | Production agents |
| `compliance_admin` | `compliance` | Human operators, supersession review |

---

## Regulated Namespace Guard

All namespaces default to `regulated: true`. A regulated namespace cannot be
configured with a TTL. If both are present in config, **the server refuses to
start** with an explicit error message:

```
ERROR: Namespace 'compliance' has regulated=true but ttl_days is set.
       Regulated namespaces must retain cards indefinitely.
```

This prevents accidental expiry of cards in regulated contexts. An operator
must explicitly set `regulated: false` (with a logged startup warning) to
permit TTL on a namespace.

---

## Supersession Governance Access Control

The `supersession_approve` permission is required to call
`/supersession/confirm` or `/supersession/reject`. A key without this
permission receives HTTP 403 before any database access occurs. There is no
escalation path from a plain API key to `supersession_approve`.

Every governance decision (confirm or reject) writes an immutable record to
the `supersession_audit` table before the response is returned. If the audit
write fails, the operation fails — a governance action without an audit record
cannot occur.

---

## Card Provenance and Tamper Detection

Every committed reasoning card carries a **provenance envelope**:

- `committed_by` — identity of the committing agent or user
- `agent_model` — which model produced this answer (e.g. `gpt-4o-mini`)
- `reasoning_chain` — ordered steps that produced the answer (optional; auto-derived
  from tool-call history for tool-augmented gateway conversations, otherwise
  caller-supplied)
- `sources_read` — every source document/section the agent read, each flagged
  `stale: true/false` against that document's current content hash
- `depends_on` — other cards this answer explicitly depends on, resolved to full
  card summaries on read (not just IDs)
- `card_hash` — BLAKE3 fingerprint of the committed question + answer
- `content_hash` — BLAKE3 fingerprint of the source content at commit time
- `tokens_used` — LLM cost at commit time (exact, reconstructed, or estimated)
- `recall_count` — cumulative number of times this card has been served

`content_hash` is recomputed on every re-ingest and on every resolve hit.
If the stored hash does not match the recomputed hash, the card is flagged
`integrity_verified: false` and demoted — it is not served as a `recall`
result. This detects in-place database tampering or storage corruption.

Every cache-hit response from `POST /resolve` and the `/v1/chat/completions`
gateway includes an `audit_url` field. Calling `GET /cards/{id}/audit` with a
valid API key returns all provenance fields, current card status (`active` or
`superseded`), and the `integrity_verified` boolean.

**Recall events are logged separately from card provenance.** In addition to
the cumulative `recall_count` on a card, every individual serve is written as
its own timestamped record — `GET /cards/{id}/recall-events` for one card's
full serve history, or `GET /recall-events?caller=&since=` to search across
every recall on the server. There is no recall event for which provenance is
unavailable.

Test coverage: acceptance_test.py Pattern 9 (trusted card round-trip) and
Pattern 10 (DB-level answer mutation → `integrity_verified: false`).
Full audit endpoint coverage: card_audit_test.py (9 steps).

---

## Gateway Security Model

The `/v1/chat/completions` gateway sits behind the same `x-api-key` middleware
as every other endpoint — it is not an unauthenticated bypass. The caller's own
provider `Authorization` header (their real OpenAI/Anthropic/etc. key) is
forwarded to the real upstream **unmodified, on a cache miss only** — SCE never
stores, logs, or inspects that key. This means the only required client-side
change to use the gateway is pointing `base_url` at SCE and adding the existing
`x-api-key` header; the orchestrator's provider credentials never touch SCE's
storage layer.

---

## Multi-Tenant Isolation

Namespace isolation is enforced at the SQL query planner level, before any
lexical or vector execution occurs (see Namespace Isolation above).

Validated under namespace_isolation_test.py (6 tests): an ambiguous query
that would match a card in namespace A returns `generic` when issued from
namespace B.

---

## Key Management

Private signing keys are **never** stored in plaintext on disk or embedded
in the binary. The server integrates with the host's native key store:

- AWS environments: AWS KMS
- Azure environments: Azure Key Vault
- Developer workstations: OS Keychain (Windows Credential Manager, macOS Keychain, Linux SecretService)

Signing operations execute in-memory via sealed key access. The key material
is not written to any log or response payload.

---

## Write Serialisation

All write paths (ingest, commit, purge, annotate) are serialised through a
single Rust `mpsc` channel to a dedicated writer task. There is never more
than one concurrent writer touching the SQLite database. This eliminates
write-write races and `SQLITE_BUSY` errors regardless of concurrent agent count.

Validated under concurrent_write_purge_test.py: 500-card purge running
concurrently with 200 parallel commits — zero data corruption, zero 5xx errors,
all post-purge committed cards readable.

---

## TLS

TLS is supported natively via the `axum-server` rustls backend. Set
`tls_cert_path` and `tls_key_path` in config to enable HTTPS. Both fields
must be set together; setting only one causes the server to refuse to start.

The server defaults to plain HTTP when neither field is configured, which is
appropriate for localhost or VPC-internal deployments where TLS termination
is handled at the load balancer or gateway layer.

---

## Optimization Log

When `optimization_log.enabled: true`, every resolve decision is written to
a JSONL file and a bounded ring buffer. Each trace records the namespace, a
question hash, and the resolution outcome. This log is **off by default**
because it contains scored query hashes. It is intended for operator tuning
engagements, not production enabled-always logging.

---

## Fleet Governance and Offboarding

For enterprise deployments with multiple developer instances:

- **Retention policies** are distributed as signed config files to local instances.
- **Offboarding** is handled via `POST /purge` with the employee's namespace.
  The endpoint cryptographically validates the purge request and removes all
  associated cards, graph edges, and section facts.
- **Audit trails** are available through the `committed_by` and `reasoning_chain`
  fields on every card, queryable via `/cards`.

---

## Responsible Disclosure

To report a security vulnerability, contact support@leangridlabs.com.
Please do not open a public issue for security findings.
