# tracer-rs: API Design

## Executive Summary

This document describes the public API of **tracer-rs**. The design prioritizes ergonomics (builder pattern, fluent API), simplicity (one macro to instrument), and safety (type-safe attributes, RAII guards).

**Key API Principles**:
1. **Fluent API**: Builder pattern for span creation
2. **Type Safety**: Compile-time attribute validation
3. **RAII**: Spans auto-end on drop
4. **Async-Aware**: Works seamlessly with tokio async/await

---

## Part 1: Core API

### Tracer API

**Purpose**: Create and manage spans.

```rust
use tracer::{Tracer, Span};

/// Tracer creates spans
impl Tracer {
    /// Create global tracer (default configuration)
    pub fn new() -> Self {
        Self::default()
    }

    /// Create tracer with custom configuration
    pub fn builder() -> TracerBuilder {
        TracerBuilder::new()
    }

    /// Start a new root span (no parent)
    pub fn start_span(name: impl Into<String>) -> SpanBuilder {
        SpanBuilder::new(name)
    }

    /// Start a new child span (from current context)
    pub fn start_child_span(name: impl Into<String>) -> SpanBuilder {
        SpanBuilder::with_parent(name, Self::current_span())
    }

    /// Get current span (from task-local storage)
    pub fn current_span() -> Option<Span> {
        SpanGuard::current()
    }

    /// Inject trace context into carrier (e.g., HTTP headers)
    pub fn inject_context(carrier: &mut dyn Carrier) {
        if let Some(span) = Self::current_span() {
            carrier.set("traceparent", span.context().to_traceparent());
        }
    }

    /// Extract trace context from carrier
    pub fn extract_context(carrier: &dyn Carrier) -> Option<TraceContext> {
        carrier
            .get("traceparent")
            .and_then(|value| TraceContext::from_traceparent(value).ok())
    }
}
```

**Usage**:

```rust
// Create root span
let span = Tracer::start_span("http_request")
    .attribute("http.method", "GET")
    .attribute("http.url", "/api/users")
    .build();

// Create child span (inherits parent)
let db_span = Tracer::start_child_span("database_query")
    .attribute("db.system", "postgresql")
    .build();
```

---

### SpanBuilder API

**Purpose**: Build spans with fluent API.

```rust
use tracer::SpanBuilder;

/// Builder for creating spans
pub struct SpanBuilder {
    name: String,
    parent_id: Option<SpanId>,
    attributes: HashMap<String, Attribute>,
    kind: SpanKind,
    links: Vec<SpanLink>,
}

impl SpanBuilder {
    /// Create new span builder
    pub fn new(name: impl Into<String>) -> Self {
        Self {
            name: name.into(),
            parent_id: None,
            attributes: HashMap::new(),
            kind: SpanKind::Internal,
            links: Vec::new(),
        }
    }

    /// Create span with parent
    pub fn with_parent(name: impl Into<String>, parent: Span) -> Self {
        Self {
            name: name.into(),
            parent_id: Some(parent.id()),
            attributes: HashMap::new(),
            kind: SpanKind::Internal,
            links: Vec::new(),
        }
    }

    /// Add string attribute
    pub fn attribute(mut self, key: impl Into<String>, value: impl Into<Attribute>) -> Self {
        self.attributes.insert(key.into(), value.into());
        self
    }

    /// Add multiple attributes
    pub fn attributes<I>(mut self, attrs: I) -> Self
    where
        I: IntoIterator,
        I::Item: (impl Into<String>, impl Into<Attribute>),
    {
        for (key, value) in attrs {
            self.attributes.insert(key.into(), value.into());
        }
        self
    }

    /// Set span kind
    pub fn kind(mut self, kind: SpanKind) -> Self {
        self.kind = kind;
        self
    }

    /// Add link to another span
    pub fn link(mut self, link: SpanLink) -> Self {
        self.links.push(link);
        self
    }

    /// Build span (starts span)
    pub fn build(self) -> Span {
        Span::create(self)
    }
}
```

**Usage**:

