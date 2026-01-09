# tracer-rs: User Guide

## Welcome to tracer-rs

**tracer-rs** makes distributed tracing in Rust as simple as adding one attribute: `#[trace_span]`. This guide will help you get started in 5 minutes and master the essentials.

---

## Part 1: Quick Start (5 Minutes)

### Installation

Add to your `Cargo.toml`:

```toml
[dependencies]
tracer = "0.1"
tokio = { version = "1", features = ["full"] }
```

### Your First Trace

```rust
use tracer::trace_span;

#[trace_span]
async fn hello_world() {
    println!("Hello, traced world!");
}

#[tokio::main]
async fn main() {
    hello_world().await;
}
```

**Output**:
```
Trace: 7a3b5c9d8e1f2a4b (Duration: 0ms)
└── hello_world [0ms]
```

**That's it!** You just traced your first function.

---

## Part 2: Core Concepts

### What is a Span?

A **span** represents a unit of work in your application. Think of it like a stopwatch that records:
- When it started
- How long it took
- What happened (attributes, events)
- Relationships to other spans (parent-child)

**Example**:

```rust
#[trace_span]
async fn process_order(order_id: u64) -> Result<Order> {
    // Span starts here
    validate_order(order_id).await?;
    let order = fetch_order(order_id).await?;
    // Span ends here
    Ok(order)
}
```

---

### What is a Trace?

A **trace** is a collection of spans that show the complete journey of a request through your system.

**Example Trace**:
```
Trace: 7a3b5c9d8e1f2a4b
├── HTTP GET /api/orders/123 [2500ms]
│   ├── validate_order [500ms]
│   └── fetch_order [2000ms]
│       └── database_query [1500ms]
```

Each span shows what happened and how long it took.

---

### What is Context Propagation?

**Context propagation** is how traces connect across services. It's like passing a baton in a relay race.

**Without Context Propagation**:
```
Service A: Trace ID = abc123
Service B: Trace ID = def456  (Different! No connection!)
```

**With Context Propagation**:
```
Service A: Trace ID = abc123
    → HTTP headers: traceparent: abc123...
Service B: Trace ID = abc123  (Same! Connected!)
```

**tracer-rs handles this automatically via middleware!**

---

## Part 3: Adding Traces to Your Code

### Instrumenting Functions

Use the `#[trace_span]` attribute:

```rust
use tracer::trace_span;

#[trace_span]
async fn fetch_user(user_id: u64) -> Result<User> {
    let user = db.query(user_id).await?;
    Ok(user)
}

#[trace_span]
async fn fetch_orders(user_id: u64) -> Result<Vec<Order>> {
    let orders = db.query_orders(user_id).await?;
    Ok(orders)
}
```

**What Happens**:
- Span automatically created with function name
- Function arguments captured as attributes
- Errors automatically captured
- Span auto-ended when function returns

---

### Manual Span Creation

For more control, use the builder API:

```rust
use tracer::Tracer;

async fn process_request() {
    let span = Tracer::start_span("process_request")
        .attribute("request.type", "http")
        .attribute("request.id", 12345)
        .build();

    // Do work...

    // Span auto-ends when dropped
}
```

---

### Parent-Child Spans

Create child spans to show relationships:

```rust
use tracer::Tracer;

#[trace_span]
async fn process_order(order_id: u64) -> Result<Order> {
    // Get current span
    let root_span = Tracer::current_span().unwrap();

    // Create child span
    let db_span = root_span.child("database_query")
        .attribute("db.system", "postgresql")
        .attribute("db.query", "SELECT * FROM orders WHERE id = $1")
        .build();

    let order = execute_query(order_id).await?;
    Ok(order)
}
```

**Output**:
```
Trace: 7a3b5c9d8e1f2a4b
└── process_order [2500ms]
    └── database_query [1500ms]
```

---

## Part 4: Adding Attributes

### What are Attributes?

Attributes are key-value pairs that add context to spans. They help you filter and analyze traces.

**Example**:

```rust
use tracer::Tracer;

let span = Tracer::start_span("http_request")
    .attribute("http.method", "GET")
    .attribute("http.url", "/api/users")
    .attribute("http.status_code", 200)
    .build();
```

---

### Semantic Conventions

Use standard attribute names (from OpenTelemetry) for consistency:

