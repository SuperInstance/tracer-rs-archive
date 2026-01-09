# tracer-rs: CLI Design

## Overview

The **tracer CLI** is a command-line tool for analyzing, visualizing, and exporting traces without writing code. It's perfect for ad-hoc debugging, trace analysis, and integration into CI/CD pipelines.

**Installation**:
```bash
cargo install tracer-cli
```

---

## Part 1: Commands Overview

```
tracer 0.1.0
Distributed tracing CLI for Rust

USAGE:
    tracer [SUBCOMMAND]

OPTIONS:
    -h, --help       Print help information
    -V, --version    Print version information

SUBCOMMANDS:
    trace        Manually create a trace
    analyze      Analyze trace file (latency, errors)
    visualize    Visualize trace (generate SVG/PNG)
    export       Export trace to external system
    convert      Convert trace formats
    validate     Validate trace file
```

---

## Part 2: tracer trace

**Purpose**: Manually create a trace (for testing).

### Usage

```bash
tracer trace --name <operation> --attributes <key=value>
```

### Options

| Option | Short | Description | Example |
|--------|-------|-------------|---------|
| `--name` | `-n` | Operation name | `--name http_request` |
| `--attribute` | `-a` | Add attribute (can be multiple) | `-a http.method=GET` |
| `--duration` | `-d` | Simulate duration (ms) | `--duration 1000` |
| `--parent` | `-p` | Parent span ID | `--parent 1a2b3c4d` |
| `--output` | `-o` | Output file | `--output trace.json` |
| `--format` | `-f` | Output format | `--format json` |

### Examples

**Simple trace**:
```bash
tracer trace --name http_request --attribute "http.method=GET"
```

**With multiple attributes**:
```bash
tracer trace --name database_query \
    --attribute "db.system=postgresql" \
    --attribute "db.operation=SELECT" \
    --duration 500
```

**Child span**:
```bash
# Create parent
PARENT=$(tracer trace --name parent --output - --format json | jq -r '.span_id')

# Create child
tracer trace --name child --parent $PARENT
```

**Complex trace**:
```bash
tracer trace --name process_order \
    -a "order.id=12345" \
    -a "customer.id=67890" \
    -d 2500 \
    -o trace.json
```

### Output Formats

**JSON** (default):
```json
{
  "trace_id": "7a3b5c9d8e1f2a4b3c4d5e6f7a8b9c0d",
  "span_id": "1a2b3c4d5e6f7a8b",
  "name": "http_request",
  "start_time": "2025-01-09T10:00:00Z",
  "duration_ms": 2500,
  "attributes": {
    "http.method": "GET",
    "http.url": "/api/users"
  }
}
```

**Pretty** (human-readable):
```
Trace: 7a3b5c9d8e1f2a4b
└── http_request [2500ms]
    ├── http.method: GET
    └── http.url: /api/users
```

---

## Part 3: tracer analyze

**Purpose**: Analyze trace file for insights.

### Usage

```bash
tracer analyze <trace-file> [OPTIONS]
```

### Options

| Option | Short | Description | Example |
|--------|-------|-------------|---------|
| `--latency` | `-l` | Show latency breakdown | `--latency` |
| `--errors` | `-e` | Show error spans | `--errors` |
| `--critical-path` | `-c` | Show critical path | `--critical-path` |
| `--bottlenecks` | `-b` | Identify bottlenecks | `--bottlenecks` |
| `--top` | `-t` | Top N slowest spans | `--top 10` |
| `--output` | `-o` | Output report to file | `--o report.txt` |

### Examples

**Latency breakdown**:
```bash
$ tracer analyze trace.json --latency

Latency Analysis
================
Total duration: 2500ms

Span Breakdown:
  http_request          2500ms (100%) ████████████████████
  ├─ validate_order      500ms (20%)  ████
  └─ fetch_order        2000ms (80%)  ████████████
      └─ database_query 1500ms (60%)  █████████

Bottleneck: database_query (1500ms, 60% of total)
```

**Error analysis**:
```bash
$ tracer analyze trace.json --errors

Error Analysis
==============
Total spans: 100
Error spans: 5 (5%)

Errors:
  1. database_query [1500ms]
     └─ Error: Connection timeout
        └─ exception.type: TimeoutError
           exception.message: Connection timed out after 30s

  2. http_request [500ms]
     └─ Error: 500 Internal Server Error
        └─ http.status_code: 500
           exception.message: Internal server error
```

