# tracer-rs: Developer Guide

## Welcome, Contributor!

This guide is for developers who want to contribute to **tracer-rs**, build custom exporters, or understand the internals.

---

## Part 1: Architecture Overview

### System Architecture

```
Application Layer
├── #[trace_span] Macro (Procedural macro)
└── Tracer API (Builder pattern)

Core Layer
├── Span Manager (Create/end spans)
├── Context Store (Tokio task-local storage)
└── Propagator (W3C trace context)

Export Layer
├── Stdout Exporter (Console output)
├── OTLP Exporter (gRPC protocol)
└── Jaeger Exporter (UDP protocol)
```

**Key Design Decisions**:
- **Lock-free**: All span operations use atomics
- **Async-first**: Tokio task-local storage for context
- **Zero-cost**: Compile-time elimination via feature flag
- **Extensible**: Pluggable exporters, samplers, propagators

---

### Module Structure

```
tracer-rs/
├── src/
│   ├── lib.rs              # Public API
│   ├── tracer.rs           # Tracer struct
│   ├── span.rs             # Span and SpanBuilder
│   ├── context.rs          # TraceContext, Propagator
│   ├── attribute.rs        # Attribute types
│   ├── exporter.rs         # Exporter trait
│   ├── exporter/
│   │   ├── stdout.rs       # Stdout exporter
│   │   ├── otlp.rs         # OTLP exporter
│   │   └── jaeger.rs       # Jaeger exporter
│   ├── sampler.rs          # Sampler trait
│   ├── storage.rs          # Span storage (lock-free)
│   ├── macro.rs            # #[trace_span] macro
│   ├── middleware.rs       # HTTP middleware
│   └── semantic.rs         # Semantic conventions
├── Cargo.toml
└── README.md
```

---

## Part 2: Development Setup

### Prerequisites

- Rust 1.75+ (for async trait support)
- Tokio 1.0+ (for async runtime)
- Optional: Docker (for Jaeger/Zipkin testing)

---

### Clone and Build

```bash
# Clone repository
git clone https://github.com/SuperInstance/tracer-rs
cd tracer-rs

# Build
cargo build

# Run tests
cargo test

# Run with all features
cargo build --all-features

# Run clippy
cargo clippy --all-targets
```

---

### Development Dependencies

```toml
[dev-dependencies]
tokio = { version = "1", features = ["full"] }
tokio-test = "0.4"
criterion = "0.5"  # Benchmarks
proptest = "1.0"   # Property testing
```

---

### Feature Flags

```toml
[features]
default = ["exporter-stdout", "exporter-otlp", "exporter-jaeger"]

# Exporters
exporter-stdout = []  # Stdout export
exporter-otlp = ["prost", "tonic"]  # OTLP/gRPC
exporter-jaeger = ["thrift"]  # Jaeger UDP

# Sampling
sampler-probabilistic = []
sampler-rate-limited = []

# Tracing macro
trace-span = []

# Zero-cost tracing
tracing = ["trace-span"]  # Enable all tracing
```

---

## Part 3: Building Custom Exporters

### Exporter Trait

```rust
use std::sync::Arc;
use crate::span::Span;

pub trait Exporter: Send + Sync {
    /// Export a span
    fn export(&self, span: Arc<Span>) -> Result<(), ExportError>;

    /// Shutdown exporter (flush pending spans)
    fn shutdown(&self) -> Result<(), ExportError>;
}
```

---

### Example: JSON File Exporter

