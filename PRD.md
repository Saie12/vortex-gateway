# velarium-gateway — Product Requirements Document (Phase 2)

> **Status**: Draft — Ready for planning
> **Author**: Saie12
> **Created**: 2026-09-26
> **Target release**: v0.2.0
> **Language**: Rust (no other runtimes)

---

## 1. Vision

velarium-gateway is a minimalist, memory-only API gateway that presents a single
OpenAI-compatible interface while routing requests across a tiered pool of upstream
LLM providers. It is built for **resource-constrained environments** — edge devices,
developer workstations, background daemons — where RAM is scarce and every byte counts.

This PRD covers Phase 2: **Smart Routing & Observability** — the set of features
that transform velarium-gateway from a failover proxy into an intelligent,
observable gateway that developers and operators can trust in production.

---

## 2. Goals

| # | Goal | Metric |
|---|---|---|
| G1 | Enable automatic model/provider selection per request | ≥80% of requests routed to an optimal (non-primary) provider when the default is unhealthy or expensive |
| G2 | Reduce token spend through compression | ≥15% token reduction on repetitive tool output; ≥10% on prose |
| G3 | Eliminate redundant upstream calls via caching | ≥30% cache hit rate on repeated prompts within 1h TTL |
| G4 | Provide per-client rate limiting and budget guardrails | Per-key token budgets enforced; no overspend beyond configured limit |
| G5 | Surface actionable observability data | Prometheus metrics endpoint; structured request logs available |

---

## 3. Non-Goals

| # | Out of scope | Reason |
|---|---|---|
| N1 | Protocol translation (non-OpenAI APIs) | Keep the binary small; expect upstreams to be OpenAI-compatible |
| N2 | Persistent state / local database | Zero disk persistence is a core invariant; state lives in memory |
| N3 | Web UI / dashboard | Keep the binary headless; ops use existing observability stacks |
| N4 | Fine-tuning / model training | Gateway only — no ML workloads |
| N5 | Hosted/managed service | Self-hosted only — this is a binary you run |

---

## 4. Current State (Phase 1 — Complete, v0.1.12)

### Done

- [x] **OpenAI-compatible API surface**: `POST /v1/chat/completions`, `POST /v1/embeddings`, `GET /v1/models`
- [x] **Adaptive Priority Routing Strategy**: `RoutingStrategy` trait with pluggable implementations
- [x] **Circuit breakers**: Closed/Open/HalfOpen state machine with exponential backoff, thuderning-herd protection via lock-free atomic probes
- [x] **Rate-limit detection**: 429 handling with `Retry-After` header parsing
- [x] **Tiered fallback**: `virtual_models` chain in `config.toml`
- [x] **Hot reload**: SIGHUP + `POST /reload` — swap provider pool without dropping connections
- [x] **In-memory state**: zero disk persistence; lock-free atomics for counters
- [x] **Pure-Rust TLS**: `rustls` only — no OpenSSL dependency
- [x] **OpenTelemetry**: OTLP/HTTP JSON export (traces + metrics)
- [x] **W3C trace context**: traceparent extraction/injection for continuous spans
- [x] **Admin API**: `POST /admin/providers/{name}/{offline|online|reset}`
- [x] **CLI subcommands**: `serve`, `validate`, `status`, `reload`, `stop`, `provider {list|offline|online|reset}`
- [x] **ASCII status dashboard**: `velarium-gateway status` shows routing table + per-provider counters
- [x] **Dual-stack binding**: `bind_family` config (ipv4/ipv6/dual)
- [x] **Graceful shutdown**: drains in-flight SSE streams on SIGTERM
- [x] **Graceful upstream timeouts**: configurable `upstream_timeout_secs` (default 5s)
- [x] **CORS headers**: `Access-Control-Allow-Origin: *` on public endpoints
- [x] **JSON error format**: OpenAI-compatible `{"error": {...}}` on all errors
- [x] **`x-request-id` correlation**: unique per request, propagated to logs and OTel spans
- [x] **Token counting**: from upstream JSON responses (non-streaming)

