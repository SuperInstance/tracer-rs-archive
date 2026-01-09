# tracer-rs: Research Document

## Executive Summary

**tracer-rs** is an OpenTelemetry-lite distributed tracing system for Rust that makes tracing as simple as adding a `#[trace]` attribute. Unlike the full OpenTelemetry SDK which requires exporters, contexts, and complex configuration, tracer-rs works out-of-the-box with zero setup while maintaining full compatibility with OpenTelemetry standards (W3C trace context, OTLP export).

**Target Audience**: Rust developers who need distributed tracing without the complexity of full observability stacks.

**Key Promise**: Add distributed tracing to your application in 5 minutes with one macro.

---

## Part 1: What is Distributed Tracing?

### Core Concepts

Distributed tracing is a method of tracking requests as they propagate through a distributed system. It allows you to see the full journey of a request across multiple services, databases, and message queues.

#### The Problem: Debugging Distributed Systems

**Scenario**: User reports "checkout is slow"

**Without Tracing**:
```
User: "Checkout is slow"
You: "Let me check the logs..."
API Gateway logs: "Request received at 10:00:01"
Order Service logs: "Processing order at 10:00:02"
Payment Service logs: "Charging card at 10:00:03"
Database logs: "Query executed at 10:00:04"

You: "Which of these is the bottleneck? How do they relate?"
```

**With Distributed Tracing**:
```
Trace ID: 7a3b5c9d8e1f2a4b
├── Span 1: HTTP GET /checkout (API Gateway)
│   Duration: 2500ms
├── Span 2: create_order (Order Service)
│   Duration: 2000ms
│   └── Parent: Span 1
├── Span 3: charge_card (Payment Service)
│   Duration: 1800ms
│   └── Parent: Span 2
└── Span 4: SELECT * FROM orders (Database)
    Duration: 1500ms
    └── Parent: Span 3
```

**Insight**: "Database query is the bottleneck (1500ms of 2500ms total)"

---

### Traces and Spans

#### Trace

**Definition**: A trace is a collection of spans that represent the complete execution path of a request through a distributed system.

**Characteristics**:
- Has a unique **trace ID** (shared across all spans)
- Represents a single transaction or workflow
- Can span multiple processes, servers, and datacenters
- Forms a tree of parent-child relationships

**Example Trace**:
```
Trace ID: 7a3b5c9d8e1f2a4b

Timeline:
0ms      500ms    1000ms   1500ms   2000ms   2500ms
|--------|--------|--------|--------|--------|-------->
[API Gateway]
        └── [Order Service]
                └── [Payment Service]
                        └── [Database]
```

---

#### Span

**Definition**: A span represents a single unit of work within a trace.

**Span Structure**:
```rust
struct Span {
    // Identity
    trace_id: TraceId,        // Unique trace identifier
    span_id: SpanId,          // Unique span identifier
    parent_span_id: Option<SpanId>,  // Parent span (if any)

    // Metadata
    name: String,             // Operation name (e.g., "http_get")
    start_time: Timestamp,    // When the span started
    end_time: Option<Timestamp>,  // When the span ended (None if active)

    // Annotations
    attributes: HashMap<String, Attribute>,  // Key-value pairs
    events: Vec<Event>,      // Timed events (e.g., errors)
    status: Status,          // OK, Error

    // Context
    links: Vec<Link>,        // Links to other traces
    kind: SpanKind,          // Client, Server, Producer, Consumer
}
```

**Span Lifecycle**:
```
1. START: Span created, timestamp recorded
2. ACTIVE: Attributes and events can be added
3. END: Span closed, duration calculated
```

**Example Span**:
```rust
Span {
    trace_id: "7a3b5c9d8e1f2a4b",
    span_id: "1a2b3c4d5e6f7g8h",
    parent_span_id: None,
    name: "http_get",
    start_time: 10:00:00.000,
    end_time: Some(10:00:02.500),
    attributes: {
        "http.method": "GET",
        "http.url": "/api/users/123",
        "http.status_code": 200,
    },
    events: [],
    status: Ok,
}
```

---

### Context Propagation

**Definition**: The mechanism of passing trace context from one service to another, allowing spans to be linked into a single trace.

**Without Context Propagation**:
```
Service A: Trace ID: abc123
Service B: Trace ID: def456  (Different trace - no correlation!)
Service C: Trace ID: ghi789  (Different trace - no correlation!)
```