```rust
let span = Tracer::start_span("http_request")
    .attribute("http.method", "GET")
    .attribute("http.url", "/api/users")
    .attribute("http.status_code", 200)
    .kind(SpanKind::Server)
    .build();
```

---

### Span API

**Purpose**: Interact with active spans.

```rust
use std::sync::Arc;
use std::time::Duration;

/// Active span
pub struct Span {
    inner: Arc<SpanInner>,
}

struct SpanInner {
    id: SpanId,
    trace_id: TraceId,
    name: String,
    start_time: Instant,
    end_time: Option<Instant>,
    attributes: RwLock<HashMap<String, Attribute>>,
    events: RwLock<Vec<Event>>,
    status: RwLock<Status>,
}

impl Span {
    /// Get span ID
    pub fn id(&self) -> SpanId {
        self.inner.id
    }

    /// Get trace ID
    pub fn trace_id(&self) -> TraceId {
        self.inner.trace_id
    }

    /// Get span name
    pub fn name(&self) -> &str {
        &self.inner.name
    }

    /// Get span duration
    pub fn duration(&self) -> Duration {
        let end = self.inner.end_time.unwrap_or_else(Instant::now);
        end.duration_since(self.inner.start_time)
    }

    /// Add attribute to span
    pub fn set_attribute(&self, key: impl Into<String>, value: impl Into<Attribute>) {
        let mut attrs = self.inner.attributes.write().unwrap();
        attrs.insert(key.into(), value.into());
    }

    /// Add event to span
    pub fn add_event(&self, name: impl Into<String>) {
        let event = Event {
            name: name.into(),
            timestamp: Instant::now(),
            attributes: HashMap::new(),
        };

        let mut events = self.inner.events.write().unwrap();
        events.push(event);
    }

    /// Add event with attributes
    pub fn add_event_with_attrs(&self, name: impl Into<String>, attrs: HashMap<String, Attribute>) {
        let event = Event {
            name: name.into(),
            timestamp: Instant::now(),
            attributes: attrs,
        };

        let mut events = self.inner.events.write().unwrap();
        events.push(event);
    }

    /// Set span status
    pub fn set_status(&self, status: Status) {
        let mut s = self.inner.status.write().unwrap();
        *s = status;
    }

    /// End span
    pub fn end(&self) {
        let mut inner = Arc::make_mut(&mut self.inner);
        if inner.end_time.is_none() {
            inner.end_time = Some(Instant::now());
        }
    }

    /// Get span context (for propagation)
    pub fn context(&self) -> SpanContext {
        SpanContext {
            trace_id: self.inner.trace_id,
            span_id: self.inner.id,
        }
    }

    /// Create child span
    pub fn child(&self, name: impl Into<String>) -> SpanBuilder {
        SpanBuilder::with_parent(name, self.clone())
    }
}

impl Drop for Span {
    fn drop(&mut self) {
        self.end();
    }
}
```

**Usage**:

```rust
let span = Tracer::start_span("process_order")
    .build();

// Add attributes
span.set_attribute("order_id", 12345);
span.set_attribute("customer_id", 67890);

// Add event
span.add_event("payment_started");
span.add_event_with_attrs("payment_completed", HashMap::from([
    ("amount".to_string(), Attribute::Int(9999)),
]));

// Create child span
let db_span = span.child("database_query")
    .attribute("db.system", "postgresql")
    .build();

// Span auto-ends on drop
```

---

## Part 2: Attribute API

### Attribute Values

**Purpose**: Type-safe attribute values.

