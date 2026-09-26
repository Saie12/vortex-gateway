# velarium-gateway — Task Breakdown

> **Status**: Active backlog — tasks ordered by priority
> **Branch flow**: `working` → PR → `development` → `main`
> **Language**: Rust only | **Constraint**: <15 MB binary, <8 MB RAM idle, zero deps where possible

---

## Legend

| Field | Values |
|---|---|
| **Priority** | `P0` (must), `P1` (should), `P2` (could) |
| **Effort** | `S` (1-2 days), `M` (3-5 days), `L` (1-2 weeks), `XL` (2+ weeks) |
| **Type** | `feature`, `refactor`, `docs`, `test`, `chore` |
| **Deps** | External crates required to start this task |

---

## Phase 1 — Foundation ✅ (All Complete)

These tasks are done and verified. No action required.

| ID | Task | Priority | Effort | Status |
|---|---|---|---|---|
| F1-01 | OpenAI-compatible API surface (`chat/completions`, `embeddings`, `models`) | P0 | XL | ✅ Done |
| F1-02 | `RoutingStrategy` trait + `AdaptivePriorityStrategy` | P0 | L | ✅ Done |
| F1-03 | Circuit breaker (Closed/Open/HalfOpen, exponential backoff) | P0 | L | ✅ Done |
| F1-04 | Rate-limit detection (429 + `Retry-After`) | P0 | M | ✅ Done |
| F1-05 | Tiered fallback via `virtual_models` config chains | P0 | M | ✅ Done |
| F1-06 | Hot reload (SIGHUP + `POST /reload`) | P0 | M | ✅ Done |
| F1-07 | Pure-Rust TLS (rustls, no OpenSSL) | P0 | M | ✅ Done |
| F1-08 | OpenTelemetry OTLP export (traces + metrics) | P0 | L | ✅ Done |
| F1-09 | Admin API (`/admin/providers/{name}/{offline|online|reset}`) | P0 | M | ✅ Done |
| F1-10 | CLI subcommands (`serve`, `validate`, `status`, `reload`, `stop`, `provider`) | P0 | L | ✅ Done |
| F1-11 | ASCII status dashboard | P1 | S | ✅ Done |
| F1-12 | Dual-stack IPv4/IPv6 binding | P1 | S | ✅ Done |
| F1-13 | Graceful shutdown (drain SSE) | P1 | S | ✅ Done |
| F1-14 | JSON error format + CORS headers | P1 | S | ✅ Done |
| F1-15 | `x-request-id` correlation + OTel spans | P1 | M | ✅ Done |
| F1-16 | Token counting (non-streaming) | P2 | S | ✅ Done |

---

## Phase 2 — Smart Routing & Observability (In Progress)

### P0 — Must have

| ID | Task | Priority | Effort | Deps | Description |
|---|---|---|---|---|---|
| F2-01 | **Auto-routing: classifier trait** | P0 | M | — | Define `RoutingClassifier` trait in `velarium-gateway-core`. Methods: `classify(&self, request: &ChatCompletionsRequest) -> RoutingDecision`. Return type is an enum: `Auto`, `Pinned(Model)`, `Fallback(virtual_model_name)`. Pluggable — default impl is rule-based. |
| F2-02 | **Auto-routing: rule-based classifier** | P0 | M | F2-01 | Implement `RuleBasedClassifier` using keyword matching. 5 categories: `simple`, `complex`, `coding`, `creative`, `reasoning`. Config-driven keywords in `config.toml` under `[routing.auto_classifier]`. Zero new deps. |
| F2-03 | **Auto-routing: integrate with router** | P0 | M | F2-01, F2-02 | When request omits `model` field, use classifier to select virtual model. Thread request body context into the `RoutingStrategy::select()` call. |
| F2-04 | **RTK token compressor** | P0 | L | — | Implement `RtkCompressor` — identifies repetitive patterns (identical log lines, repeated JSON entries) in tool output and collapses them. Pure Rust string operations. Config-gated: `[compression.rtk]` section with `enabled = false` default. |
| F2-05 | **Caveman token compressor** | P0 | L | — | Implement `CavemanCompressor` — summarizes verbose prose prompts into concise equivalents using rule-based shortening (max sentence length, redundant phrase removal). Pure Rust. Config-gated. |
| F2-06 | **Compression pipeline integration** | P0 | M | F2-04, F2-05 | Insert compression layer between Axum handler and upstream request. Decompress is transparent (responses pass through unchanged to client). |

### P1 — Should have

