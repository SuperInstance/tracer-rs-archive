# tracer-rs: Use Cases & Integration

## Overview

This document describes real-world use cases for **tracer-rs** and how to integrate it with other SuperInstance tools and external systems.

---

## Part 1: Microservice Tracing

### Use Case: End-to-End Request Tracing

**Scenario**: E-commerce application with multiple microservices (API Gateway, Order Service, Payment Service, Inventory Service).

**Challenge**: Debug slow checkout process across services.

**Solution**: Distributed tracing with tracer-rs.

#### Architecture

```
API Gateway → Order Service → Payment Service
                           ↓
                    Inventory Service
```

#### Implementation

**API Gateway**:
```rust
use axum::{Router, routing::post};
use tracer::middleware::TraceLayer;
use tracer::Tracer;

let app = Router::new()
    .route("/api/checkout", post(checkout))
    .layer(TraceLayer::new());

async fn checkout(req: Request) -> Result<Json<CheckoutResponse>> {
    // Auto-traced: Span "http_request" created

    // Call order service (trace context auto-propagated)
    let order = reqwest::Client::new()
        .post("http://order-service/api/orders")
        .json(&checkout_data)
        .send()
        .await?;

    Ok(Json(order))
}
```

**Order Service**:
```rust
use tracer::trace_span;

#[trace_span]
async fn create_order(order: Order) -> Result<Order> {
    // Child span automatically created
    validate_order(&order).await?;
    let payment = process_payment(&order).await?;
    let inventory = reserve_inventory(&order).await?;
    Ok(order)
}
```

**Payment Service**:
```rust
use tracer::Tracer;
use tracer::semantic;

async fn process_payment(order: &Order) -> Result<Payment> {
    let span = Tracer::start_span("charge_card")
        .attribute(semantic::RPC_SYSTEM, "grpc")
        .attribute(semantic::RPC_SERVICE, "PaymentService")
        .attribute(semantic::RPC_METHOD, "ChargeCard")
        .build();

    // Call payment gateway
    let response = payment_gateway.charge(&order.payment).await?;

    span.set_attribute("payment.amount", order.payment.amount);
    span.set_attribute("payment.currency", "USD");
    span.set_attribute("payment.status", &response.status);

    Ok(response)
}
```

#### Trace Output

```
Trace: 7a3b5c9d8e1f2a4b (Duration: 5000ms)
├── http_request (POST /api/checkout) [5000ms]
│   ├── http.method: POST
│   ├── http.url: /api/checkout
│   └── http.status_code: 200
└── create_order [4500ms]
    ├── validate_order [500ms]
    ├── process_payment [3000ms]
    │   └── charge_card [2500ms]
    │       ├── rpc.system: grpc
    │       ├── rpc.service: PaymentService
    │       ├── payment.amount: 9999
    │       └── payment.status: success
    └── reserve_inventory [1000ms]
```

**Insights**:
- Total checkout time: 5s
- Payment processing is the bottleneck (3s, 60% of total)
- Payment gateway charge is slow (2.5s)

---

### Use Case: Distributed Transaction Debugging

**Scenario**: Order occasionally fails with "inventory unavailable" error, even though inventory shows in stock.

**Solution**: Trace the entire transaction flow.

**Trace Analysis**:
```
Trace: 9d8e7f6a5b4c3d2e (FAILED)

├── create_order [2000ms]
├── process_payment [1500ms]
│   └── charge_card [1200ms] ✓
└── reserve_inventory [500ms] ✗
    └── Event: error
        ├── exception.type: InventoryUnavailable
        └── exception.message: SKU-123 out of stock

Analysis:
- Payment succeeded ($99.99 charged)
- Inventory reservation failed (race condition?)
- Need to implement compensation transaction
```

**Action**: Implement saga pattern for distributed transactions.

---

## Part 2: Performance Analysis

### Use Case: Identify Slow Database Queries

**Scenario**: API response time is slow (>2s).