```rust
use std::fs::OpenOptions;
use std::io::Write;
use std::sync::{Arc, Mutex};
use crate::span::Span;
use crate::exporter::{Exporter, ExportError};

pub struct JsonFileExporter {
    file: Arc<Mutex<std::fs::File>>,
}

impl JsonFileExporter {
    pub fn new(path: &str) -> Result<Self, ExportError> {
        let file = OpenOptions::new()
            .create(true)
            .append(true)
            .open(path)
            .map_err(|e| ExportError::Io(e))?;

        Ok(Self {
            file: Arc::new(Mutex::new(file)),
        })
    }
}

impl Exporter for JsonFileExporter {
    fn export(&self, span: Arc<Span>) -> Result<(), ExportError> {
        let json = serde_json::to_string_pretty(&*span)
            .map_err(|e| ExportError::Serialize(e.to_string()))?;

        let mut file = self.file.lock().unwrap();
        writeln!(file, "{}", json)
            .map_err(|e| ExportError::Io(e))?;

        Ok(())
    }

    fn shutdown(&self) -> Result<(), ExportError> {
        // Flush file
        let file = self.file.lock().unwrap();
        file.sync_all()
            .map_err(|e| ExportError::Io(e))?;
        Ok(())
    }
}
```

---

### Example: Prometheus Remote Write Exporter

```rust
use reqwest::Client;
use crate::span::Span;

pub struct PrometheusExporter {
    endpoint: String,
    client: Client,
}

impl PrometheusExporter {
    pub fn new(endpoint: &str) -> Self {
        Self {
            endpoint: endpoint.to_string(),
            client: Client::new(),
        }
    }
}

impl Exporter for PrometheusExporter {
    fn export(&self, span: Arc<Span>) -> Result<(), ExportError> {
        // Convert span to Prometheus metrics
        let metrics = self.to_prometheus_metrics(&span);

        // Send to Prometheus
        let response = self.client
            .post(&self.endpoint)
            .body(metrics)
            .send()
            .await
            .map_err(|e| ExportError::Send(e.to_string()))?;

        Ok(())
    }

    fn shutdown(&self) -> Result<(), ExportError> {
        Ok(())
    }
}
```

---

## Part 4: Building Custom Samplers

### Sampler Trait

```rust
use crate::trace_id::TraceId;
use crate::sampler::SamplingDecision;

pub trait Sampler: Send + Sync {
    /// Decide whether to sample a trace
    fn should_sample(&self, trace_id: TraceId) -> SamplingDecision;
}
```

---

### Example: Dynamic Rate Sampler

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::time::{Duration, Instant};

pub struct DynamicRateSampler {
    target_rate: AtomicU64,
    current_rate: AtomicU64,
    last_reset: AtomicU64,
}

impl DynamicRateSampler {
    pub fn new(initial_rate: u64) -> Self {
        Self {
            target_rate: AtomicU64::new(initial_rate),
            current_rate: AtomicU64::new(0),
            last_reset: AtomicU64::new(Instant::now().elapsed().as_secs()),
        }
    }

    pub fn set_rate(&self, rate: u64) {
        self.target_rate.store(rate, Ordering::Relaxed);
    }
}

impl Sampler for DynamicRateSampler {
    fn should_sample(&self, trace_id: TraceId) -> SamplingDecision {
        let now = Instant::now().elapsed().as_secs();
        let last_reset = self.last_reset.load(Ordering::Relaxed);

        // Reset counter every second
        if now != last_reset {
            self.current_rate.store(0, Ordering::Relaxed);
            self.last_reset.store(now, Ordering::Relaxed);
        }

        // Check if under rate limit
        let current = self.current_rate.fetch_add(1, Ordering::Relaxed);
        let target = self.target_rate.load(Ordering::Relaxed);

        if current < target {
            SamplingDecision::Record
        } else {
            SamplingDecision::Drop
        }
    }
}
```

---

### Example: Attribute-Based Sampler

```rust
use std::collections::HashMap;
use crate::attribute::Attribute;
use crate::span::Span;

pub struct AttributeSampler {
    predicate: Box<dyn Fn(&HashMap<String, Attribute>) -> bool + Send + Sync>,
}