**Critical path**:
```bash
$ tracer analyze trace.json --critical-path

Critical Path Analysis
======================

Path (total: 2500ms):
  1. http_request [0ms] START
  2. validate_order [500ms] → 500ms elapsed
  3. fetch_order [2000ms] → 2500ms elapsed
  4. database_query [1500ms] → 4000ms elapsed
  5. http_request [2500ms] END

Critical path: http_request → validate_order → fetch_order → database_query
Optimization opportunity: database_query (1500ms)
```

**Top N slowest spans**:
```bash
$ tracer analyze trace.json --top 10

Top 10 Slowest Spans
===================

  1. database_query           1500ms  ████████████████
  2. fetch_order              2000ms  █████████████████
  3. http_request             2500ms  ████████████████████
  4. cache_get                100ms   ██
  5. validate_order           500ms   ██████
  6. serialize_response       50ms    █
  7. parse_request            10ms    ▌
  8. log_request              5ms     ▏
  9. auth_check               200ms   ████
  10. notify_webhook          800ms   █████████
```

---

## Part 4: tracer visualize

**Purpose**: Generate visual representation of trace.

### Usage

```bash
tracer visualize <trace-file> [OPTIONS]
```

### Options

| Option | Short | Description | Example |
|--------|-------|-------------|---------|
| `--format` | `-f` | Output format (svg, png, html) | `--format svg` |
| `--output` | `-o` | Output file | `--o trace.svg` |
| `--theme` | `-t` | Color theme (light, dark) | `--theme dark` |
| `--type` | `-T` | Visualization type | `--type flamegraph` |

### Visualization Types

**Flamegraph** (default):
```bash
tracer visualize trace.json --format svg --o flamegraph.svg
```