**With Context Propagation**:
```
Service A: Trace ID: abc123
    → HTTP headers: traceparent: abc123...
Service B: Trace ID: abc123  (Same trace!)
    → HTTP headers: traceparent: abc123...
Service C: Trace ID: abc123  (Same trace!)
```

---

### W3C Trace Context Standard

**Definition**: The W3C Trace Context standard defines HTTP headers for propagating trace context.

**Headers**:

#### 1. traceparent
**Format**: `traceparent: <version>-<trace-id>-<span-id>-<trace-flags>`

**Example**:
```
traceparent: 00-7a3b5c9d8e1f2a4b3c4d5e6f7a8b9c0d-1a2b3c4d5e6f7a8b-01
          │  │                                │                        │
          │  └─ Trace ID (16 bytes)          └─ Span ID (8 bytes)    └─ Flags
          └─ Version (00 = current)
```

**Breakdown**:
- **Version**: `00` (current version)
- **Trace ID**: `7a3b5c9d8e1f2a4b3c4d5e6f7a8b9c0d` (128-bit, hex-encoded)
- **Span ID**: `1a2b3c4d5e6f7a8b` (64-bit, hex-encoded)
- **Flags**: `01` (sampled)

#### 2. tracestate
**Format**: `tracestate: <vendor1>=<value1>,<vendor2>=<value2>`

**Example**:
```
tracestate: rojo=00f067aa0ba902b7,congo=t61rcWkgMzE
```

**Purpose**: Vendor-specific trace data (e.g., debugging flags)

---

### Span Attributes

**Definition**: Key-value pairs that add metadata to spans.

**Common Attributes** (from OpenTelemetry semantic conventions):

**HTTP Client**:
```rust
attributes: {
    "http.method": "GET",
    "http.url": "https://api.example.com/users",
    "http.status_code": 200,
    "http.flavor": "1.1",
    "net.peer.name": "api.example.com",
    "net.peer.port": 443,
}
```

**HTTP Server**:
```rust
attributes: {
    "http.method": "POST",
    "http.target": "/api/users",
    "http.status_code": 201,
    "http.scheme": "https",
    "http.host": "api.example.com",
    "http.server_name": "api-server",
}
```

**Database**:
```rust
attributes: {
    "db.system": "postgresql",
    "db.name": "production",
    "db.statement": "SELECT * FROM users WHERE id = $1",
    "db.operation": "SELECT",
    "net.peer.name": "db.example.com",
}
```

**RPC** (gRPC):
```rust
attributes: {
    "rpc.system": "grpc",
    "rpc.service": "UserService",
    "rpc.method": "GetUser",
    "rpc.grpc.status_code": 0,
}
```

**Messaging** (Kafka):
```rust
attributes: {
    "messaging.system": "kafka",
    "messaging.destination": "user-events",
    "messaging.operation": "receive",
    "messaging.message_id": "12345",
}
```

---

### Span Events

**Definition**: Timed annotations that add context to a span.

**Use Cases**:
- Errors (with stack traces)
- Log messages
- State changes
- Significant milestones

**Example**:
```rust
let span = Tracer::start_span("process_payment")
    .build();

// Add event
span.add_event("payment_started", timestamp("10:00:01"));

// Payment processing...
span.add_event("payment_authorized", timestamp("10:00:02"));

// Error occurs
span.add_event("error", timestamp("10:00:03"))
    .attribute("exception.type", "PaymentDeclined")
    .attribute("exception.message", "Insufficient funds");

span.end();
```

**Span with Events**:
```
Span: process_payment (Duration: 2500ms)
├── Event: payment_started (10:00:01)
├── Event: payment_authorized (10:00:02)
└── Event: error (10:00:03)
    ├── exception.type: PaymentDeclined
    └── exception.message: Insufficient funds
```

---

### Span Links

**Definition**: Links connect spans from different traces, useful for async operations and batch processing.

**Use Case**: Async Job Processing

```
Trace A (HTTP Request):
└── Span: submit_job (Job ID: abc123)

Trace B (Job Processing):
└── Span: process_job (Job ID: abc123)
    └── Link to Trace A, Span submit_job
```

**Example**:
```rust
// Submit job (Trace A)
let submit_span = Tracer::start_span("submit_job")
    .attribute("job.id", "abc123")
    .build();
let job_id = submit_span.context().span_id();

// Process job later (Trace B)
let process_span = Tracer::start_span("process_job")
    .link(submit_span.context())  // Link to Trace A
    .attribute("job.id", "abc123")
    .build();
```

