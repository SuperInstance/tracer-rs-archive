# tracer-rs (archive snapshot)

A slice-of-life artifact from the SuperInstance era. This is a **research/design
package** for an OpenTelemetry-lite distributed tracing library for Rust — not
an implementation. The work stopped at the design phase, and this archive
preserves the package as it was.

> **Take this, fork it, evolve it. No attribution required.** We hope you'll
> build on these ideas, but you owe us nothing. If you want to tell us about
> what you make, great — but don't feel obligated.

## What's here

- `Cargo.toml` — package manifest stub (intentionally minimal)
- `LICENSE` — MIT
- `research/` — ~39,500 words across 10 design documents
- `STATUS.md` — slice-of-life context, written by the archiver

## Start here

Open [`research/INDEX.md`](research/INDEX.md). It lists every research document
in the package with a per-document summary and word count.

If you want to **build the library**, follow the steps in `STATUS.md` under
*"How to use this archive"*. The short version:

1. Read `research/01-research.md` (problem statement and comparison).
2. Read `research/02-architecture.md` (component model).
3. Read `research/03-api.md` (proposed public surface).
4. Start filling in `src/lib.rs` with `Tracer::start_span` from §03.

## Provenance

- **Original commit**: `0f61735` *Initial commit: tracer-rs* (branch `main`).
- **Archived**: 2026-09-18 by an external archiver.
- **Source path**: `C:\reseahclocal\claudeSuperInstance\repos\tracer-rs`.
- **Upstream**: none — this is the first public release. A different repo
  named `SuperInstance/tracer-rs` exists independently.

See `STATUS.md` for the full story, including what this design was good at,
why it was never built, and which mechanisms are worth re-reading.