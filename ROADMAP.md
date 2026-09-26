# velarium-gateway — Roadmap

> **Status**: Active
> **Author**: Saie12
> **Updated**: 2026-09-26
> **Branch flow**: `working` → PR → `development` → (merge to `main` when ready)

---

## Vision

A minimalist, memory-only, OpenAI-compatible API gateway that routes LLM traffic
across a tiered pool of upstream providers with intelligent failover, zero
external dependencies, and the smallest possible footprint.

Built for: edge devices, developer workstations, CI/CD agents, and any environment
where RAM is scarce and reliability matters.

---

## Release Phases

### Phase 1 — Foundation ✅ (Complete — v0.1.12)

**Status**: Done. Released on `main` and `development` branches.

| Capability | Description | Status |
|---|---|---|
| OpenAI-compatible API | `chat/completions`, `embeddings`, `models` | ✅ Complete |
| Adaptive routing | `RoutingStrategy` trait + `AdaptivePriorityStrategy` | ✅ Complete |
| Circuit breakers | Closed/Open/HalfOpen with exponential backoff | ✅ Complete |
| Rate-limit handling | 429 detection + `Retry-After` parsing | ✅ Complete |
| Tiered failover | `virtual_models` chains in `config.toml` | ✅ Complete |
| Hot reload | SIGHUP + `POST /reload` | ✅ Complete |
| Zero persistence | All state in-memory, lock-free atomics | ✅ Complete |
| Pure-Rust TLS | `rustls` only, no OpenSSL | ✅ Complete |
| OpenTelemetry | OTLP/HTTP JSON traces + metrics | ✅ Complete |
| W3C trace context | traceparent extraction/injection | ✅ Complete |
| Admin API | `/admin/providers/{name}/{offline\|online\|reset}` | ✅ Complete |
| CLI | `serve`, `validate`, `status`, `reload`, `stop`, `provider` subcommands | ✅ Complete |
| ASCII dashboard | `velarium-gateway status` with routing table | ✅ Complete |
| Dual-stack networking | `bind_family` (ipv4/ipv6/dual) | ✅ Complete |
| Graceful shutdown | Drains in-flight SSE on SIGTERM | ✅ Complete |
| JSON errors + CORS | OpenAI-compatible error format, `ACAO: *` | ✅ Complete |
| Request tracing | `x-request-id` correlation + OTel spans | ✅ Complete |

### Phase 2 — Smart Routing & Observability (In Progress — v0.2.0)

**Target branch**: `working`
**Status**: Planning complete (see [PRD.md](./PRD.md) for full specs)

| Capability | Milestone | Status |
|---|---|---|
| **F1: Auto-routing** | v0.2.0-alpha | ☐ Not started |
| Prompt classification (rule-based, no ML) | v0.2.0-alpha | ☐ Not started |
| Per-model routing configs | v0.2.0-alpha | ☐ Not started |
| **F2: Token compression** | v0.2.0-alpha | ☐ Not started |
| RTK (repetitive token killer) for tool output | v0.2.0-beta | ☐ Not started |
| Caveman (prose compression) | v0.2.0-beta | ☐ Not started |
| **F3: Exact caching** | v0.2.0-beta | ☐ Not started |
| In-memory TTL-based cache | v0.2.0-beta | ☐ Not started |
| Cache control headers | v0.2.0-beta | ☐ Not started |
| **F4: Per-key rate limiting** | v0.2.0 | ☐ Not started |
| Token bucket per API key | v0.2.0-rc1 | ☐ Not started |
| Per-key token budgets | v0.2.0-rc1 | ☐ Not started |
| **F6: Prometheus metrics** | v0.2.0-rc1 | ☐ Not started |
| Feature-gated metrics endpoint | v0.2.0-rc1 | ☐ Not started |
| **F5: Guardrails** | v0.2.0 (optional) | ☐ Not started |
| PII redaction (regex-based) | v0.2.0-rc2 | ☐ Not started |
| Prompt injection heuristics | v0.2.0-rc2 | ☐ Not started |
| **F7: Audit logging** | v0.2.0 (optional) | ☐ Not started |
| Structured JSON request logs | v0.2.0-rc2 | ☐ Not started |

### Phase 3 — Enterprise & Scale (v0.3.0+)

**Status**: Planning stage

| Capability | Status |
|---|---|
| Semantic caching (embedding-based) | ☐ Planned |
| Multi-key per provider (BYOK-style) | ☐ Planned |
| Model capability registry | ☐ Planned |
| Embedding-specific routing | ☐ Planned |
| Request/response body transformation hooks | ☐ Planned |
| Plugin system (Lua/WASM filters) | ☐ Planned |

### Phase 4 — Long-term (v0.4.0+)

| Capability | Status |
|---|---|
| Fine-grained RBAC (per-key model access) | ☐ Future |
| Request queuing + priority scheduling | ☐ Future |
| Upstream response streaming optimization | ☐ Future |
| Schema validation for tool calls | ☐ Future |

---

## Milestone Timeline

```
Q1 2026  Q2 2026  Q3 2026  Q4 2026  Q1 2027
  │        │        │        │        │
Phase 1 ✅  Phase 2 🧩   Phase 2 🧩   Phase 3 📋  Phase 4 📋
           v0.2.0-α   v0.2.0-β/RC   v0.3.0       v0.4.0
```

- 🧩 In progress
- ✅ Complete
- 📋 Planned

---

## How to Track Progress

- **Issues**: https://github.com/Saie12/velarium-gateway/issues
- **Development branch**: `development` — latest complete features
- **Working branch**: `working` — active development, PR'd to `development`
- **CI**: `cargo fmt --check`, `cargo clippy -- -D warnings`, `cargo test --all-features`
- **Coverage gate**: workspace ≥43%, `velarium-gateway` ≥36%, `velarium-gateway-core` ≥55%