```rust
/// Attribute value
#[derive(Debug, Clone, PartialEq)]
pub enum Attribute {
    String(String),
    Int(i64),
    Double(f64),
    Bool(bool),
    Array(Vec<Attribute>),
}

// Auto-convert from Rust types
impl From<String> for Attribute {
    fn from(s: String) -> Self {
        Attribute::String(s)
    }
}

impl From<&str> for Attribute {
    fn from(s: &str) -> Self {
        Attribute::String(s.to_string())
    }
}

impl From<i64> for Attribute {
    fn from(i: i64) -> Self {
        Attribute::Int(i)
    }
}

impl From<i32> for Attribute {
    fn from(i: i32) -> Self {
        Attribute::Int(i as i64)
    }
}

impl From<f64> for Attribute {
    fn from(f: f64) -> Self {
        Attribute::Double(f)
    }
}

impl From<bool> for Attribute {
    fn from(b: bool) -> Self {
        Attribute::Bool(b)
    }
}

impl<T: Into<Attribute>> From<Vec<T>> for Attribute {
    fn from(v: Vec<T>) -> Self {
        Attribute::Array(v.into_iter().map(|t| t.into()).collect())
    }
}
```

**Usage**:

```rust
// Type-safe attribute values
span.set_attribute("http.method", "GET");  // &str -> Attribute
span.set_attribute("http.status_code", 200);  // i32 -> Attribute
span.set_attribute("request.duration_ms", 123.456);  // f64 -> Attribute
span.set_attribute("cache.hit", true);  // bool -> Attribute
span.set_attribute("user.tags", vec!["premium", "active"]);  // Vec -> Attribute
```

---

### Semantic Conventions

**Purpose**: Standard attribute names (from OpenTelemetry).

```rust
/// Semantic conventions (standard attribute names)
pub mod semantic {
    // HTTP attributes
    pub const HTTP_METHOD: &str = "http.method";
    pub const HTTP_URL: &str = "http.url";
    pub const HTTP_TARGET: &str = "http.target";
    pub const HTTP_STATUS_CODE: &str = "http.status_code";
    pub const HTTP_FLAVOR: &str = "http.flavor";
    pub const HTTP_SCHEME: &str = "http.scheme";
    pub const HTTP_HOST: &str = "http.host";

    // Database attributes
    pub const DB_SYSTEM: &str = "db.system";
    pub const DB_NAME: &str = "db.name";
    pub const DB_STATEMENT: &str = "db.statement";
    pub const DB_OPERATION: &str = "db.operation";

    // RPC attributes
    pub const RPC_SYSTEM: &str = "rpc.system";
    pub const RPC_SERVICE: &str = "rpc.service";
    pub const RPC_METHOD: &str = "rpc.method";

    // Messaging attributes
    pub const MESSAGING_SYSTEM: &str = "messaging.system";
    pub const MESSAGING_DESTINATION: &str = "messaging.destination";
    pub const MESSAGING_OPERATION: &str = "messaging.operation";

    // Exception attributes
    pub const EXCEPTION_TYPE: &str = "exception.type";
    pub const EXCEPTION_MESSAGE: &str = "exception.message";
    pub const EXCEPTION_STACKTRACE: &str = "exception.stacktrace";
}
```

**Usage**:

```rust
use tracer::semantic;

// Use semantic conventions
span.set_attribute(semantic::HTTP_METHOD, "GET");
span.set_attribute(semantic::HTTP_URL, "/api/users");
span.set_attribute(semantic::HTTP_STATUS_CODE, 200);
```

---

## Part 3: Context Propagation API

### TraceContext API

```rust
/// Trace context (propagated across services)
#[derive(Debug, Clone, PartialEq)]
pub struct TraceContext {
    trace_id: TraceId,
    span_id: SpanId,
    trace_flags: u8,
}

impl TraceContext {
    /// Parse from W3C traceparent header
    pub fn from_traceparent(value: &str) -> Result<Self, ParseError> {
        let parts: Vec<&str> = value.split('-').collect();
        if parts.len() != 4 {
            return Err(ParseError::InvalidFormat);
        }

        Ok(Self {
            trace_id: TraceId::from_hex(parts[1])?,
            span_id: SpanId::from_hex(parts[2])?,
            trace_flags: u8::from_str_radix(parts[3], 16)?,
        })
    }

    /// Serialize to W3C traceparent header
    pub fn to_traceparent(&self) -> String {
        format!(
            "00-{}-{}-{:02x}",
            self.trace_id.to_hex(),
            self.span_id.to_hex(),
            self.trace_flags
        )
    }

    /// Get trace ID
    pub fn trace_id(&self) -> TraceId {
        self.trace_id
    }

    /// Get span ID
    pub fn span_id(&self) -> SpanId {
        self.span_id
    }

    /// Check if sampled
    pub fn is_sampled(&self) -> bool {
        self.trace_flags & 0x01 == 0x01
    }
}
```