impl AttributeSampler {
    /// Sample only error spans
    pub fn errors_only() -> Self {
        Self {
            predicate: Box::new(|attrs| {
                attrs
                    .get("http.status_code")
                    .and_then(|attr| match attr {
                        Attribute::Int(code) => Some(*code >= 500),
                        _ => None,
                    })
                    .unwrap_or(false)
            }),
        }
    }

    /// Sample only slow spans (>1s)
    pub fn slow_only() -> Self {
        Self {
            predicate: Box::new(|attrs| {
                attrs
                    .get("duration_ms")
                    .and_then(|attr| match attr {
                        Attribute::Int(duration) => Some(*duration > 1000),
                        _ => None,
                    })
                    .unwrap_or(false)
            }),
        }
    }
}
```

---

## Part 5: Testing Guide

### Unit Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::Tracer;

    #[test]
    fn test_span_creation() {
        let span = Tracer::start_span("test").build();
        assert_eq!(span.name(), "test");
        assert!(span.duration().as_millis() < 100);
    }

    #[test]
    fn test_parent_child_relationship() {
        let parent = Tracer::start_span("parent").build();
        let child = parent.child("child").build();

        assert_eq!(child.trace_id(), parent.trace_id());
        assert_ne!(child.span_id(), parent.span_id());
    }

    #[test]
    fn test_trace_context_parsing() {
        let ctx = TraceContext::from_traceparent(
            "00-7a3b5c9d8e1f2a4b3c4d5e6f7a8b9c0d-1a2b3c4d5e6f7a8b-01"
        ).unwrap();

        assert_eq!(ctx.trace_id().to_hex(), "7a3b5c9d8e1f2a4b3c4d5e6f7a8b9c0d");
        assert_eq!(ctx.span_id().to_hex(), "1a2b3c4d5e6f7a8b");
        assert!(ctx.is_sampled());
    }
}
```

---

### Integration Tests

```rust
#[tokio::test]
async fn test_http_context_propagation() {
    use axum::{Router, routing::get};
    use tracer::middleware::TraceLayer;

    let app = Router::new()
        .route("/test", get(|| async { "OK" }))
        .layer(TraceLayer::new());

    // Make request
    let client = reqwest::Client::new();
    let response = client
        .get("http://localhost:3000/test")
        .header("traceparent", "00-7a3b5c9d...-1a2b3c4d...-01")
        .send()
        .await
        .unwrap();

    // Verify traceparent in response
    assert!(response.headers().contains_key("traceparent"));
}
```

---

### Property-Based Tests

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn test_trace_context_roundtrip(trace_id in any::<u128>(), span_id in any::<u64>) {
        let ctx = TraceContext::new(trace_id, span_id);
        let traceparent = ctx.to_traceparent();
        let parsed = TraceContext::from_traceparent(&traceparent).unwrap();

        assert_eq!(ctx, parsed);
    }

    #[test]
    fn test_probabilistic_sampler_ratio(ratio in 0.0f64..1.0) {
        let sampler = ProbabilisticSampler::new(ratio);
        let trace_id = TraceId::new();

        let decision = sampler.should_sample(trace_id);
        // Verify distribution matches ratio (with tolerance)
        // ...
    }
}
```

---

### Benchmark Tests

```rust
use criterion::{black_box, criterion_group, criterion_main, Criterion, BenchmarkId};

fn bench_span_creation(c: &mut Criterion) {
    let mut group = c.benchmark_group("span_creation");

    group.bench_function("root_span", |b| {
        b.iter(|| {
            black_box(Tracer::start_span("test").build());
        });
    });

    group.bench_function("child_span", |b| {
        let parent = Tracer::start_span("parent").build();
        b.iter(|| {
            black_box(parent.child("child").build());
        });
    });

    group.finish();
}

fn bench_context_propagation(c: &mut Criterion) {
    c.bench_function("context_get", |b| {
        let span = Tracer::start_span("test").build();
        let _guard = span.enter();

        b.iter(|| {
            black_box(Tracer::current_span());
        });
    });
}