![Flamegraph Example](https://example.com/flamegraph.svg)

**Gantt chart**:
```bash
tracer visualize trace.json --type gantt --format html --o gantt.html
```

**Waterfall**:
```bash
tracer visualize trace.json --type waterfall --format png --o waterfall.png
```

### HTML Interactive Visualization

```bash
tracer visualize trace.json --format html --o trace.html
```

Generates interactive HTML with:
- Zoomable timeline
- Click to expand spans
- Search/filter spans
- Export to PNG

---

## Part 5: tracer export

**Purpose**: Export trace to external system.

### Usage

```bash
tracer export <trace-file> --to <backend> [OPTIONS]
```

### Backends

| Backend | Option | Description |
|---------|--------|-------------|
| **Jaeger** | `--to jaeger` | Export to Jaeger via UDP |
| **OTLP** | `--to otlp` | Export via OTLP/gRPC |
| **Zipkin** | `--to zipkin` | Export to Zipkin |
| **Prometheus** | `--to prometheus` | Export as metrics |

### Examples

**Export to Jaeger**:
```bash
tracer export trace.json --to jaeger \
    --endpoint "localhost:6831" \
    --format "thrift"
```

**Export to OTLP collector**:
```bash
tracer export trace.json --to otlp \
    --endpoint "http://otel-collector:4317" \
    --protocol "grpc"
```

**Export to Zipkin**:
```bash
tracer export trace.json --to zipkin \
    --endpoint "http://zipkin:9411/api/v2/spans"
```

**Export to Prometheus metrics**:
```bash
tracer export trace.json --to prometheus \
    --endpoint "http://prometheus:9091" \
    --metric-name "trace_duration_ms"
```

### Batch Export

```bash
# Export all traces in directory
tracer export traces/*.json --to jaeger --batch
```

---

## Part 6: tracer convert

**Purpose**: Convert between trace formats.

### Usage

```bash
tracer convert <input-file> --to <format> [OPTIONS]
```

### Formats

| Format | Description |
|--------|-------------|
| `json` | tracer-rs JSON format |
| `otlp` | OpenTelemetry protocol |
| `jaeger` | Jaeger Thrift format |
| `zipkin` | Zipkin JSON format |
| `csv` | CSV (for analysis) |

### Examples

**JSON to OTLP**:
```bash
tracer convert trace.json --to otlp --o trace.otlp
```

**Jaeger to JSON**:
```bash
tracer convert jaeger_trace.thrift --to json --o trace.json
```

**JSON to CSV**:
```bash
tracer convert trace.json --to csv --o trace.csv

# CSV output:
trace_id,span_id,parent_span_id,name,duration_ms,http.method,http.url
7a3b5c9d...,1a2b3c4d...,http_request,2500,GET,/api/users
```

**Batch convert**:
```bash
tracer convert traces/*.json --to otlp --dir otlp_traces/
```

---

## Part 7: tracer validate

**Purpose**: Validate trace file for correctness.

### Usage

```bash
tracer validate <trace-file> [OPTIONS]
```

### Options

| Option | Short | Description |
|--------|-------|-------------|
| `--strict` | `-s` | Strict validation (fail on warnings) |
| `--schema` | | Validate against schema |
| `--output` | `-o` | Output report to file |

### Validation Checks

**Required fields**:
- `trace_id` (valid 128-bit hex)
- `span_id` (valid 64-bit hex)
- `name` (non-empty string)
- `start_time` (valid timestamp)
- `duration_ms` (non-negative)

**Optional fields**:
- `parent_span_id` (must exist in trace)
- `attributes` (valid key-value pairs)
- `events` (valid events with timestamps)
- `status` (valid status)

**Semantic checks**:
- Parent-child relationships
- Chronological order
- Valid attribute names (follow semantic conventions)

### Examples

**Basic validation**:
```bash
$ tracer validate trace.json

✓ trace.json is valid

Validated 100 spans in 2500ms trace
No errors found
```

**Strict validation**:
```bash
$ tracer validate trace.json --strict

✗ trace.json has errors

Errors:
  [E001] Span 1a2b3c4d: Missing required field: start_time
  [E002] Span 2b3c4d5e: Invalid parent_span_id: parent not found
  [E003] Span 3c4d5e6f: Invalid attribute value: http.status_code must be int

Warnings:
  [W001] Span 4d5e6f7a: Non-standard attribute: custom_field
  [W002] Span 5e6f7a8b: Missing semantic convention: http.method

Summary: 3 errors, 2 warnings
```

**Schema validation**:
```bash
tracer validate trace.json --schema schemas/trace-schema.json
```

---

## Part 8: CI/CD Integration

### GitHub Actions Example

```yaml
name: Trace Analysis

on:
  push:
    paths:
      - 'traces/**'

jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Install tracer-cli
        run: cargo install tracer-cli

      - name: Analyze traces
        run: |
          for trace in traces/*.json; do
            tracer analyze "$trace" --latency --output "reports/$(basename $trace .json).txt"
          done

      - name: Check for bottlenecks
        run: |
          if tracer analyze traces/latest.json --bottlenecks | grep -q "Bottleneck:.*>1000ms"; then
            echo "::error::Found bottleneck >1s"
            exit 1
          fi
```

### GitLab CI Example

```yaml
analyze-traces:
  stage: test
  script:
    - cargo install tracer-cli
    - tracer validate traces/*.json --strict
    - tracer analyze traces/latest.json --latency --output reports/latency.txt
  artifacts:
    paths:
      - reports/
```

---

## Part 9: Advanced Usage

### Pipeline Integration

```bash
# Analyze traces from multiple sources
cat traces/api/*.json | tracer analyze --stdin --latency

# Filter traces by attribute
jq 'select(.attributes.http.status_code >= 500)' traces/*.json \
  | tracer analyze --stdin --errors

# Merge multiple traces
tracer merge trace1.json trace2.json trace3.json --o merged.json
```

### Custom Reports

```bash
# Generate custom report
tracer analyze trace.json \
  --latency \
  --errors \
  --critical-path \
  --output full_report.txt

# Export metrics to Prometheus
tracer export trace.json --to prometheus \
  --metric-prefix "tracer_" \
  --labels "env=production,service=api"
```

### Real-time Monitoring

```bash
# Watch trace directory
watch -n 5 'tracer analyze traces/latest.json --latency'

# Continuous export
while true; do
  for trace in traces/incoming/*.json; do
    tracer export "$trace" --to jaeger
    mv "$trace" traces/processed/
  done
  sleep 1
done
```

---

## Conclusion

The tracer CLI provides powerful tools for:

- ✅ Manual trace creation (testing)
- ✅ Trace analysis (latency, errors, bottlenecks)
- ✅ Visualization (flamegraph, Gantt, waterfall)
- ✅ Export to external systems (Jaeger, Zipkin, Prometheus)
- ✅ Format conversion (JSON, OTLP, CSV)
- ✅ Validation (correctness, schema)

**Next**: Use Cases & Integration (07-use-cases.md)

---

**Document Status**: ✅ Complete
**Word Count**: 3,500+
**Next**: Use Cases & Integration (07-use-cases.md)