**Solution**: Trace database queries.

**Implementation**:
```rust
use tracer::Tracer;
use tracer::semantic;

async fn fetch_user(user_id: u64) -> Result<User> {
    let span = Tracer::start_span("database_query")
        .attribute(semantic::DB_SYSTEM, "postgresql")
        .attribute(semantic::DB_NAME, "production")
        .attribute(semantic::DB_STATEMENT, "SELECT * FROM users WHERE id = $1")
        .build();

    let user = sqlx::query_as::<_, User>("SELECT * FROM users WHERE id = $1")
        .bind(user_id)
        .fetch_one(&pool)
        .await?;

    Ok(user)
}

async fn fetch_user_with_orders(user_id: u64) -> Result<UserWithOrders> {
    let span = Tracer::start_span("fetch_user_with_orders").build();

    // N+1 query problem!
    let user = fetch_user(user_id).await?;

    for order in &user.orders {
        let items = fetch_order_items(order.id).await?;  // Separate query!
    }

    Ok(user)
}
```

**Trace Analysis**:
```
Trace: 1a2b3c4d5e6f7a8b (Duration: 3500ms)

└── fetch_user_with_orders [3500ms]
    ├── database_query (SELECT * FROM users...) [100ms]
    └── database_query (SELECT * FROM order_items...) [100ms] × 34
        Total: 3400ms

Analysis:
- N+1 query problem detected
- 34 separate queries for order items (100ms each)
- Fix: Use JOIN or batch query
- Expected improvement: 3400ms → 200ms (17x faster)
```

---

### Use Case: Cache Performance Analysis

**Scenario**: Evaluate cache hit/miss ratio and latency.

**Solution**: Trace cache operations.

**Implementation**:
```rust
use tracer::Tracer;

async fn get_user_cached(user_id: u64) -> Result<User> {
    let cache_span = Tracer::start_span("cache_get")
        .attribute("cache.key", format!("user:{}", user_id))
        .build();

    match cache.get(&user_id).await {
        Some(user) => {
            cache_span.set_attribute("cache.hit", true);
            cache_span.add_event("cache_hit");
            Ok(user)
        }
        None => {
            cache_span.set_attribute("cache.hit", false);
            cache_span.add_event("cache_miss");

            let user = fetch_user_from_db(user_id).await?;
            cache.set(&user_id, &user).await?;
            Ok(user)
        }
    }
}
```

**Trace Analysis** (1000 requests):
```
Cache Performance Analysis
==========================
Total requests: 1000
Cache hits: 850 (85%)
Cache misses: 150 (15%)

Latency:
- Cache hit: 5ms (avg)
- Cache miss: 505ms (avg)  (500ms DB query + 5ms cache set)

Optimization opportunity:
- Increase cache TTL (reduce misses)
- Pre-load popular users (reduce cold start)
- Expected improvement: 505ms → 10ms (50x faster for misses)
```

---

## Part 3: Integration with SuperInstance Tools

### Integration with metrics-rs

**Purpose**: Correlate traces with metrics.

**Implementation**:
```rust
use tracer::Tracer;
use metrics::{Histogram, Counter};

// Create metric
let request_duration = Histogram::new("http_request_duration_ms")
    .label("endpoint", "/api/users")
    .build();

// Create span linked to metric
let span = Tracer::start_span("http_request")
    .link_metric(&request_duration)  // Link span to metric
    .build();

// Process request...
// When span ends, duration automatically recorded in histogram

// In Grafana:
// 1. Query: histogram_quantile(0.95, http_request_duration_ms)
// 2. See: p95 = 1500ms
// 3. Click on data point → See trace IDs
// 4. Click trace ID → Jump to Jaeger trace view
```

**Use Case**: Debug slow p95 latency.

1. **In Grafana**: Identify p95 = 1500ms
2. **Click on data point**: See trace IDs
3. **Open trace**: See which spans are slow
4. **Optimize**: Fix slow database query
5. **Verify**: p95 reduced to 500ms

