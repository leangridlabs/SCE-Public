# SCE — Semantic Cache Engine

A local-first, zero-egress semantic cache server for AI agent pipelines.

SCE sits between your agents and your LLM provider. Questions that match a
previously committed reasoning card are served directly from the local store
at zero token cost. Questions that do not match fall through to your model
with structured context attached — reducing prompt size on partial hits.
Every resolved card is committed back, so the hit rate improves continuously.

**Primary metric:** 73–80% reduction in inference cost on warm (previously-seen) queries
across document-heavy workloads.[^1]

[^1]: Baseline is **engine disabled entirely** — i.e. the typical current-state cost of
    asking an LLM the same question repeatedly with no caching layer at all, not a
    cold-vs-warm comparison within the engine itself. Per-run savings measured at
    73–77% on the first warm pass after a question is cached; the all-time ratio
    reaches ~80% as the card store matures and more questions hit recall. Verified
    with real LLM API calls, engine-on vs. engine-off. See
    [docs/TEST-EVIDENCE.md](docs/TEST-EVIDENCE.md) for methodology and test coverage.

---

## What It Is (and What It Is Not)

| It is | It is not |
|---|---|
| A local semantic cache and retrieval server | An agent framework or orchestrator |
| An MCP server + REST API | An LLM |
| A shared reasoning substrate for agent fleets | A cloud vector database |
| A zero-dependency Rust binary | A SaaS product with cloud egress |

The server exposes two interfaces: a **REST API** and a **Model Context
Protocol (MCP)** endpoint. Any MCP-capable client — VS Code + Copilot,
Cursor, JetBrains AI, Claude Desktop, or a bare HTTP script — can use it
without installing a plugin or SDK.

---

## Key Properties

- **Framework-agnostic** — any agent that can speak HTTP or MCP works. No SDK, no lock-in.
- **Model-agnostic** — works with any model; swap providers freely. Cards are fingerprinted against the model that produced them, so a model change is detected automatically: affected cards demote gracefully and re-commit under the new signature. No manual purge required.
- **IDE-agnostic** — VS Code, Cursor, JetBrains, Neovim, or a script. One config line.
- **Zero egress to us** — cards, embeddings, and graph edges live only in a local
  SQLite file, and nothing is ever sent to LeanGrid Labs or any third party we
  control. SCE is a reasoning **cache and router**, not a reasoning generator — on a
  cache miss, the gateway forwards only to the LLM endpoint you configure. Point it
  at an in-house or fully air-gapped model and the entire stack stays air-gapped;
  point it at a cloud provider and only that provider ever sees the request. Nothing
  ties SCE itself to any cloud dependency.
- **Savings compound** — every agent handoff that produces an answer commits it back. The next agent asking an equivalent question pays zero. The ratio improves with use.

---

## Gateway — Zero Code Changes for Existing Orchestrators

Any orchestrator that already supports a custom `base_url` — LangChain, LangGraph,
LlamaIndex, AutoGen, CrewAI, or a raw `openai`/`anthropic` SDK client — gets cache-hit
recall by pointing `base_url` at SCE's `/v1/chat/completions` endpoint and adding one
`x-api-key` header alongside the orchestrator's existing provider `Authorization` header.
That header is forwarded to the real upstream provider untouched on a cache miss — SCE
never stores or needs the caller's provider API key.

- **Cache hit** — an OpenAI-shaped response is synthesized locally, zero upstream calls, zero tokens billed.
- **Cache miss** — forwarded to the real provider, the real response is captured and committed, and returned to the caller unmodified.
- Works transparently through nested/multi-orchestrator stacks, since every layer eventually converges on the same outbound HTTP call.

---

## Quick Start

### Try the demo image (fastest — no build, no config)

```bash
docker pull ghcr.io/leangridlabs/sce-demo-verticals:latest
docker run --rm ghcr.io/leangridlabs/sce-demo-verticals:latest
```

Runs a real compiled `sce-server` plus scripted demo scenarios — full unedited
terminal transcripts, no staged output. Defaults to running all five story
scenarios in sequence; run one at a time with `-s <name>`:

| `-s` name | What it tests |
|---|---|
| `insurance` | Shared recall across a team of agents (Policy → Claims/Compliance/Fraud), plus a cross-namespace isolation check — a different carrier's agent asking the same question correctly gets nothing back |
| `legal` | Reasoning graph — Contract → Compliance → Risk → Negotiation, each card `depends_on` the last, not just repeated recall |
| `regulatory` | 4-agent chain-of-custody (Interpretation → Compliance → Enforcement → Audit) over a real regulation, then a live amendment triggers cascade invalidation and full re-derivation |
| `fedfsr` | Compounding cache-warming against a real 75-page public document (Fed Financial Stability Report) — hit rate measured after each of 4 rounds, plus its own P50/P95 latency footer |
| `provenance` | Human-auditable — walks the real `/cards/{id}/audit` chain hop-by-hop back to the source document, showing content-hash integrity at each link |
| `stress` *(bonus, not part of the 5-scenario story)* | Bulk-commits 10,000 cards, then measures exact/paraphrase resolve latency against pass/fail thresholds. **Note:** at this volume you may see a `WARN cascade invalidation hit safety cap` line during cleanup — this is a bounded safety limit on the dependent-cascade walk firing as designed (all 10,000 cards still purge correctly, zero errors); it is not a failure. |

**Run a specific scenario:**