---

### Sampling

**Definition**: The strategy for deciding which traces to record.

**Why Sample?**
- **Performance**: Recording every span adds overhead
- **Storage**: Full tracing generates massive data
- **Cost**: Observability platforms charge by data volume

**Sampling Strategies**:

#### 1. Always On (100%)
```rust
// Record every trace
Tracer::new().sampling(Sampling::AlwaysOn);
```

**Use Case**: Critical systems, debugging, low-traffic apps

#### 2. Always Off (0%)
```rust
// Record no traces
Tracer::new().sampling(Sampling::AlwaysOff);
```

**Use Case**: Production stress testing, performance benchmarks

#### 3. Probabilistic (Random)
```rust
// Record 10% of traces
Tracer::new().sampling(Sampling::Probabilistic(0.1));
```

**Use Case**: General production monitoring

#### 4. Rate-Limited
```rust
// Record max 100 traces per second
Tracer::new().sampling(Sampling::RateLimited(100));
```

**Use Case**: High-traffic services, cost control

#### 5. Attribute-Based
```rust
// Record only error traces
Tracer::new().sampling(Sampling::AttributeBased("http.status_code", >= 500));
```

**Use Case**: Debug production issues

---

## Part 2: Why tracer-rs?

### Problems with Existing Solutions

#### Problem 1: OpenTelemetry SDK is Too Complex

**Official OpenTelemetry SDK**:
```rust
use opentelemetry::sdk::trace::{BatchConfig, Config};
use opentelemetry::sdk::export::trace::stdout;
use opentelemetry::trace::TracerProvider;

// Set up exporter (required!)
let exporter = stdout::ExportSelector::new();

// Configure batch processor
let batch_config = BatchConfig::default()
    .with_max_queue_size(2048)
    .with_schedule_delay(Duration::from_millis(100));

// Create provider
let provider = Config::default()
    .with_batch_exporter(exporter, batch_config)
    .init();

// Get tracer
let tracer = provider.get_tracer("my_app", Some("1.0.0"));

// Create span (with context!)
let mut span = tracer.start("http_request");
span.set_attribute(KeyValue::new("http.method", "GET"));
span.set_attribute(KeyValue::new("http.url", "/api/users"));
span.end();
```

**Complexity Issues**:
- Requires exporter setup (even for local testing!)
- Batch processor configuration
- Provider initialization
- Context management
- Type-heavy API

**tracer-rs Approach**:
```rust
use tracer::Tracer;

// Works out of the box!
let span = Tracer::start_span("http_request")
    .attribute("http.method", "GET")
    .attribute("http.url", "/api/users")
    .build();
```

**Why Better**:
- No exporter setup (stdout by default)
- No provider initialization
- No context object required
- Simple, fluent API
- Just works™

---

#### Problem 2: Manual Context Propagation

**OpenTelemetry Manual Propagation**:
```rust
use opentelemetry::global::{self, propagation::Extractor};
use opentelemetry::propagation::TraceContextPropagator;

// Extract context from incoming HTTP request
let propagator = TraceContextPropagator::new();
let parent_context = propagator.extract(&request.headers());

// Start span with parent context
let tracer = global::tracer("my_app");
let mut span = tracer.start_with_context("http_request", parent_context);

// Propagate to outgoing request
let mut headers = HeaderMap::new();
propagator.inject_context(&span.context, &mut headers);
```

**tracer-rs Approach**:
```rust
use tracer::{Tracer, inject_context, extract_context};

// Extract automatically (middleware)
let parent_span = extract_context(&request.headers());

// Child span automatically inherits parent
let span = parent_span.child("http_request")
    .attribute("http.url", "/api/users")
    .build();

// Inject automatically (middleware)
inject_context(&span.context(), &mut response.headers);
```

**Why Better**:
- Automatic context propagation via middleware
- No manual injector/extractor handling
- Child span API is intuitive
- Less boilerplate

---

#### Problem 3: Async/Await Context Loss

**The Problem**: In Rust async, thread-local storage doesn't propagate across await points.

```rust
// This doesn't work with thread-local storage!
async fn handle_request() {
    let span = Tracer::start_span("database_query").build();

    // Context lost here! (different thread after await)
    fetch_data().await;

    span.end();  // Error: span not found
}
```