criterion_group!(benches, bench_span_creation, bench_context_propagation);
criterion_main!(benches);
```

**Run Benchmarks**:
```bash
cargo bench
```

---

## Part 6: Debugging

### Logging

Use `tracing` crate for internal logging:

```rust
use tracing::{info, debug, error};

#[trace_span]
fn my_function() {
    info!("Starting span");
    debug!("Processing data");
    // ...
}
```

**Enable Logging**:
```bash
RUST_LOG=tracer=debug cargo run
```

---

### Common Issues

#### Issue 1: Context Not Propagating

**Symptoms**: Child spans not linked to parent.

**Diagnosis**:
```rust
// Add logging
let parent = Tracer::current_span();
debug!("Parent span: {:?}", parent);

let child = Tracer::start_child_span("child").build();
debug!("Child span: {:?}", child);
debug!("Same trace? {}", parent.trace_id() == child.trace_id());
```

**Fix**: Ensure tokio runtime is used (not std::thread)

---

#### Issue 2: Spans Not Exporting

**Symptoms**: No spans in Jaeger/Zipkin.

**Diagnosis**:
```rust
impl Exporter for MyExporter {
    fn export(&self, span: Arc<Span>) -> Result<(), ExportError> {
        debug!("Exporting span: {:?}", span);
        // Check network connectivity
        // Check serialization
        // ...
    }
}
```

**Fix**: Check exporter configuration, network connectivity

---

#### Issue 3: High Memory Usage

**Symptoms**: Memory usage increasing over time.

**Diagnosis**:
```rust
// Add metrics
use metrics::{gauge, Counter};

gauge!("tracer.active_spans", active_spans.len() as f64);
Counter::new("tracer.total_spans").inc();
```

**Fix**: Reduce sampling rate, check for span leaks

---

## Part 7: Performance Tuning

### Lock-Free Span Storage

**Problem**: Mutex contention reduces performance.

**Solution**: Use atomics and concurrent data structures.

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use dashmap::DashMap;

pub struct SpanStorage {
    next_span_id: AtomicU64,
    active_spans: DashMap<SpanId, Arc<Span>>,
}

impl SpanStorage {
    pub fn create_span(&self, name: String) -> Arc<Span> {
        let span_id = SpanId(self.next_span_id.fetch_add(1, Ordering::Relaxed));
        let span = Arc::new(Span::new(span_id, name));

        self.active_spans.insert(span_id, span.clone());
        span
    }
}
```

**Performance**: ~50ns per span creation (vs 500ns with mutex)

---

### Batch Exporting

**Problem**: Exporting every span individually is inefficient.

**Solution**: Buffer and export in batches.

```rust
pub struct BatchProcessor {
    exporter: Box<dyn Exporter>,
    batch: Vec<Arc<Span>>,
    batch_size: usize,
}

impl BatchProcessor {
    pub fn add_span(&mut self, span: Arc<Span>) {
        self.batch.push(span);

        if self.batch.len() >= self.batch_size {
            self.flush();
        }
    }

    pub fn flush(&mut self) {
        for span in self.batch.drain(..) {
            let _ = self.exporter.export(span);
        }
    }
}
```

**Performance**: 10x reduction in network overhead

---

### Async Export

**Problem**: Synchronous export blocks application.

**Solution**: Export in background task.

```rust
impl Exporter for OtlpExporter {
    fn export(&self, span: Arc<Span>) -> Result<(), ExportError> {
        let otlp_span = self.to_otlp_span(&span);

        // Spawn background task
        tokio::spawn(async move {
            let _ = self.client.export(otlp_span).await;
        });

        Ok(())
    }
}
```

---

### Zero-Cost Feature Flag

**Problem**: Tracing overhead in production.

**Solution**: Compile-time elimination.