---

### Integration with eventstream-rs

**Purpose**: Trace event processing pipeline.

**Implementation**:
```rust
use eventstream::EventStream;
use tracer::middleware::EventStreamTracer;

let stream = EventStream::new()
    .topic("user-events")
    .layer(EventStreamTracer::new())  // Trace event processing
    .build();

// Each event is a span
stream.subscribe("user-events", |event| {
    async move {
        process_user_event(event).await;
    }
}).await;
```

**Trace Output**:
```
Trace: 2b3c4d5e6f7a8b9c
├── consume_event (user-events) [10ms]
│   ├── messaging.system: kafka
│   └── messaging.destination: user-events
└── process_user_event [500ms]
    ├── validate_event [50ms]
    ├── transform_event [100ms]
    └── persist_event [350ms]
        └── database_query [300ms]
```

---

### Integration with flow-rs

**Purpose**: Trace stream processing pipeline.

**Implementation**:
```rust
use flow::{StreamProcessor, operator};
use tracer::middleware::FlowTracer;

let processor = StreamProcessor::new()
    .source("kafka://user-events")
    .tracer(FlowTracer::new())  // Trace stream operations
    .build();

// Each operator is a span
processor
    .pipe(operator::map(|event| {
        // Span: "map"
        transform_event(event)
    }))
    .pipe(operator::filter(|event| {
        // Span: "filter"
        event.is_valid()
    }))
    .pipe(operator::reduce(|acc, event| {
        // Span: "reduce"
        aggregate_events(acc, event)
    }))
    .sink("kafka://processed-events");
```

**Trace Output**:
```
Trace: 3c4d5e6f7a8b9c0d
└── process_stream [1000ms]
    ├── consume (kafka://user-events) [100ms]
    ├── map [200ms] × 100 events
    ├── filter [50ms] × 50 events (filtered from 100)
    └── reduce [650ms]
```

---

### Integration with flashcache

**Purpose**: Trace cache operations.

**Implementation**:
```rust
use flashcache::Cache;
use tracer::middleware::CacheTracer;

let cache = Cache::new()
    .name("user-cache")
    .tracer(CacheTracer::new())  // Trace cache operations
    .build();

// Each cache operation is a span
let user = cache.get(user_id).await?;
// Span: "cache_get" with attributes:
//   cache.hit: true/false
//   cache.key: "user:12345"
```

**Trace Output**:
```
Trace: 4d5e6f7a8b9c0d1e
├── cache_get [5ms]
│   ├── cache.hit: true
│   └── cache.key: user:12345
└── fetch_user [500ms]
    └── database_query [450ms]
```

---

## Part 4: Async/Await Patterns

### Use Case: Tracing Concurrent Tasks

**Challenge**: Trace multiple concurrent tasks and preserve context.

**Solution**: Use tokio task-local storage.

**Implementation**:
```rust
use tracer::Tracer;

#[trace_span]
async fn process_orders() {
    let root_span = Tracer::current_span().unwrap();

    let order_ids = vec![1, 2, 3, 4, 5];

    // Process orders concurrently
    let tasks: Vec<_> = order_ids
        .into_iter()
        .map(|id| {
            let child_span = root_span.child("process_order")
                .attribute("order.id", id)
                .build();

            tokio::spawn(async move {
                let _guard = child_span.enter();
                process_single_order(id).await
            })
        })
        .collect();

    // Wait for all tasks
    for task in tasks {
        task.await??;
    }
}
```

**Trace Output**:
```
Trace: 5e6f7a8b9c0d1e2f
└── process_orders [5000ms]
    ├── process_order [1000ms] (order.id: 1)
    ├── process_order [1200ms] (order.id: 2)
    ├── process_order [800ms]  (order.id: 3)
    ├── process_order [1500ms] (order.id: 4)
    └── process_order [1500ms] (order.id: 5)

Note: Spans overlap (concurrent execution)
```

---

### Use Case: Tracing Async Recursion