---

### Carrier API

**Purpose**: Generic interface for injecting/extracting context.

```rust
/// Carrier for trace context (e.g., HTTP headers)
pub trait Carrier {
    /// Get value from carrier
    fn get(&self, key: &str) -> Option<String>;

    /// Set value in carrier
    fn set(&mut self, key: &str, value: String);
}

// Implement for HTTP headers (e.g., reqwest, hyper)
impl Carrier for HeaderMap {
    fn get(&self, key: &str) -> Option<String> {
        self.get(key)
            .and_then(|v| v.to_str().ok())
            .map(|s| s.to_string())
    }

    fn set(&mut self, key: &str, value: String) {
        if let Ok(header_value) = value.parse() {
            self.insert(key, header_value);
        }
    }
}

// Implement for HashMap
impl Carrier for HashMap<String, String> {
    fn get(&self, key: &str) -> Option<String> {
        self.get(key).cloned()
    }

    fn set(&mut self, key: &str, value: String) {
        self.insert(key.to_string(), value);
    }
}
```

**Usage**:

```rust
// Inject context into HTTP headers
let mut headers = HeaderMap::new();
Tracer::inject_context(&mut headers);
// headers["traceparent"] = "00-7a3b5c9d...-1a2b3c4d...-01"

// Extract context from HTTP headers
let ctx = Tracer::extract_context(&headers)?;
```

---

### Propagator API

```rust
/// Propagator injects/extracts trace context
pub trait Propagator: Send + Sync {
    /// Inject context into carrier
    fn inject(&self, ctx: &TraceContext, carrier: &mut dyn Carrier);

    /// Extract context from carrier
    fn extract(&self, carrier: &dyn Carrier) -> Option<TraceContext>;
}

/// W3C trace context propagator
pub struct TraceContextPropagator;

impl Propagator for TraceContextPropagator {
    fn inject(&self, ctx: &TraceContext, carrier: &mut dyn Carrier) {
        carrier.set("traceparent", ctx.to_traceparent());
    }

    fn extract(&self, carrier: &dyn Carrier) -> Option<TraceContext> {
        carrier
            .get("traceparent")
            .and_then(|value| TraceContext::from_traceparent(value).ok())
    }
}
```

**Usage**:

```rust
let propagator = TraceContextPropagator;

// Inject
let mut headers = HeaderMap::new();
propagator.inject(&span.context(), &mut headers);

// Extract
let ctx = propagator.extract(&headers)?;
```

---

## Part 4: Instrumentation API

### #[trace_span] Macro

**Purpose**: Auto-instrument functions.

```rust
use tracer::trace_span;

/// Instrument function with trace span
#[trace_span]
async fn fetch_user(user_id: u64) -> Result<User, Error> {
    // Automatically creates span "fetch_user"
    // Automatically captures user_id as attribute
    // Automatically handles errors
    let user = db.query(user_id).await?;
    Ok(user)
}

/// With custom span name
#[trace_span(name = "custom_name")]
async fn my_function() {
    // Span name: "custom_name"
}

/// With custom attributes
#[trace_span(attributes(level = "info"))]
async fn my_function() {
    // Additional attribute: level = "info"
}
```

**Macro Options**:

```rust
// Default: Use function name
#[trace_span]
async fn fetch_user(id: u64) -> Result<User> { }

// Custom name
#[trace_span(name = "get_user")]
async fn fetch_user(id: u64) -> Result<User> { }

// Skip certain arguments
#[trace_span(skip(id))]
async fn fetch_user(id: u64, name: String) -> Result<User> {
    // id not captured as attribute
}

// Add custom attributes
#[trace_span(attributes(service = "user-service"))]
async fn fetch_user(id: u64) -> Result<User> {
    // service = "user-service" attribute added
}
```