```bash
docker run --rm ghcr.io/leangridlabs/sce-demo-verticals:latest -s insurance
docker run --rm ghcr.io/leangridlabs/sce-demo-verticals:latest -s legal
docker run --rm ghcr.io/leangridlabs/sce-demo-verticals:latest -s regulatory
docker run --rm ghcr.io/leangridlabs/sce-demo-verticals:latest -s provenance
docker run --rm ghcr.io/leangridlabs/sce-demo-verticals:latest -s fedfsr
docker run --rm ghcr.io/leangridlabs/sce-demo-verticals:latest -s stress
```

**With real OpenAI reasoning** (insurance/legal/regulatory/provenance support
this — otherwise they fall back to clearly-labeled simulated answers):

```bash
# PowerShell (Windows)
$env:OPENAI_API_KEY = "sk-..."
docker run --rm -e OPENAI_API_KEY=$env:OPENAI_API_KEY ghcr.io/leangridlabs/sce-demo-verticals:latest -s insurance

# bash/zsh (macOS/Linux)
export OPENAI_API_KEY="sk-..."
docker run --rm -e OPENAI_API_KEY=$OPENAI_API_KEY ghcr.io/leangridlabs/sce-demo-verticals:latest -s insurance
```

The key is only ever passed at container-launch time via the environment —
it is never baked into the image and never touches SCE's own storage layer.

The VS Code extension (Alpha) is available as a `.vsix` on the
[Releases](https://github.com/leangridlabs/SCE-Public/releases) page.

For a self-hosted deployment (your own config, API keys, and data directory),
contact [leangridlabs.com](https://leangridlabs.com) — this repo is a public
distribution front door, not the deployable source package.

---

## MCP Client Setup

Add one entry to your AI tool's MCP config:

```json
{
  "mcpServers": {
    "sce": {
      "url": "http://<SERVER_IP>:35100/mcp",
      "headers": { "x-api-key": "your-api-key" }
    }
  }
}
```

**VS Code / Copilot:** `%APPDATA%\Code\User\mcp.json` (Windows) or `~/.config/Code/User/mcp.json` (Mac/Linux)  
**Cursor:** Settings → MCP → Add server  
**Claude Desktop:** `claude_desktop_config.json`

Two MCP tools are exposed: `resolve` (query the cache) and `ingest_document` (add a document to the index).

---

## REST API Summary

| Endpoint | Method | Purpose |
|---|---|---|
| `/healthz` | GET | Liveness probe — always 200 while process is alive |
| `/ready` | GET | Readiness probe — 503 until DB + model initialised |
| `/mcp` | POST | MCP Streamable HTTP (JSON-RPC 2.0) |
| `/v1/chat/completions` | POST | OpenAI-wire-compatible gateway — cache-hit synthesis or real-provider passthrough + auto-commit |
| `/resolve` | POST | Query the cache — returns route + card + `audit_url` on hit |
| `/ingest` | POST | Chunk and index a document file |
| `/commit` | POST | Directly commit a reasoning card |
| `/purge` | POST | Remove all cards for a source document |
| `/annotate` | POST | Attach a symbol-level anchor fact to a card |
| `/cards` | GET | List committed cards |
| `/cards/{id}/audit` | GET | Full provenance + integrity check for a single card |
| `/cards/{id}/recall-events` | GET | Every recorded serve of one card — who received it, and when |
| `/recall-events` | GET | Searchable log across all recall events (filter by caller / time range) |
| `/stats` | GET | Live token savings and hit-rate telemetry |
| `/supersession/candidates` | GET | List pending supersession candidates |
| `/supersession/confirm` | POST | Mark old-document cards superseded (requires `supersession_approve`) |
| `/supersession/reject` | POST | Suppress document pair permanently (requires `supersession_approve`) |
| `/acknowledge-regulatory-responsibility` | POST | Record operator regulatory acknowledgment |
| `/debug/triage` | GET | Subsystem health detail and gate configuration |
| `/debug/scoring-export` | GET | Last 500 scoring traces from the optimization log |

Full API reference: [ARCHITECTURE.md](ARCHITECTURE.md#api-reference)

---

## Route Model

Every `/resolve` call returns one of four routes:

| Route | Meaning | Token cost |
|---|---|---|
| `recall` | High-confidence match — card served from local store | **Zero** |
| `graph_assisted` | Partial match — card + graph context returned to compress prompt | **Reduced** |
| `agent_handoff` | No match but structural index available — guided starting set attached | **Reduced** |
| `generic` | No local signal — forwarded to model with no attached context | **Full** |

The resolver never drops a query. Cold misses pay the same latency penalty
as a direct model call plus ~12–18ms local lookup overhead.

---

## Supported Ingest Formats

`.md` `.pdf` `.docx` `.txt` `.csv` `.py` `.rs` `.ts` `.js` `.go` `.java`
`.cs` `.c` `.cpp` `.rb` `.swift` `.kt` `.sql` `.yaml` `.json` `.toml`
`.xlsx` `.tex` and more — 30+ formats total.

---

## Resource Footprint

Default Docker resource cap: **1 GB RAM, 1 CPU**.

Typical warm-state memory for a 5,000-card store: ~250–400 MB.
Cold-miss lookup overhead: ~12–18 ms before forwarding to the model provider.

---

## Documentation

| Document | Audience |
|---|---|
| [ARCHITECTURE.md](ARCHITECTURE.md) | Engineers evaluating the system design |
| [SECURITY.md](SECURITY.md) | Security teams and CISOs |
| [docs/TEST-EVIDENCE.md](docs/TEST-EVIDENCE.md) | Anyone verifying correctness and performance claims |

---

## Status

**Stage 1 — Developer Alpha.** In-room group testing underway.
Enterprise pilot planned for late 2026 pending alpha evidence report.

Contact: [leangridlabs.com](https://leangridlabs.com)