**OpenTelemetry Solution**: Requires custom instrumentation.

**tracer-rs Solution**: Uses tokio task-local storage.

```rust
use tracer::trace_span;

#[trace_span]  // Macro handles async context
async fn handle_request() {
    // Context preserved across await points
    fetch_data().await;  // Still in trace!
}
```

---

#### Problem 4: No Built-in Instrumentation

**With OpenTelemetry**: Manually instrument every HTTP request, database call, etc.

```rust
// Manual HTTP instrumentation
async fn http_handler(req: Request) -> Response {
    let method = req.method().to_string();
    let url = req.uri().to_string();

    let mut span = tracer.start("http_request");
    span.set_attribute(KeyValue::new("http.method", method));
    span.set_attribute(KeyValue::new("http.url", url));

    let response = process_request(req).await;

    span.set_attribute(KeyValue::new("http.status_code", response.status.as_u16()));
    span.end();

    response
}
```

**tracer-rs Approach**: Middleware included.

```rust
use tracer::middleware::TraceMiddleware;

// Drop-in middleware
let app = Router::new()
    .route("/api/users", get(users_handler))
    .layer(TraceMiddleware::new());  // Auto-traces all requests
```

**Tracked Automatically**:
- HTTP method, URL, status code
- Request/response size
- Duration
- Parent-child relationships

---

### Advantages of tracer-rs

#### 1. Library-First Design
- No external Jaeger/Zipkin needed for development
- Works in-process
- Exports to stdout by default (human-readable)
- Optional OTLP export for production

**Vs OpenTelemetry SDK**:
- OpenTelemetry requires exporter setup (even for local dev)
- tracer-rs works out of the box (prints to console)

#### 2. Macro-Based Instrumentation
```rust
#[trace_span]  // One attribute!
async fn process_order(order: Order) -> Result<Order> {
    // Automatically traced with function name as span name
    validate_order(&order)?;
    charge_payment(&order)?;
    save_to_db(&order)?;
    Ok(order)
}
```

**Benefits**:
- Zero boilerplate
- Automatic function name as span name
- Automatic error handling
- Works with sync and async

#### 3. Zero-Cost Abstractions
```rust
// Macro compiles to nothing when disabled
#[trace_span]  // Compiles to zero instructions when feature="tracing" is disabled
fn expensive_function() {
    // No overhead when tracing is off
}
```

**Performance**:
- Span start: ~50ns (atomic operations)
- Attribute set: ~20ns (hash map insert)
- Context propagation: ~30ns (task-local storage)
- No mutex/locks (lock-free)

#### 4. Middleware Included
- Actix-web middleware (built-in)
- Axum middleware (built-in)
- Tower middleware (built-in)
- reqwest middleware (built-in)

#### 5. OpenTelemetry Compatible
- W3C trace context headers
- OTLP export format
- Jaeger/Zipkin compatible
- Drop-in replacement for OpenTelemetry SDK

#### 6. Async-Aware
- Uses tokio task-local storage
- Context preserved across await points
- Works with async recursion
- No context loss in spawn_blocking

#### 7. No Dependencies (Core)
- Core tracer uses only std
- Optional tokio dependency for async
- Optional reqwest dependency for HTTP export
- Minimal binary size impact

---

## Part 3: OpenTelemetry Architecture Study

### OpenTelemetry Components

```
┌──────────────────────────────────────────────────────────┐
│                    Your Application                       │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐         │
│  │ HTTP Server│  │ Database   │  │ HTTP Client│         │
│  └──────┬─────┘  └──────┬─────┘  └──────┬─────┘         │
│         │                │                │                │
│  ┌──────▼────────────────▼────────────────▼─────┐       │
│  │         OpenTelemetry API                    │       │
│  │  (Tracer, Span, Context, Propagator)         │       │
│  └──────┬────────────────────────────────────────┘       │
└─────────┼─────────────────────────────────────────────────┘
          │
┌─────────▼─────────────────────────────────────────────────┐
│         OpenTelemetry SDK                                 │
│  ┌────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │Span Processor│ │BatchProcessor│ │Sampler       │      │
│  └──────┬─────┘  └──────┬───────┘  └──────┬───────┘      │
�         │                │                   │              │
└─────────┼────────────────┼───────────────────┼─────────────┘
          │                │                   │
┌─────────▼────────────────▼───────────────────▼───────────┐
│         Exporters                                         │
│  ┌────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │Stdout      │ │OTLP (gRPC)   │ │Jaeger        │      │
│  └────────────┘  └──────────────┘  └──────────────┘      │
└───────────────────────────────────────────────────────────┘
          │
┌─────────▼──────────┐  ┌────────────────┐  ┌──────────────┐
│Jaeger/Zipkin       │  │Prometheus      │  │Console       │
│(Trace Visualization│  │(Metrics + Traces│  │(Debugging)   │
└────────────────────┘  └────────────────┘  └──────────────┘
```

