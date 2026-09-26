# CLAUDE.md - Developer Guide for `velarium-gateway`

This guide outlines the commands and guidelines for building, testing, linting, and maintaining the `velarium-gateway` (Velarium Gateway) codebase.

## Quick Links

| Doc | Purpose |
|---|---|
| [PRD.md](./PRD.md) | Product Requirements — Phase 2 features, goals, constraints |
| [ROADMAP.md](./ROADMAP.md) | Release phases, milestones, status of each capability |
| [TASKS.md](./TASKS.md) | Prioritized task breakdown with effort estimates and deps |
| [docs/architecture.md](./docs/architecture.md) | Technical architecture, concurrency model, circuit breaker rules |
| [docs/providers.md](./docs/providers.md) | Supported free-tier providers and model names |
| [implementation_plan.md](./implementation_plan.md) | Original technical implementation plan |

## Branch Workflow

```
working → (PR) → development → (merge to main when ready)
```

- **`working`** — active development. Branch from here for features.
- **`development`** — integration branch. PRs target this.
- **`main`** — stable releases only.

## Build Commands

- **Check Compilation**: `cargo check`
- **Build Development Binary**: `cargo build`
- **Build Size-Optimized Release Binary**: `cargo build --release` (stripped binary size under 15 MB)

## Test Commands

- **Run All Tests**: `cargo test`
- **Run Unit Tests only**: `cargo test --lib`
- **Run Latency Performance Test**: `cargo test --test performance -- --nocapture`

## Formatting & Linting Commands

- **Enforce Code Formatting**: `cargo fmt`
- **Verify Formatting Check**: `cargo fmt --check`
- **Strict Clippy Lint Gate**: `cargo clippy --workspace --all-targets --all-features -- -D warnings`

## Code Guidelines

- **Pure Rust TLS**: Enforce pure-Rust TLS by compiling with reqwest's `rustls-tls` feature to avoid dynamically linking `OpenSSL` on targeted edge routers (OpenWrt/musl).
- **Zero unwraps**: Never use `.unwrap()` or `.expect()` in user-facing paths. Convert all errors to `VelariumGatewayError` or map them gracefully.
- **Lock-Free Hot Path**: Always preserve the lock-free atomic `probe_in_flight` permit when evaluating cooled down `HalfOpen` providers to defend against thundering herds under shared locks.
- **Feature flags**: Non-core features (Prometheus metrics, compression, caching) must be gated behind `#[cfg(feature = "...")]` to keep the default binary small.
- **Zero disk persistence**: All runtime state must live in memory. No local database, no file-based state files.

## Project Layout

```
crates/
  velarium-gateway/          # CLI binary (Axum server, routes, signal handling)
  velarium-gateway-core/     # Core library (config, router, circuit breaker, telemetry)
docs/
  architecture.md            # Concurrency model, circuit breaker rules, telemetry
  providers.md             # Free-tier provider guide
.github/workflows/
  ci.yml                     # Format, lint, test, coverage, security
  release.yml                # Binary releases via cargo-dist
  publish.yml                # crates.io publishing
  bump-and-tag.yml           # Version bump + tag on main push
```