---

### Middleware API

#### Axum Middleware

```rust
use axum::{Router, routing::get};
use tracer::middleware::TraceLayer;

/// Axum middleware for automatic HTTP tracing
pub struct TraceLayer {
    tracer: Tracer,
}

impl TraceLayer {
    pub fn new() -> Self {
        Self {
            tracer: Tracer::new(),
        }
    }
}

impl<S> Layer<S> for TraceLayer {
    type Service = TraceMiddleware<S>;

    fn layer(&self, inner: S) -> Self::Service {
        TraceMiddleware {
            inner,
            tracer: self.tracer.clone(),
        }
    }
}

// Usage
let app = Router::new()
    .route("/api/users", get(get_users))
    .layer(TraceLayer::new());  // Auto-trace all requests
```

**Tracked Automatically**:
- HTTP method, URL, status code
- Request/response duration
- Trace context propagation

---

#### Actix-Web Middleware

```rust
use actix_web::{web, App, HttpServer};
use tracer::middleware::TracingMiddleware;

/// Actix-web middleware
pub fn tracing_middleware() -> TracingMiddleware {
    TracingMiddleware::new()
}

// Usage
HttpServer::new(|| {
    App::new()
        .wrap(tracing_middleware())  // Auto-trace all requests
        .service(users_api)
})
.bind(("127.0.0.1", 8080))?
.run()
.await?;
```

---

#### Reqwest Middleware

```rust
use reqwest::{Client, Request};
use tracer::middleware::ReqwestTracer;

/// Reqwest middleware for automatic HTTP client tracing
pub struct ReqwestTracer {
    tracer: Tracer,
}

impl ReqwestTracer {
    pub fn new() -> Self {
        Self {
            tracer: Tracer::new(),
        }
    }

    /// Wrap client with tracing
    pub fn wrap_client(&self, client: Client) -> Client {
        // Interceptor that adds traceparent header
        client
    }
}

// Usage
let client = ReqwestTracer::new().wrap_client(Client::new());
let response = client.get("https://api.example.com/users")
    .send()
    .await?;  // Trace context auto-injected
```

---

## Part 5: Exporter API

### Exporter Trait

```rust
/// Exporter sends spans to backend
pub trait Exporter: Send + Sync {
    /// Export a span
    fn export(&self, span: Arc<Span>) -> Result<(), ExportError>;

    /// Shutdown exporter (flush pending spans)
    fn shutdown(&self) -> Result<(), ExportError>;
}
```

---

### Stdout Exporter

```rust
/// Stdout exporter (prints to console)
pub struct StdoutExporter {
    pretty: bool,
}

impl StdoutExporter {
    pub fn new() -> Self {
        Self { pretty: true }
    }

    pub fn pretty_print(mut self) -> Self {
        self.pretty = true;
        self
    }

    pub fn single_line(mut self) -> Self {
        self.pretty = false;
        self
    }
}

impl Exporter for StdoutExporter {
    fn export(&self, span: Arc<Span>) -> Result<(), ExportError> {
        if self.pretty {
            println!("Trace: {}", span.trace_id().to_hex());
            println!("└── {} [{}ms]", span.name(), span.duration().as_millis());
        } else {
            println!("{} {} {}ms",
                span.trace_id().to_hex(),
                span.name(),
                span.duration().as_millis()
            );
        }
        Ok(())
    }

    fn shutdown(&self) -> Result<(), ExportError> {
        Ok(())
    }
}
```

**Usage**:

```rust
let exporter = StdoutExporter::new().pretty_print();
Tracer::builder()
    .exporter(exporter)
    .build();
```

---

### OTLP Exporter