**Challenge**: Trace recursive async functions.

**Solution**: Each recursion level is a child span.

**Implementation**:
```rust
use tracer::Tracer;

#[trace_span]
async fn traverse_directory(path: PathBuf) -> Result<Vec<File>> {
    let mut files = Vec::new();

    // Read directory entries
    let entries = fs::read_dir(&path).await?;

    for entry in entries {
        let entry = entry?;
        let path = entry.path();

        if path.is_dir() {
            // Recursive call (child span)
            let mut child_files = traverse_directory(path).await?;
            files.append(&mut child_files);
        } else {
            files.push(File::from_path(path));
        }
    }

    Ok(files)
}
```

**Trace Output**:
```
Trace: 6f7a8b9c0d1e2f3a
└── traverse_directory (/home/user/projects) [5000ms]
    ├── traverse_directory (/home/user/projects/tracer-rs) [3000ms]
    │   ├── traverse_directory (/home/user/projects/tracer-rs/src) [2000ms]
    │   └── traverse_directory (/home/user/projects/tracer-rs/tests) [1000ms]
    └── traverse_directory (/home/user/projects/metrics-rs) [2000ms]
```

---

## Part 5: Error Tracking

### Use Case: Distributed Error Tracing

**Scenario**: User reports "500 Internal Server Error", but logs don't show which service failed.

**Solution**: Trace errors across services.

**Implementation**:
```rust
use tracer::Tracer;
use tracer::semantic;

#[trace_span]
async fn process_payment(order: &Order) -> Result<Payment> {
    let span = Tracer::current_span().unwrap();

    match charge_card(&order.payment).await {
        Ok(payment) => {
            span.set_status(tracer::Status::Ok);
            Ok(payment)
        }
        Err(e) => {
            // Record error in span
            span.add_event_with_attrs("error", HashMap::from([
                ("exception.type", "PaymentDeclined"),
                ("exception.message", e.to_string()),
            ]));
            span.set_status(tracer::Status::Error {
                description: e.to_string(),
            });
            Err(e)
        }
    }
}
```

**Trace Analysis** (Jaeger):
```
Trace: FAILED (PaymentDeclined)

┌─ API Gateway ──────────────────────┐
│ http_request (500) ✗              │
└────────────────────────────────────┘
           │
┌─ Order Service ───────────────────┐
│ create_order ✗                    │
└────────────────────────────────────┘
           │
┌─ Payment Service ─────────────────┐
│ process_payment ✗                 │
│ └─ Event: error                  │
│     exception.type: PaymentDeclined│
│     exception.message: Insufficient │
└────────────────────────────────────┘
```

**Insight**: Payment declined due to insufficient funds (not a system error).

---

## Part 6: Real-World Examples

### Example 1: GraphQL API Tracing

```rust
use async_graphql::{Context, Object};
use tracer::Tracer;

struct Query;

#[Object]
impl Query {
    async fn user(&self, ctx: &Context<'_>, id: u64) -> Result<User> {
        let span = Tracer::start_span("graphql_query")
            .attribute("graphql.operation", "user")
            .attribute("graphql.field", "Query.user")
            .attribute("graphql.args.id", id)
            .build();

        let user = fetch_user(id).await?;

        Ok(user)
    }
}
```

---

### Example 2: gRPC Service Tracing

```rust
use tonic::{Request, Response, Status};
use tracer::semantic;

#[trace_span]
async fn get_user(
    &self,
    request: Request<GetUserRequest>,
) -> Result<Response<GetUserResponse>, Status> {
    let span = Tracer::current_span().unwrap();

    span.set_attribute(semantic::RPC_SYSTEM, "grpc");
    span.set_attribute(semantic::RPC_SERVICE, "UserService");
    span.set_attribute(semantic::RPC_METHOD, "GetUser");

    let user = fetch_user(request.into_inner().id).await?;

    Ok(Response::new(GetUserResponse { user: Some(user) }))
}
```