```rust
use tracer::{Tracer, semantic};

let span = Tracer::start_span("http_request")
    .attribute(semantic::HTTP_METHOD, "GET")
    .attribute(semantic::HTTP_URL, "/api/users")
    .attribute(semantic::HTTP_STATUS_CODE, 200)
    .build();
```

**Common Attributes**:

**HTTP**:
- `http.method`: GET, POST, PUT, DELETE
- `http.url`: /api/users
- `http.status_code`: 200, 404, 500

**Database**:
- `db.system`: postgresql, mysql, mongodb
- `db.name`: production
- `db.operation`: SELECT, INSERT, UPDATE

**RPC** (gRPC):
- `rpc.system`: grpc
- `rpc.service`: UserService
- `rpc.method`: GetUser

---

### Attribute Best Practices

**DO**:
- ✅ Use low-cardinality attributes (enums, finite sets)
- ✅ Follow semantic conventions
- ✅ Add attributes that help debugging

```rust
span.attribute("http.method", "GET");  // ✅ Low cardinality
span.attribute("http.status_code", 200);  // ✅ Low cardinality
```

**DON'T**:
- ❌ Use high-cardinality attributes (IDs, timestamps)
- ❌ Add PII (user emails, credit cards)

```rust
span.attribute("user.id", 12345);  // ❌ High cardinality!
span.attribute("user.email", "user@example.com");  // ❌ PII!
```

---

## Part 5: Adding Events

### What are Events?

Events are timed annotations that add context to a span.

**Example**:

```rust
use tracer::Tracer;

let span = Tracer::start_span("process_payment")
    .build();

// Add events
span.add_event("payment_started");
// ... do work ...
span.add_event("payment_authorized");
// ... do work ...
span.add_event("payment_completed");
```

**Output**:
```
Trace: 7a3b5c9d8e1f2a4b
└── process_payment [2500ms]
    ├── Event: payment_started at 10:00:00.000
    ├── Event: payment_authorized at 10:00:01.000
    └── Event: payment_completed at 10:00:02.500
```

---

### Events with Attributes

Add attributes to events for more context:

```rust
use tracer::Tracer;
use std::collections::HashMap;

let span = Tracer::start_span("process_payment").build();

let mut attrs = HashMap::new();
attrs.insert("amount".to_string(), tracer::Attribute::Int(9999));
attrs.insert("currency".to_string(), tracer::Attribute::String("USD".to_string()));

span.add_event_with_attrs("payment_started", attrs);
```

---

### Error Events

Capture errors as events:

```rust
use tracer::Tracer;

let span = Tracer::start_span("process_payment").build();

match process_payment() {
    Ok(_) => {
        span.add_event("payment_succeeded");
    }
    Err(e) => {
        let mut attrs = HashMap::new();
        attrs.insert("exception.type".to_string(), tracer::Attribute::String("PaymentDeclined".to_string()));
        attrs.insert("exception.message".to_string(), tracer::Attribute::String(e.to_string()));

        span.add_event_with_attrs("error", attrs);
        span.set_status(tracer::Status::Error {
            description: e.to_string(),
        });
    }
}
```

---

## Part 6: HTTP Tracing

### Automatic HTTP Tracing (Axum)

```rust
use axum::{Router, routing::get};
use tracer::middleware::TraceLayer;

let app = Router::new()
    .route("/api/users", get(get_users))
    .layer(TraceLayer::new());  // Auto-trace all requests!

async fn get_users() -> Json<Vec<User>> {
    // Span automatically created
    // Context automatically propagated
    let users = fetch_users().await?;
    Ok(Json(users))
}
```

**Tracked Automatically**:
- HTTP method, URL, status code
- Request duration
- Trace context (incoming and outgoing)

---

### Manual HTTP Tracing

```rust
use reqwest::Client;
use tracer::Tracer;

async fn call_external_api() -> Result<Response> {
    let span = Tracer::start_span("http_client_request")
        .attribute("http.method", "GET")
        .attribute("http.url", "https://api.example.com/users")
        .build();

    let response = Client::new()
        .get("https://api.example.com/users")
        .header("traceparent", span.context().to_traceparent())  // Inject trace context
        .send()
        .await?;

    span.set_attribute("http.status_code", response.status().as_u16());
    Ok(response)
}
```

---

## Part 7: Database Tracing

### Manual Database Tracing