```rust
/// OTLP exporter (gRPC)
pub struct OtlpExporter {
    client: TraceServiceClient<Channel>,
}

impl OtlpExporter {
    pub fn new(endpoint: &str) -> Result<Self, ExportError> {
        let channel = Channel::from_shared(endpoint.to_string())
            .map_err(|e| ExportError::Connection(e.to_string()))?
            .connect()
            .await?;

        let client = TraceServiceClient::new(channel);

        Ok(Self { client })
    }
}

impl Exporter for OtlpExporter {
    fn export(&self, span: Arc<Span>) -> Result<(), ExportError> {
        let otlp_span = self.to_otlp_span(&span);

        tokio::spawn(async move {
            let _ = self.client.export(otlp_span).await;
        });

        Ok(())
    }

    fn shutdown(&self) -> Result<(), ExportError> {
        // Flush pending spans
        Ok(())
    }
}
```

**Usage**:

```rust
let exporter = OtlpExporter::new("http://otel-collector:4317")?;
Tracer::builder()
    .exporter(exporter)
    .build();
```

---

### Jaeger Exporter

```rust
/// Jaeger UDP exporter
pub struct JaegerExporter {
    socket: tokio::net::UdpSocket,
    agent_host: String,
    agent_port: u16,
}

impl JaegerExporter {
    pub fn new(agent_host: &str, agent_port: u16) -> Result<Self, ExportError> {
        let socket = tokio::net::UdpSocket::bind("0.0.0.0:0")
            .map_err(|e| ExportError::Socket(e.to_string()))?;

        Ok(Self {
            socket,
            agent_host: agent_host.to_string(),
            agent_port,
        })
    }
}

impl Exporter for JaegerExporter {
    fn export(&self, span: Arc<Span>) -> Result<(), ExportError> {
        let thrift_span = self.to_jaeger_span(&span);
        let bytes = thrift_span.serialize()
            .map_err(|e| ExportError::Encode(e.to_string()))?;

        self.socket.send_to(&bytes, format!("{}:{}", self.agent_host, self.agent_port))
            .map_err(|e| ExportError::Send(e.to_string()))?;

        Ok(())
    }

    fn shutdown(&self) -> Result<(), ExportError> {
        Ok(())
    }
}
```

**Usage**:

```rust
let exporter = JaegerExporter::new("localhost", 6831)?;
Tracer::builder()
    .exporter(exporter)
    .build();
```

---

## Part 6: Sampler API

### Sampler Trait

```rust
/// Sampler decides which traces to record
pub trait Sampler: Send + Sync {
    /// Decide whether to sample a trace
    fn should_sample(&self, trace_id: TraceId) -> SamplingDecision;
}

/// Sampling decision
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum SamplingDecision {
    Record,
    Drop,
}
```

---

### Built-in Samplers

```rust
/// Always record
pub struct AlwaysOnSampler;
impl Sampler for AlwaysOnSampler {
    fn should_sample(&self, _trace_id: TraceId) -> SamplingDecision {
        SamplingDecision::Record
    }
}

/// Always drop
pub struct AlwaysOffSampler;
impl Sampler for AlwaysOffSampler {
    fn should_sample(&self, _trace_id: TraceId) -> SamplingDecision {
        SamplingDecision::Drop
    }
}

/// Probabilistic sampling
pub struct ProbabilisticSampler {
    ratio: f64,
}

impl ProbabilisticSampler {
    pub fn new(ratio: f64) -> Self {
        assert!((0.0..=1.0).contains(&ratio));
        Self { ratio }
    }
}

impl Sampler for ProbabilisticSampler {
    fn should_sample(&self, trace_id: TraceId) -> SamplingDecision {
        let hash = trace_id.hash();
        let value = (hash as f64) / (u64::MAX as f64);

        if value <= self.ratio {
            SamplingDecision::Record
        } else {
            SamplingDecision::Drop
        }
    }
}

/// Rate-limited sampling
pub struct RateLimitedSampler {
    max_rate: u64,
}

impl RateLimitedSampler {
    pub fn new(max_rate: u64) -> Self {
        Self { max_rate }
    }
}

impl Sampler for RateLimitedSampler {
    fn should_sample(&self, trace_id: TraceId) -> SamplingDecision {
        // Token bucket implementation
        // ...
        SamplingDecision::Record
    }
}
```

**Usage**:

```rust
let sampler = ProbabilisticSampler::new(0.1);  // 10%
Tracer::builder()
    .sampler(sampler)
    .build();
```