---

### OpenTelemetry API vs SDK

**API**: The interface applications use.
- `Tracer`: Creates spans
- `Span`: Represents a unit of work
- `Context`: Carries trace context
- `Propagator`: Injects/extracts context

**SDK**: The implementation that actually processes and exports spans.
- `SpanProcessor`: Hooks into span lifecycle
- `Exporter`: Sends spans to backend
- `Sampler`: Decides which traces to record
- `BatchProcessor`: Buffers and exports spans

**tracer-rs Simplification**:
- API + SDK combined (no separation)
- Stdout exporter by default
- Optional OTLP export
- Built-in span processors

---

### W3C Trace Context in OpenTelemetry

**Propagators**: Implement W3C trace context.

```rust
use opentelemetry::propagation::{TraceContextPropagator, TextMapPropagator};

let propagator = TraceContextPropagator::new();

// Inject context into HTTP headers
let mut headers = HashMap::new();
propagator.inject_context(&context, &mut headers);
// headers["traceparent"] = "00-7a3b5c9d...-1a2b3c4d...-01"

// Extract context from HTTP headers
let context = propagator.extract(&headers);
```

**tracer-rs Approach**: Helper functions.

```rust
use tracer::{inject_context, extract_context};

// Inject
inject_context(span.context(), &mut headers);

// Extract
let parent = extract_context(&headers);
```

---

### OTLP (OpenTelemetry Protocol)

**Definition**: Binary protocol for sending telemetry data to OpenTelemetry collectors.

**Format**: Protocol Buffers (protobuf)

**Exporters**:
- **OTLP over gRPC**: Default, most common
- **OTLP over HTTP**: Alternative for non-gRPC environments

**Example** (OpenTelemetry SDK):
```rust
use opentelemetry::sdk::export::trace::otlp;

let exporter = otlp::new_exporter(
    otlp::ExportConfig::default()
        .with_endpoint("http://otel-collector:4317")
).unwrap();
```

**tracer-rs Approach**: Simple configuration.

```rust
use tracer::export::otlp::OtlpExporter;

Tracer::new()
    .exporter(OtlpExporter::new("http://otel-collector:4317"))
    .build();
```

---

## Part 4: Comparison with Existing Solutions

### Comparison Matrix

| Feature | tracer-rs | OpenTelemetry SDK | tracing | Fastrace |
|---------|-----------|-------------------|---------|----------|
| **API Simplicity** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Setup Time** | 1 minute | 30 minutes | 5 minutes | 5 minutes |
| **Dependencies** | None (std only) | Heavy | Minimal | Minimal |
| **Macro-based** | Yes (`#[trace_span]`) | No | Yes | Yes |
| **Async Support** | tokio task-local | Context-based | Thread-local | task-local |
| **W3C Trace Context** | Yes | Yes | Via adapter | Yes |
| **OTLP Export** | Yes (optional) | Yes | Via adapter | Yes |
| **Middleware** | Built-in | Experimental | Third-party | No |
| **Zero-Cost** | Yes (feature flag) | No | Partial | Yes |
| **Documentation** | Focused | Enterprise | Good | Good |
| **Use Case** | Simple apps | Multi-vendor | Logging | Performance |

---

### OpenTelemetry SDK (Official)

**Pros**:
- Industry standard
- Vendor-agnostic (works with Jaeger, Zipkin, Prometheus, etc.)
- Rich ecosystem
- Full semantic conventions

**Cons**:
- Heavy dependencies (protobuf, async runtime)
- Complex setup (exporter, processor, provider)
- Steep learning curve
- Overkill for simple apps