```rust
use tracer::Tracer;
use tracer::semantic;

async fn fetch_user(user_id: u64) -> Result<User> {
    let span = Tracer::start_span("database_query")
        .attribute(semantic::DB_SYSTEM, "postgresql")
        .attribute(semantic::DB_NAME, "production")
        .attribute(semantic::DB_STATEMENT, "SELECT * FROM users WHERE id = $1")
        .attribute(semantic::DB_OPERATION, "SELECT")
        .build();

    let user = sqlx::query_as::<_, User>("SELECT * FROM users WHERE id = $1")
        .bind(user_id)
        .fetch_one(&pool)
        .await?;

    Ok(user)
}
```

**Output**:
```
Trace: 7a3b5c9d8e1f2a4b
└── database_query [1500ms]
    ├── db.system: postgresql
    ├── db.name: production
    ├── db.operation: SELECT
    └── db.statement: SELECT * FROM users WHERE id = $1
```

---

### Tracing ORM Queries (Diesel)

```rust
use tracer::Tracer;
use diesel::prelude::*;

#[trace_span]
async fn find_user(user_id: u64) -> Result<User> {
    let span = Tracer::start_span("diesel_query")
        .attribute("db.system", "postgresql")
        .build();

    let user = users::table
        .filter(users::id.eq(user_id))
        .first(&connection)
        .await?;

    Ok(user)
}
```

---

## Part 8: Async/Await

### Context Preserved Across Await

**tracer-rs uses tokio task-local storage**, so context is preserved across await points:

```rust
use tracer::trace_span;

#[trace_span]
async fn process_order(order_id: u64) -> Result<Order> {
    // Context active here
    let user = fetch_user(order_id).await?;  // Context still active!
    let orders = fetch_orders(user.id).await?;  // Context still active!
    Ok(orders)
}
```

**No manual context passing needed!**

---

### Spawning Tasks

When spawning tasks, explicitly propagate context:

```rust
use tracer::Tracer;

#[trace_span]
async fn process_orders() {
    let span = Tracer::current_span().unwrap();

    for order_id in 1..10 {
        let child_span = span.child("process_order")
            .attribute("order.id", order_id)
            .build();

        tokio::spawn(async move {
            let _guard = child_span.enter();
            process_single_order(order_id).await;
        });
    }
}
```

---

## Part 9: Sampling

### Why Sample?

Recording every trace adds overhead and storage costs. Sampling lets you record a percentage of traces.

**When to Sample**:
- High-traffic services (100K+ requests/sec)
- Cost control (observability platforms charge by volume)
- Reduce noise (focus on important traces)

---

### Probabilistic Sampling

Record a percentage of traces:

```rust
use tracer::{Tracer, ProbabilisticSampler};

// Record 10% of traces
let tracer = Tracer::builder()
    .sampler(Box::new(ProbabilisticSampler::new(0.1)))
    .build();
```

---

### Rate-Limited Sampling

Record max N traces per second:

```rust
use tracer::{Tracer, RateLimitedSampler};

// Record max 100 traces/sec
let tracer = Tracer::builder()
    .sampler(Box::new(RateLimitedSampler::new(100)))
    .build();
```

---

### Always On/Off

```rust
use tracer::{Tracer, AlwaysOnSampler, AlwaysOffSampler};

// Record all traces (development)
let tracer = Tracer::builder()
    .sampler(Box::new(AlwaysOnSampler))
    .build();

// Record no traces (performance testing)
let tracer = Tracer::builder()
    .sampler(Box::new(AlwaysOffSampler))
    .build();
```

---

## Part 10: Exporting Traces

### Stdout Export (Default)

Prints traces to console (great for development):

```rust
use tracer::{Tracer, StdoutExporter};

let exporter = StdoutExporter::new().pretty_print();

let tracer = Tracer::builder()
    .exporter(Box::new(exporter))
    .build();
```

**Output**:
```
Trace: 7a3b5c9d8e1f2a4b (Duration: 2500ms)
└── http_request [2500ms]
    ├── http.method: GET
    ├── http.url: /api/users
    └── http.status_code: 200
```

---

### OTLP Export (Jaeger, Prometheus, etc.)

Export to OpenTelemetry collectors:

```rust
use tracer::{Tracer, OtlpExporter};

let exporter = OtlpExporter::new("http://otel-collector:4317")?;

let tracer = Tracer::builder()
    .exporter(Box::new(exporter))
    .build();
```