### Verified

- `cargo fmt --check` ✅
- `cargo clippy --workspace --all-targets -- -D warnings` ✅
- `cargo test --workspace --all-features` — 10/10 pass ✅

---

## 5. Proposed Features (Phase 2)

### F1: Auto-Routing (Prompt Classification) — High Priority

**Problem**: Users currently specify a model explicitly. If the model is unhealthy
or expensive, the request still goes there until circuit-breaker failover kicks in.

**Solution**: Implement a classification layer that inspects the incoming request
body and routes to the appropriate `virtual_model` based on task type and cost
preference.

**Approach (minimal footprint)**:
- Rule-based classifier (regex + keyword matching) — no ML model, no extra deps
- 5 task categories: `simple`, `complex`, `coding`, `creative`, `reasoning`
- Configurable via `config.toml`:
  ```toml
  [routing]
  default_strategy = "auto"      # or "priority" (default)
  
  [routing.auto_classifier]
  simple_prompt = ["what", "who", "list", "summarize"]
  coding_prompt = ["code", "function", "fix", "error", "compile"]
  creative_prompt = ["write", "story", "poem", "design"]
  reasoning_prompt = ["prove", "explain why", "analyze", "compare"]
  ```

**Acceptance criteria**:
- `/v1/chat/completions` without a `model` field routes to `auto` virtual model
- Classifier correctly identifies task type ≥85% on benchmark set
- Configurable per virtual model
- Zero additional dependencies

### F2: Request-Level Token Compression — High Priority

**Problem**: Repetitive tool output (build logs, test runs, diffs) and verbose
prompts bloat token usage, increasing cost and latency.

**Solution**: Implement two compression strategies that run **before** forwarding
to the upstream provider:

1. **Caveman** — compresses prose prompts (long summaries, verbose instructions)
   into concise equivalents. Rule-based, not ML.
2. **RTK** (Repetitive Token Killer) — identifies and collapses repetitive patterns
   in tool output (e.g., identical log lines, repeated JSON entries).

**Approach (minimal footprint)**:
- Compression happens in the request pipeline (Axum handler → compression middleware → upstream)
- Decompression is transparent for responses (no change to client)
- Configurable per-model: enable Caveman on prose-heavy models, RTK on coding tasks
- No external dependencies — pure Rust string manipulation

**Acceptance criteria**:
- ≥15% token reduction on repetitive tool output (RTK)
- ≥10% token reduction on verbose prompts (Caveman)
- Compression disabled by default (opt-in via config)
- Streaming responses decompress transparently

### F3: Caching (Exact + Semantic) — Medium Priority

**Problem**: Identical or semantically-similar prompts result in redundant
upstream calls, wasting tokens and adding latency.

**Solution**:
- **Exact caching**: Hash the normalized request body (model + messages + params),
  return cached response if hit. TTL-based eviction (short-lived, in-memory).
- **Semantic caching**: Embed the prompt, check vector similarity against recent
  requests. Requires an embedding backend (self-hosted, tiny model).

**Approach (minimal footprint)**:
- Exact cache: in-memory `HashMap<Sha256, Response>` with TTL eviction (configurable)
- Semantic cache: deferred to Phase 3 or pluggable (requires embedder)
- Cache control via request headers: `Cache-Control: no-cache` bypasses cache

**Acceptance criteria**:
- Exact cache hit returns response without upstream call
- TTL configurable per virtual model (default: 60s)
- Cache stats exposed via `/status` and `velarium-gateway status`
- Zero additional dependencies for exact cache

### F4: Per-Key Rate Limiting & Budget Guardrails — Medium Priority

**Problem**: Current rate limiting only handles upstream 429s. There's no
per-client throttling or budget enforcement.

**Solution**:
- Token-bucket rate limiter per API key (configurable RPS and burst)
- Per-key token budget (monthly/daily limit, enforced via in-memory counters)
- Budget exhaustion returns 429 with retry-after