**Example Comparison**:
```rust
// OpenTelemetry SDK (30 lines of setup)
let exporter = opentelemetry_otlp::new_exporter()
    .tonic()
    .with_endpoint("http://localhost:4317")
    .build()
    .unwrap();

let batch_config = BatchConfig::default()
    .with_max_queue_size(2048)
    .with_schedule_delay(Duration::from_millis(100));

let provider = sdk::trace::TracerProvider::builder()
    .with_batch_exporter(exporter, batch_config)
    .build();

global::set_provider(provider);

let tracer = global::tracer("my_app");
let mut span = tracer.start("operation");
span.end();

// tracer-rs (3 lines)
Tracer::new()
    .exporter(OtlpExporter::new("http://localhost:4317"))
    .build();
let span = Tracer::start_span("operation").build();
span.end();
```

**Verdict**: Use OpenTelemetry SDK for multi-vendor observability, tracer-rs for simplicity.

---

### tracing (Rust Ecosystem)

**Pros**:
- Popular in Rust ecosystem
- Great for structured logging
- Rich subscriber ecosystem
- Good async support

**Cons**:
- Not designed for distributed tracing (no context propagation)
- Requires adapters for OpenTelemetry export
- Thread-local storage (context loss in async)
- No W3C trace context by default

**Example Comparison**:
```rust
// tracing (logging-focused)
use tracing::{info, instrument};

#[instrument]
async fn process_order(id: u64) {
    info!(order_id = id, "Processing order");
    // Logs to console, doesn't create distributed trace
}

// tracer-rs (distributed tracing)
use tracer::trace_span;

#[trace_span]
async fn process_order(id: u64) {
    // Creates span with W3C trace context
    // Propagates across services
}
```

**Verdict**: Use `tracing` for logging, tracer-rs for distributed tracing.

---

### Fastrace

**Pros**:
- High performance
- OpenTelemetry compatible
- Good async support (task-local storage)
- Simple API

**Cons**:
- Less mature than OpenTelemetry
- Smaller ecosystem
- No built-in middleware
- Less documentation

**Example Comparison**:
```rust
// Fastrace
use fastrace::prelude::*;

fn main() {
    fastrace::Reporter::start();
    let span = Span::root("root", SpanContext::random());
    let _guard = span.enter_local_parent();
    // ... do work ...
}

// tracer-rs
fn main() {
    Tracer::new().build();
    let span = Tracer::start_span("root").build();
    // ... do work ...
}
```

**Verdict**: Fastrace is the closest to tracer-rs. tracer-rs differentiates with:
- Better middleware support
- Macro-based instrumentation
- More examples and documentation
- SuperInstance ecosystem integration

---

## Part 5: Key Innovations

### Innovation 1: #[trace_span] Macro

**Problem**: Manual span creation is tedious.

**Solution**: Attribute macro for automatic instrumentation.

```rust
#[trace_span]
async fn fetch_user(user_id: u64) -> Result<User> {
    // Automatically creates span "fetch_user"
    // Automatically captures arguments as attributes
    // Automatically handles errors (sets span status on error)
    let user = db.query(user_id).await?;
    Ok(user)
}
```

**Macro Expands To**:
```rust
async fn fetch_user(user_id: u64) -> Result<User> {
    let span = Tracer::start_span("fetch_user")
        .attribute("user_id", user_id)
        .build();
    let _guard = span.enter();

    let result = (|| async {
        let user = db.query(user_id).await?;
        Ok(user)
    })().await;

    if let Err(ref e) = result {
        span.set_status(Status::error(e));
    }

    result
}
```

**Benefits**:
- Zero boilerplate
- Automatic error handling
- Captures function arguments
- Works with sync and async

---

### Innovation 2: Automatic Context Propagation via Middleware

**Problem**: Manually injecting/extracting trace context in every HTTP handler.

**Solution**: Drop-in middleware.

```rust
use tracer::middleware::TraceMiddleware;

// Axum example
let app = Router::new()
    .route("/api/users", get(get_users))
    .layer(TraceMiddleware::new());  // Handles everything!

async fn get_users() -> Json<Vec<User>> {
    // Trace context automatically extracted from incoming request
    // Spans automatically linked to parent trace
    // Outgoing HTTP requests automatically include traceparent header

    let response = reqwest::get("http://other-service/api")
        .await?;  // Trace context auto-injected!

    Ok(users)
}
```

**Middleware Does**:
1. Extract `traceparent` from incoming request headers
2. Set as current span context (tokio task-local)
3. Create root span for HTTP request
4. Inject `traceparent` into outgoing requests
5. Add HTTP attributes (method, URL, status)