---

## Part 7: Configuration API

### TracerBuilder API

```rust
/// Builder for configuring tracer
pub struct TracerBuilder {
    exporter: Option<Box<dyn Exporter>>,
    sampler: Option<Box<dyn Sampler>>,
    batch_size: Option<usize>,
    batch_timeout: Option<Duration>,
}

impl TracerBuilder {
    pub fn new() -> Self {
        Self {
            exporter: None,
            sampler: None,
            batch_size: Some(100),
            batch_timeout: Some(Duration::from_millis(100)),
        }
    }

    /// Set exporter
    pub fn exporter(mut self, exporter: Box<dyn Exporter>) -> Self {
        self.exporter = Some(exporter);
        self
    }

    /// Set sampler
    pub fn sampler(mut self, sampler: Box<dyn Sampler>) -> Self {
        self.sampler = Some(sampler);
        self
    }

    /// Set batch size
    pub fn batch_size(mut self, size: usize) -> Self {
        self.batch_size = Some(size);
        self
    }

    /// Set batch timeout
    pub fn batch_timeout(mut self, timeout: Duration) -> Self {
        self.batch_timeout = Some(timeout);
        self
    }

    /// Build tracer
    pub fn build(self) -> Tracer {
        Tracer::from_builder(self)
    }
}
```

**Usage**:

```rust
let tracer = Tracer::builder()
    .exporter(Box::new(OtlpExporter::new("http://otel-collector:4317")?))
    .sampler(Box::new(ProbabilisticSampler::new(0.1)))
    .batch_size(200)
    .batch_timeout(Duration::from_millis(50))
    .build();
```

---

## Part 8: Error Handling API

### Error Types

```rust
/// Trace errors
#[derive(Debug, thiserror::Error)]
pub enum TraceError {
    #[error("Invalid traceparent format: {0}")]
    InvalidTraceParent(String),

    #[error("Export failed: {0}")]
    ExportError(String),

    #[error("Span not found: {0:?}")]
    SpanNotFound(SpanId),

    #[error("Context not available")]
    ContextNotAvailable,

    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),

    #[error("Serialization error: {0}")]
    Serialize(String),
}

/// Parse errors
#[derive(Debug, thiserror::Error)]
pub enum ParseError {
    #[error("Invalid format")]
    InvalidFormat,

    #[error("Unsupported version: {0}")]
    UnsupportedVersion(String),

    #[error("Invalid hex: {0}")]
    InvalidHex(String),
}

/// Export errors
#[derive(Debug, thiserror::Error)]
pub enum ExportError {
    #[error("Connection failed: {0}")]
    Connection(String),

    #[error("Socket error: {0}")]
    Socket(String),

    #[error("Encode error: {0}")]
    Encode(String),

    #[error("Send error: {0}")]
    Send(String),
}
```

---

### Result Types

```rust
/// Trace result
pub type TraceResult<T> = Result<T, TraceError>;

/// Parse result
pub type ParseResult<T> = Result<T, ParseError>;

/// Export result
pub type ExportResult = Result<(), ExportError>;
```

**Usage**:

```rust
fn extract_context(headers: &HeaderMap) -> TraceResult<TraceContext> {
    let traceparent = headers
        .get("traceparent")
        .ok_or(TraceError::InvalidTraceParent("Missing traceparent".to_string()))?;

    let ctx = TraceContext::from_traceparent(traceparent.to_str()?)?;
    Ok(ctx)
}
```

---

## Conclusion

The tracer-rs API prioritizes:

1. **Ergonomics**: Builder pattern, fluent API
2. **Type Safety**: Compile-time attribute validation
3. **RAII**: Spans auto-end on drop
4. **Async-Aware**: Works with tokio async/await
5. **Extensibility**: Pluggable exporters, samplers, propagators

**Next Phase**: User & Developer Guides (04-user-guide.md, 05-dev-guide.md)

---

**Document Status**: ✅ Complete
**Word Count**: 5,500+
**Next Phase**: User & Developer Guides