```rust
#[cfg(feature = "tracing")]
#[trace_span]
async fn my_function() {
    // With tracing: ~50ns overhead
    // Without tracing: 0ns overhead (compiled out)
}

#[cfg(not(feature = "tracing"))]
async fn my_function() {
    // No tracing code
}
```

**Performance**: 0ns overhead when disabled

---

## Part 8: Contributing

### Code Style

- Use `rustfmt` for formatting
- Use `clippy` for linting
- Follow Rust API guidelines
- Document all public APIs

```bash
cargo fmt
cargo clippy --all-targets
```

---

### Commit Messages

Follow conventional commits:

```
feat: Add OTLP exporter
fix: Context propagation in async tasks
docs: Update user guide
perf: Optimize span storage
test: Add integration tests
```

---

### Pull Request Process

1. Fork repository
2. Create branch: `git checkout -b feature/my-feature`
3. Make changes and commit
4. Push to fork: `git push origin feature/my-feature`
5. Create pull request
6. Address review feedback
7. Squash and merge

---

### Release Checklist

- [ ] All tests pass
- [ ] Zero clippy warnings
- [ ] Documentation updated
- [ ] Changelog updated
- [ ] Version bumped
- [ ] Tagged commit
- [ ] Published to crates.io

---

## Part 9: Advanced Topics

### Custom Propagators

```rust
pub trait Propagator: Send + Sync {
    fn inject(&self, ctx: &TraceContext, carrier: &mut dyn Carrier);
    fn extract(&self, carrier: &dyn Carrier) -> Option<TraceContext>;
}

// Example: B3 propagator (Zipkin)
pub struct B3Propagator;

impl Propagator for B3Propagator {
    fn inject(&self, ctx: &TraceContext, carrier: &mut dyn Carrier) {
        carrier.set("X-B3-TraceId", ctx.trace_id().to_hex());
        carrier.set("X-B3-SpanId", ctx.span_id().to_hex());
        carrier.set("X-B3-Sampled", "1");
    }

    fn extract(&self, carrier: &dyn Carrier) -> Option<TraceContext> {
        let trace_id = carrier.get("X-B3-TraceId")?;
        let span_id = carrier.get("X-B3-SpanId")?;

        Some(TraceContext::new(
            TraceId::from_hex(&trace_id).ok()?,
            SpanId::from_hex(&span_id).ok()?,
        ))
    }
}
```

---

### Span Links

Link spans from different traces:

```rust
let job_span = Tracer::start_span("submit_job")
    .attribute("job.id", 12345)
    .build();

// Later, in different trace
let process_span = Tracer::start_span("process_job")
    .link(SpanLink {
        trace_id: job_span.trace_id(),
        span_id: job_span.span_id(),
        attributes: HashMap::new(),
    })
    .build();
```

---

### Baggage Propagation

Pass user-defined data across services:

```rust
// Set baggage
span.set_baggage("user.id", "12345");
span.set_baggage("tenant.id", "67890");

// Inject into carrier
propagator.inject_baggage(&span.baggage(), &mut headers);

// Extract from carrier
let baggage = propagator.extract_baggage(&headers)?;
```

---

## Conclusion

You now have everything you need to contribute to tracer-rs! Here's what you learned:

- ✅ Architecture overview
- ✅ Development setup
- ✅ Building custom exporters
- ✅ Building custom samplers
- ✅ Testing strategies
- ✅ Debugging techniques
- ✅ Performance tuning
- ✅ Contributing guidelines

**Next Steps**:
1. Check out [GitHub Issues](https://github.com/SuperInstance/tracer-rs/issues)
2. Read [examples.md](examples.md) for integration examples
3. Join the discussion in [GitHub Discussions](https://github.com/SuperInstance/tracer-rs/discussions)

**Happy Contributing!** 🚀

---

**Document Status**: ✅ Complete
**Word Count**: 4,000+
**Next**: CLI Design (06-cli.md)