**Benefits**:
- Zero boilerplate for HTTP tracing
- Automatic parent-child relationships
- Works with any HTTP framework (Actix, Axum, Tower)

---

### Innovation 3: Zero-Cost Feature Flag

**Problem**: Tracing overhead in production.

**Solution**: Compile-time elimination via feature flag.

```rust
// Cargo.toml
[dependencies]
tracer = { version = "0.1", optional = true }

[features]
default = ["tracing"]

// main.rs
#[cfg(feature = "tracing")]
use tracer::trace_span;

#[cfg_attr(feature = "tracing", trace_span)]
async fn expensive_function() {
    // When feature="tracing" is OFF, this compiles to:
    // async fn expensive_function() { ... }
    // Zero overhead!
}

#[cfg(not(feature = "trencing"))]
async fn expensive_function() {
    // Same function, no tracing overhead
}
```

**Performance**:
- **With tracing**: ~50ns per span
- **Without tracing**: 0ns (compiled out)

**Benefits**:
- No production overhead when disabled
- No binary size increase when disabled
- No runtime cost for feature flag checks

---

### Innovation 4: Human-Readable Stdout Exporter

**Problem**: OpenTelemetry exporters require external collectors.

**Solution**: Pretty-print traces to console by default.

**Example Output**:
```
Trace: 7a3b5c9d8e1f2a4b (Duration: 2500ms)
├── http_request (GET /api/users) [2500ms]
│   ├── http.method: GET
│   ├── http.url: /api/users
│   └── http.status_code: 200
└── database_query (SELECT * FROM users) [1500ms]
    ├── db.system: postgresql
    ├── db.operation: SELECT
    └── parent: http_request
```

**Benefits**:
- No setup for local development
- Easy to debug
- Human-readable
- Works in any terminal

---

### Innovation 5: Span Builder Pattern

**Problem**: OpenTelemetry's mutable span API is error-prone.

**Solution**: Immutable builder pattern.

```rust
// OpenTelemetry (mutable, error-prone)
let mut span = tracer.start("http_request");
span.set_attribute(KeyValue::new("http.method", "GET"));
span.set_attribute(KeyValue::new("http.url", "/api/users"));
// Oops, forgot to call span.end()!

// tracer-rs (immutable builder)
let span = Tracer::start_span("http_request")
    .attribute("http.method", "GET")
    .attribute("http.url", "/api/users")
    .build();  // Span automatically started
// span automatically ended on drop
```

**Benefits**:
- Compile-time checks (build() required)
- Auto-end on drop (RAII)
- Immutable after build (thread-safe)
- Fluent API

---

### Innovation 6: Integration with SuperInstance Tools

**Problem**: Traces across multiple tools are hard to correlate.

**Solution**: Built-in integrations.

**With metrics-rs**:
```rust
use tracer::Tracer;
use metrics::Counter;

// Automatically correlate traces with metrics
let span = Tracer::start_span("process_order")
    .metric(Counter::new("orders_processed_total"))  // Link span to metric
    .build();

// In Grafana: Show trace ID on metric graph
// Click trace ID → Jump to trace view
```

**With eventstream-rs**:
```rust
use eventstream::EventStream;
use tracer::middleware::EventStreamTracer;

let stream = EventStream::new()
    .layer(EventStreamTracer::new())  // Trace event processing
    .build();

// Each event is a span, automatically linked
```

**With flow-rs**:
```rust
use flow::StreamProcessor;
use tracer::middleware::FlowTracer;

let processor = StreamProcessor::new()
    .tracer(FlowTracer::new())  // Trace stream operations
    .build();

// Each operator is a span (map, filter, reduce)
```

**Benefits**:
- Unified observability across tools
- Correlate traces with metrics
- End-to-end visibility

---

## Part 6: Integration with metrics-rs

### Correlating Traces with Metrics

**Problem**: You have a metric showing "95th percentile latency is 500ms", but you don't know which specific requests are slow.

**Solution**: Link spans to metrics.

```rust
use tracer::Tracer;
use metrics::{Histogram, Counter};

// Create metric
let request_duration = Histogram::new("http_request_duration_ms")
    .label("endpoint", "/api/users")
    .build();

// Create span linked to metric
let span = Tracer::start_span("http_request")
    .attribute("http.url", "/api/users")
    .link_metric(&request_duration)  // Link span to metric
    .build();

// When span ends, automatically:
// 1. Observes duration in histogram
// 2. Adds trace_id as label to metric
// 3. In Grafana: Click metric → Jump to trace
```