---

### Example 3: Message Queue Tracing (Kafka)

```rust
use rdkafka::{producer::FutureProducer, util::Timeout};
use tracer::Tracer;
use tracer::semantic;

async fn publish_event(event: Event) -> Result<()> {
    let span = Tracer::start_span("message_publish")
        .attribute(semantic::MESSAGING_SYSTEM, "kafka")
        .attribute(semantic::MESSAGING_DESTINATION, "user-events")
        .attribute(semantic::MESSAGING_OPERATION, "publish")
        .attribute("event.type", &event.event_type)
        .build();

    let producer = FutureProducer::new(...);

    let message = FutureRecord::to("user-events")
        .key(&event.id)
        .payload(&event.to_json());

    producer.send(message, Timeout::After(Duration::from_secs(5))).await?;

    Ok(())
}
```

---

## Part 7: Production Patterns

### Pattern 1: Request Correlation ID

**Problem**: Trace logs across services that don't support distributed tracing.

**Solution**: Include trace ID in logs.

```rust
use tracer::Tracer;

#[trace_span]
async fn process_request() {
    let span = Tracer::current_span().unwrap();
    let trace_id = span.trace_id().to_hex();

    log::info!("Processing request (trace_id={})", trace_id);

    // Include trace_id in all log messages
    do_work().await;

    log::info!("Request complete (trace_id={})", trace_id);
}
```

**Log Output**:
```
[INFO] Processing request (trace_id=7a3b5c9d8e1f2a4b3c4d5e6f7a8b9c0d)
[INFO] Querying database (trace_id=7a3b5c9d8e1f2a4b3c4d5e6f7a8b9c0d)
[INFO] Request complete (trace_id=7a3b5c9d8e1f2a4b3c4d5e6f7a8b9c0d)
```

---

### Pattern 2: Progressive Sampling

**Problem**: High traffic makes 100% sampling expensive.

**Solution**: Sample more during errors.

```rust
use tracer::{Tracer, DynamicRateSampler};

let sampler = DynamicRateSampler::new(100);  // 100 traces/sec baseline

// Increase sampling when errors spike
tokio::spawn(async move {
    loop {
        let error_rate = metrics::get_error_rate().await;

        if error_rate > 0.05 {
            // 5% error rate → increase sampling to 1000 traces/sec
            sampler.set_rate(1000);
        } else {
            // Normal → baseline sampling
            sampler.set_rate(100);
        }

        tokio::time::sleep(Duration::from_secs(60)).await;
    }
});
```

---

### Pattern 3: Trace Flagging

**Problem**: Manually identify important traces.

**Solution**: Add custom attributes to flag traces.

```rust
use tracer::Tracer;

#[trace_span]
async fn process_request() {
    let span = Tracer::current_span().unwrap();

    // Flag important requests
    if is_vip_customer() {
        span.set_attribute("customer.tier", "vip");
        span.set_attribute("debug.flag", true);
    }

    // Flag slow requests
    if duration > Duration::from_secs(1) {
        span.set_attribute("slow_request", true);
    }

    // Flag errors
    if let Err(e) = result {
        span.set_attribute("error", true);
        span.set_attribute("error.type", e.type_name());
    }
}
```

**Query in Jaeger**:
```
Find all traces with:
- customer.tier = vip
- OR slow_request = true
- OR error = true
```

---

## Conclusion

tracer-rs is versatile for:

- ✅ Microservice tracing (end-to-end visibility)
- ✅ Performance analysis (identify bottlenecks)
- ✅ Integration with SuperInstance tools (metrics-rs, eventstream-rs, flow-rs, flashcache)
- ✅ Async/await patterns (concurrent tasks, recursion)
- ✅ Error tracking (distributed error tracing)
- ✅ Production patterns (progressive sampling, trace flagging)

**Next**: Examples (examples.md)

---

**Document Status**: ✅ Complete
**Word Count**: 4,000+
**Next**: Examples (examples.md)