**Approach (minimal footprint)**:
- In-memory token bucket (no Redis/external dependency)
- Key tracking built on existing `allowed_api_key_hashes` concept
- Configurable in `config.toml` under `[rate_limits]` section

**Acceptance criteria**:
- Per-key RPS and burst limits enforced
- Per-key token budgets enforced
- 429 responses include accurate `Retry-After`
- No additional dependencies

### F5: Guardrails (PII Redaction, Injection Detection) — Medium Priority

**Problem**: Raw prompts and responses may contain PII or be vulnerable to
prompt injection.

**Solution**:
- **PII redaction**: Regex-based redaction of email addresses, phone numbers,
  credit card numbers, SSNs/NINs — configurable per region
- **Prompt injection detection**: Heuristic detection of system-prompt override
  attempts (e.g., `Ignore your instructions`, `New instructions:`) — flags or blocks
  suspicious requests before forwarding

**Approach (minimal footprint)**:
- Pure Rust regex patterns — no ML classifier
- Disabled by default, opt-in via config
- Configurable sensitivity levels

**Acceptance criteria**:
- PII patterns configurable per jurisdiction (US, EU, APAC)
- Injection detection flags ≥90% of common injection patterns
- No false positives on legitimate coding queries
- Zero additional dependencies

### F6: Prometheus Metrics Endpoint — Low Priority

**Problem**: OpenTelemetry export requires an external collector. Some users want
a simple `/metrics` endpoint for Prometheus scraping.

**Solution**: Add `GET /metrics` endpoint that exposes:
- `velarium_gateway_requests_total{provider, model, status}`
- `velarium_gateway_tokens_total{provider, direction}` (input/output)
- `velarium_gateway_circuit_state{provider, state}` (0=healthy, 1=cooldown, 2=tripped)
- `velarium_gateway_upstream_latency_seconds{provider}` (histogram buckets)

**Approach (minimal footprint)**:
- Optional feature flag — only compiled when `metrics-prometheus` feature is enabled
- No additional HTTP server — served on same Axum router
- No additional dependencies beyond `prometheus` crate

**Acceptance criteria**:
- Prometheus-compatible text format
- Metrics update atomically (lock-free or fast RwLock)
- Feature gated behind `#[cfg(feature = "prometheus")]`

### F7: Request/Response Audit Logging — Low Priority

**Problem**: Current logging is console-only with verbosity flags. No structured
audit trail for compliance or debugging.

**Solution**: Optional structured JSON logging of requests/responses (excluding
full message bodies by default — only metadata, model, token counts, provider, latency).

**Approach (minimal footprint)**:
- Configurable via `config.toml` — log to stdout, file, or syslog
- JSON structured — one line per request
- Exclude message bodies by default; opt-in for full logging
- No database dependency

**Acceptance criteria**:
- JSON log format includes: timestamp, request_id, model, provider, tokens_in, tokens_out, latency, status
- Configurable output destination
- PII-safe by default (no message bodies)

---

## 6. Technical Constraints

| Constraint | Detail |
|---|---|
| **Language** | Rust only — no Python, Node, or other runtimes |
| **Binary size** | Stripped release binary must stay <15 MB (current: ~14 MB) |
| **Memory** | Must run comfortably on 512 MB RAM systems |
| **Dependencies** | Prefer stdlib; minimize external crates; pure-Rust TLS only |
| **Persistence** | Zero disk persistence — all state in memory |
| **Compilation** | Must compile on stable Rust (edition 2021) |
| **Target** | Linux x86_64 + aarch64 (primary), macOS + Windows (secondary) |
| **Feature flags** | Non-core features gated behind optional Cargo features |

---

## 7. Success Metrics

| Metric | Current (v0.1.12) | Target (v0.2.0) |
|---|---|---|
| Test coverage | 46.17% workspace | ≥55% workspace |
| Binary size (release, stripped) | ~14 MB | <15 MB |
| RAM usage (idle) | <5 MB | <8 MB |
| Request latency (p99) | ~200ms | <250ms (with caching enabled) |
| Supported providers (config) | 6 | 6+ (user-configurable, no built-in limit) |