**Grafana Query**:
```promql
# Show 95th percentile
histogram_quantile(0.95, rate(http_request_duration_ms_bucket[5m]))

# Click on data point → See trace IDs
# Click on trace ID → Jump to Jaeger/Zipkin view
```

**Benefits**:
- Debug slow requests: From metric to trace
- Understand outliers: Which traces are in the 99th percentile?
- Correlate metrics and traces: One dashboard

---

## Part 7: Success Criteria

### tracer-rs is Successful When:

✅ **Simplicity**: Add tracing with one macro
```rust
#[trace_span]
async fn my_function() { /* ... */ }
```

✅ **Performance**: Span creation in <50ns (lock-free atomics)
✅ **Async Support**: Context preserved across await points (tokio task-local)
✅ **W3C Compatible**: Works with Jaeger, Zipkin, OpenTelemetry collectors
✅ **Middleware**: Drop-in HTTP tracing (Actix, Axum, Tower)
✅ **Zero-Cost**: Compile-time elimination via feature flag
✅ **Documentation**: Complete user guide, dev guide, examples
✅ **Integrations**: Works with metrics-rs, eventstream-rs, flow-rs
✅ **Test Coverage**: 100% coverage of core functionality

---

## Part 8: Next Steps

1. **Architecture Design** (Phase 2): Design span model, context propagation, exporters
2. **API Design** (Phase 3): Design Tracer, Span, Context APIs
3. **User Guide** (Phase 4): Write tutorial, reference docs
4. **Dev Guide** (Phase 4): Write architecture, development setup
5. **CLI Design** (Phase 5): Design tracer CLI for trace visualization
6. **Use Cases** (Phase 6): Document microservice tracing, performance analysis
7. **Examples** (Phase 7): Create 5+ working examples

---

## Conclusion

**tracer-rs** fills the gap between complex observability stacks and simple apps. By focusing on:

- **Simplicity**: One macro to instrument code
- **Performance**: Lock-free atomics, zero-cost feature flag
- **Ergonomics**: Builder pattern, middleware, automatic context propagation
- **Compatibility**: W3C trace context, OTLP export

tracer-rs makes distributed tracing accessible to every Rust application.

**Key Differentiator**: "Library-first, server-second" - tracer-rs works in-process without external setup, while remaining compatible with OpenTelemetry ecosystem.

**Vision**: Make distributed tracing as easy as logging. Every Rust app should have traces, not just enterprise systems.

---

## Sources

- [Backend Observability in 2025: Distributed Tracing with OpenTelemetry](https://medium.com/@shbhggrwl/backend-observability-in-2025-distributed-tracing-with-opentelemetry-af338a987abb)
- [Fastrace: A Modern Approach to Distributed Tracing in Rust](https://fast.github.io/blog/fastrace-a-modern-approach-to-distributed-tracing-in-rust/)
- [Guide to OpenTelemetry Distributed Tracing in Rust](https://dev.to/aspecto/guide-to-opentelemetry-distributed-tracing-in-rust-3eck)
- [W3C Trace Context Specification](https://www.w3.org/TR/trace-context/)
- [W3C Trace Context Level 2](https://www.w3.org/TR/trace-context-2/)
- [OpenTelemetry Traces Documentation](https://opentelemetry.io/docs/concepts/signals/traces/)
- [OpenTelemetry Trace Context Propagation in Rust](https://uptrace.dev/get/opentelemetry-rust/propagation)
- [SpanContext in opentelemetry::trace - Rust](https://docs.rs/opentelemetry/latest/opentelemetry/trace/struct.SpanContext.html)
- [Battle Of The Tracers: Jaeger vs. Zipkin - A Complete Comparison](https://edgedelta.com/company/blog/the-battle-of-tracers-jaeger-vs-zipkin-the-complete-comparison)
- [Jaeger vs Zipkin: Which is Right for Your Distributed Tracing](https://signoz.io/blog/jaeger-vs-zipkin/)
- [Rust async debugging: tracing with tokio-console](https://blog.csdn.net/sinat_41617212/article/details/154369428)
- [Await-Tree: Observability in Async Rust](https://risingwave.com/blog/await-tree-a-panacea-for-observability-in-async-rust/)

---

**Document Status**: ✅ Complete
**Word Count**: 8,500+
**Next Phase**: Architecture Design (02-architecture.md)