| ID | Task | Priority | Effort | Deps | Description |
|---|---|---|---|---|---|
| F2-07 | **Exact cache: in-memory store** | P1 | M | — | `HashMap<Sha256, CachedResponse>` with TTL eviction. `CachedResponse` stores: response body, headers, timestamp. Clean eviction via `tokio::time::interval` sweep. |
| F2-08 | **Exact cache: integration** | P1 | M | F2-07 | Hash request body (model + normalized messages + params) → check cache → return cached or forward to upstream. Store response in cache. TTL configurable per virtual model (default 60s). |
| F2-09 | **Exact cache: control headers** | P1 | S | F2-07 | Honour `Cache-Control: no-cache` and `Cache-Control: no-store` headers to bypass cache. Pass `X-Cache: HIT/MISS` response header. |
| F2-10 | **Per-key rate limiting** | P1 | M | — | Token bucket per API key. Config in `[rate_limits]` section: `rps = N`, `burst = N`. Uses `std::collections::HashMap` with `tokio::time` for refill tick. |
| F2-11 | **Per-key token budgets** | P1 | M | F2-10 | Track tokens per API key in a 30-day rolling window. Return 429 with `Retry-After` when budget exhausted. Reset at calendar month boundary. |
| F2-12 | **Prometheus metrics endpoint** | P1 | M | — | `#[cfg(feature = "prometheus")]` — add `GET /metrics` in Prometheus text format. Metrics: `velarium_gateway_requests_total`, `velarium_gateway_tokens_total`, `velarium_gateway_circuit_state`, `velarium_gateway_upstream_latency_seconds`. Uses `prometheus` crate as optional dep. |
| F2-13 | **Audit logging: structured JSON** | P1 | M | — | Log one JSON line per request: timestamp, request_id, model, provider, tokens_in, tokens_out, latency_us, status_code. Config-gated. No message bodies by default. |
| F2-14 | **PII redaction** | P1 | M | — | Regex-based redaction of emails, phone numbers, credit cards, SSNs. Configurable patterns per jurisdiction (US, EU, APAC). Opt-in via config. Pure Rust `regex` crate (already a transitive dep). |

### P2 — Could have

| ID | Task | Priority | Effort | Deps | Description |
|---|---|---|---|---|---|
| F2-15 | **Prompt injection detection** | P2 | M | — | Heuristic detection of system-prompt override attempts. Pattern list: `ignore.*instructions`, `new.*instructions`, `disregard.*above`, etc. Flag suspicious requests in logs, optionally block via config. |
| F2-16 | **Cache stats in dashboard** | P2 | S | F2-07 | Add cache hit/miss rates to `velarium-gateway status` ASCII output. Show per-virtual-model breakdown. |
| F2-17 | **Compression stats in dashboard** | P2 | S | F2-04, F2-05 | Show token savings percentage per compression strategy in status dashboard. |

---

## Phase 3 — Enterprise & Scale (Planned)

| ID | Task | Priority | Effort | Deps | Description |
|---|---|---|---|---|---|
| F3-01 | **Semantic caching** | P2 | XL | F2-07 | Embedding-based cache lookup. Requires local embedder (ggml-based, no external API). |
| F3-02 | **Multi-key per provider** | P1 | L | — | Support multiple API keys per provider with priority/failure ordering. BYOK-style key management. |
| F3-03 | **Model capability registry** | P2 | M | — | Per-model metadata: context window, tool support, vision, pricing tiers. Enables smarter routing decisions. |
| F3-04 | **Embedding routing** | P2 | M | — | Route `/v1/embeddings` to providers optimized for embedding throughput. |
| F3-05 | **WASM filter hooks** | P2 | XL | — | Allow users to write custom request/response filters in WASM (Rust/Python-compiled). Feature-gated. |

---

## How to claim a task

1. Comment on the GitHub issue (create one if needed): `@Saie12 I'm starting F2-XX`
2. Create a branch: `git checkout -b feat/f2-xx-short-name working`
   (branch from `working`, so it tracks the active dev branch)
3. Implement → `cargo fmt` → `cargo clippy -- -D warnings` → `cargo test --all-features`
4. Commit with conventional format: `feat: F2-XX — <short description>`
5. Push → open PR → `development` base → CI must pass → merge

## Conventions

- **No `unwrap()`/`expect()`** in user-facing paths — use `VelariumGatewayError`
- **Rust edition 2021** — no nightly features
- **Pure Rust TLS** — never add `openssl-sys`
- **Feature flags** — non-core features gated behind `#[cfg(feature = "...")]`
- **Zero disk persistence** — state must be recoverable from config + upstream
- **Lock-free hot path** — preserve `probe_in_flight` atomic pattern in circuit breaker
