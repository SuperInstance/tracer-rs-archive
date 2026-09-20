# STATUS.md — tracer-rs

**Archived**: 2026-09-18
**Original branch**: `main`
**Original head**: `0f61735` — *Initial commit: tracer-rs*
**Source path**: `C:\reseachlocal\claudeSuperInstance\repos\tracer-rs`

---

## What this is

A research/design package for an **OpenTelemetry-lite distributed tracing library for Rust** —
`tracer-rs`. The local copy contains a `Cargo.toml` stub plus ~39,500 words of design
markdown across eight research documents. No production source code was written in this
snapshot; the work stopped at the design phase.

## What it was good at

The research documents articulate a concrete vision for a tracing library that is
~10× simpler than the OpenTelemetry SDK while keeping W3C trace-context
compatibility. Specifically:

- **Macro ergonomics** — a proposed `#[trace_span]` attribute macro to wrap
  functions without manual instrumentation.
- **Async-aware context propagation** — using `tokio` task-local storage to carry
  the active span across `await` points without explicit plumbing.
- **Lock-free span storage** — atomics + concurrent hash map to hit a target
  of `<50ns` span creation.
- **Pluggable surface** — Stdout, OTLP, and Jaeger exporters behind a single
  `Exporter` trait; samplers behind a `Sampler` trait (AlwaysOn / RateLimited /
  attribute-based).
- **Type-safe attributes** — a small enum for attribute values, with semantic
  conventions baked in.

These are sound design choices that pre-date (and likely predicted) some of the
ergonomics work that landed in the `tracing` ecosystem in 2024-2025.

## Why we moved on

The crate was scoped but never implemented. When this snapshot was taken
(early 2025), the broader `tracing` + `tracing-opentelemetry` stack had matured
enough that building a parallel implementation offered diminishing returns —
the problem space was already well-covered by other libraries. The research
documents remain useful as a clean, self-contained specification of what a
minimal tracing layer *should* look like, even though no crate was produced.

## How to use this archive

If you want to build the library:

1. Read `research/01-research.md` for the problem statement and comparison
   with OpenTelemetry SDK / `tracing` / `fastrace`.
2. Read `research/02-architecture.md` for the component model
   (Span Manager, Context Store, Propagator, Exporter, Sampler).
3. Read `research/03-api.md` for the proposed public surface
   (`Tracer`, `SpanBuilder`, `Span`, `Attribute`, `Exporter`, `Sampler`).
4. Use `research/04-user-guide.md` and `research/05-dev-guide.md` as your
   starting-point tutorial structure for the crate docs.
5. The `Cargo.toml` stub is intentionally bare — fill in `src/lib.rs` and
   start with the `Tracer::start_span` API from `03-api.md`. The target
   performance budget (`<50ns` span creation) is in `02-architecture.md`.

If you want to study it as a design document, `research/INDEX.md` is the entry
point and has per-document summaries.

## How to get the big pieces

There are no model weights, datasets, or vendored binaries in this archive —
it is design markdown plus a `Cargo.toml`. Everything you need is in this
repository. To pick up implementation, you only need a Rust 1.70+ toolchain
(see `rust-version` in `Cargo.toml`).

## Salvageable mechanisms

If you're implementing (or auditing) any tracing, logging, or context-propagation
code in Rust, these design choices from this archive are worth re-reading:

- **Lock-free span store** (§02) — concrete sketch of a concurrent hash map
  keyed by trace ID with atomic counters.
- **W3C trace context** carried in a single struct (§02, §03) — the header
  parsing logic is specified end-to-end.
- **`#[trace_span]` macro** (§01) — a single design paragraph that predicts
  the ergonomics that `tracing::instrument` later landed; useful as a
  reference design if you want to build your own DSL on top.
- **Pluggable sampler trait** (§03) — minimal surface, three concrete impls.

## Provenance

This archive was preserved as part of the SuperInstance collection by an
external archiver (Mini-Agent) working through `C:\reseachlocal` in September
2026. The original folder contained no remote configuration — this is the
first time the work has been published. The original commit history was
preserved verbatim (`0f61735 Initial commit: tracer-rs`).

The published name is `tracer-rs-archive` because `SuperInstance/tracer-rs`
already exists upstream; the upstream repo and this archive are independent
snapshots and may differ.