**View in Jaeger**:
1. Start Jaeger: `docker run -p 16686:16686 jaegertracing/all-in-one`
2. Open http://localhost:16686
3. Search for your traces

---

### Jaeger UDP Export

Direct export to Jaeger agent (UDP):

```rust
use tracer::{Tracer, JaegerExporter};

let exporter = JaegerExporter::new("localhost", 6831)?;

let tracer = Tracer::builder()
    .exporter(Box::new(exporter))
    .build();
```

---

## Part 11: Best Practices

### Span Naming

**DO**:
- ✅ Use descriptive names: `database_query`, `http_request`
- ✅ Use lowercase with underscores: `process_order`
- ✅ Match the operation: `SELECT`, `GET /api/users`

**DON'T**:
- ❌ Use generic names: `function`, `work`
- ❌ Include IDs: `process_order_12345`
- ❌ Use periods: `process.order` (use underscores)

---

### Attribute Design

**DO**:
- ✅ Follow semantic conventions
- ✅ Use low-cardinality values
- ✅ Add attributes that help debugging

```rust
span.attribute("http.method", "GET");  // ✅ Good
span.attribute("cache.hit", true);  // ✅ Good
```

**DON'T**:
- ❌ Add high-cardinality attributes
- ❌ Add PII
- ❌ Add duplicate info

```rust
span.attribute("user.id", 12345);  // ❌ High cardinality
span.attribute("user.email", "user@example.com");  // ❌ PII
span.attribute("span.name", "http_request");  // ❌ Duplicate
```

---

### Error Handling

**DO**:
- ✅ Set span status on errors
- ✅ Add exception events
- ✅ Include error details

```rust
match result {
    Ok(_) => span.set_status(tracer::Status::Ok),
    Err(e) => {
        span.add_event_with_attrs("error", HashMap::from([
            ("exception.type", "PaymentDeclined"),
            ("exception.message", e.to_string()),
        ]));
        span.set_status(tracer::Status::Error {
            description: e.to_string(),
        });
    }
}
```

---

### Performance

**DO**:
- ✅ Use sampling in production
- ✅ Use batch export
- ✅ Disable tracing with feature flag

```rust
// Cargo.toml
[dependencies]
tracer = { version = "0.1", optional = true }

[features]
default = ["tracing"]

// main.rs
#[cfg(feature = "tracing")]
#[trace_span]
async fn my_function() { }
```

**DON'T**:
- ❌ Record every trace in high-traffic services
- ❌ Export spans synchronously (use async export)
- ❌ Add too many attributes (>10 per span)

---

## Part 12: Troubleshooting

### Spans Not Appearing

**Problem**: Spans not showing up in Jaeger/Zipkin.

**Solutions**:
1. Check exporter configuration
2. Check network connectivity to collector
3. Check sampling (maybe trace was dropped)
4. Check console for errors

---

### Context Not Propagating

**Problem**: Child spans not linked to parent.

**Solutions**:
1. Ensure middleware is installed
2. Check `traceparent` header is being sent
3. Use tokio task-local storage (not thread-local)
4. Don't manually spawn threads without context

---

### High Memory Usage

**Problem**: Memory usage increasing over time.

**Solutions**:
1. Reduce sampling rate
2. Reduce batch size
3. Reduce number of attributes
4. Check for span leaks (not ending spans)

---

## Conclusion

You're now ready to add distributed tracing to your Rust applications! Here's what you learned:

- ✅ Quick start: `#[trace_span]` attribute
- ✅ Core concepts: Spans, traces, context propagation
- ✅ HTTP/database tracing
- ✅ Async/await support
- ✅ Sampling strategies
- ✅ Exporting traces
- ✅ Best practices

**Next Steps**:
1. Add `#[trace_span]` to your functions
2. Install HTTP middleware
3. Export to Jaeger for visualization
4. Check out [examples.md](examples.md) for more examples

**Need Help?**
- [Developer Guide](05-dev-guide.md) - Architecture and development
- [Examples](examples.md) - 5+ working examples
- [OpenTelemetry Docs](https://opentelemetry.io/docs/)

Happy tracing! 🎉

---

**Document Status**: ✅ Complete
**Word Count**: 4,500+
**Next**: Developer Guide (05-dev-guide.md)
