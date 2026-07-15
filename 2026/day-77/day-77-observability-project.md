# Day 77: Full Stack Observability with Docker Compose

**Project**: End-to-end observability stack with Prometheus, Grafana, Loki, OpenTelemetry Collector, and alerting.

**Duration**: Completed as capstone of 5-day observability block (Days 73-77)

**Status**: ✅ All 8 services running, metrics flowing, logs aggregated, traces collected, unified dashboard deployed

---

## Architecture Overview

### System Diagram: 8-Service Observability Stack

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Observability Stack (Docker Compose)                │
└─────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────┐
│                            DATA COLLECTION LAYER                            │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────┐  ┌──────────────────┐  ┌──────────────────────┐  │
│  │   Node Exporter     │  │     cAdvisor     │  │  Promtail (Logs)     │  │
│  │   (Port 9100)       │  │   (Port 8080)    │  │  (Port 9080 internal)│  │
│  │ - Host CPU/Mem/Disk │  │ - Container CPU  │  │ - Docker logs        │  │
│  │ - Network I/O       │  │ - Memory usage   │  │ - Log shipping       │  │
│  └─────────────────────┘  │ - Network stats  │  └──────────────────────┘  │
│                           └──────────────────┘                             │
│                                                                              │
│  ┌──────────────────────────┐    ┌─────────────────────────────────────┐  │
│  │   Notes App (Django)     │    │  OTEL Collector (Traces)            │  │
│  │   (Port 8000)            │    │  (Ports 4317/4318 - OTLP)           │  │
│  │ - REST API               │    │  - Receives span data               │  │
│  │ - Generates traffic      │    │  - Correlates traces                │  │
│  └──────────────────────────┘    └─────────────────────────────────────┘  │
│                                                                              │
└────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────┐
│                         AGGREGATION & STORAGE LAYER                         │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────────┐           ┌──────────────────────────────────┐  │
│  │    Prometheus        │           │         Loki                     │  │
│  │   (Port 9090)        │           │    (Port 3100)                   │  │
│  │ - Scrapes metrics    │           │ - Ingests logs from Promtail     │  │
│  │   every 15s          │           │ - Stores in Chunkstore           │  │
│  │ - 15-day retention   │           │ - LogQL query engine             │  │
│  │ - Alert evaluation   │           │ - Correlates with metrics        │  │
│  └──────────────────────┘           └──────────────────────────────────┘  │
│                                                                              │
└────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────┐
│                         VISUALIZATION & QUERYING LAYER                      │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                          Grafana                                     │  │
│  │                       (Port 3000)                                    │  │
│  ├──────────────────────────────────────────────────────────────────────┤  │
│  │ Datasources: Prometheus, Loki, OTEL (debug)                         │  │
│  │ Dashboards:                                                          │  │
│  │   • Production Overview (unified metrics + logs)                     │  │
│  │   • System Health (CPU, Memory, Disk)                               │  │
│  │   • Container Metrics (per-container resource usage)                │  │
│  │   • Application Logs (notes-app)                                    │  │
│  │   • Trace Debugging (OTEL Collector output)                         │  │
│  │                                                                      │  │
│  │ Features:                                                            │  │
│  │   • Auto-provisioned datasources and dashboards                     │  │
│  │   • Template variables for dynamic filtering                        │  │
│  │   • 10-second auto-refresh, 30-minute time range                    │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└────────────────────────────────────────────────────────────────────────────┘

                              DATA FLOW DIRECTIONS
                                    ↓
  Node Exporter ──→┐
                   ├──→ Prometheus ─→ Grafana
  cAdvisor ────────┤               ↗ (Dashboards)
                   ├──→ Alert Rules
  Notes App ───────┤
                   ├──→ Promtail ──→ Loki ───→┐
  Promtail ────────┘                         ├→ Grafana
                                             ↙ (Logs/Explore)
  OTEL Collector ──────────────────────────→ Grafana (Traces Debug)
```

---

## Deployment Checklist

### ✅ Pre-Deployment

- [x] Reference repository cloned: `https://github.com/LondheShubham153/observability-for-devops.git`
- [x] Docker and Docker Compose installed (v2.0+)
- [x] 8 GB RAM and 10 GB disk space available
- [x] All ports available: 3000, 3100, 4317, 4318, 8000, 8080, 9080, 9090, 9100

### ✅ Stack Initialization

```bash
git clone https://github.com/LondheShubham153/observability-for-devops.git
cd observability-for-devops
docker compose up -d
docker compose ps  # Verify all 8 services Running
```

### ✅ Service Health Validation

| Service | Port | Status | Check Command |
|---------|------|--------|---------------|
| Prometheus | 9090 | ✅ UP | `curl http://localhost:9090/-/healthy` |
| Node Exporter | 9100 | ✅ UP | `curl http://localhost:9100/metrics \| head -5` |
| cAdvisor | 8080 | ✅ UP | `curl http://localhost:8080/api/v1/machines` |
| Grafana | 3000 | ✅ UP | `curl http://localhost:3000/api/health` |
| Loki | 3100 | ✅ UP | `curl http://localhost:3100/ready` |
| Promtail | 9080 | ✅ UP | `docker logs promtail \| tail -5` |
| OTEL Collector | 4317/4318 | ✅ UP | `docker logs otel-collector \| grep "Start receiving" ` |
| Notes App | 8000 | ✅ UP | `curl http://localhost:8000` |

---

## Validation: Metrics Pipeline

### Prometheus Targets Status

**Access**: http://localhost:9090/targets

All 4 scrape jobs should show **UP (green)**:

```
✅ prometheus (1/1 up)           - Self-monitoring
✅ node-exporter (1/1 up)        - Host system metrics
✅ docker/cadvisor (1/1 up)      - Container metrics
✅ otel-collector (1/1 up)       - OTLP metrics receiver
```

### Sample PromQL Validation Queries

Run these in http://localhost:9090 > Graph tab:

#### Query 1: All Targets Healthy
```promql
up
```
**Expected**: Time series showing value of 1 for all 4 scrapers. If any returns 0, that job is DOWN.

#### Query 2: Host CPU Usage
```promql
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```
**Expected**: Single gauge value between 0-100%, representing current CPU utilization.

#### Query 3: Memory Usage
```promql
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
```
**Expected**: Single gauge value between 0-100%, representing current memory utilization.

#### Query 4: Container CPU Per Container
```promql
rate(container_cpu_usage_seconds_total{name!=""}[5m]) * 100
```
**Expected**: Time series lines for each running container (notes-app, prometheus, grafana, etc.).

#### Query 5: Top 3 Memory-Hungry Containers
```promql
topk(3, container_memory_usage_bytes{name!=""})
```
**Expected**: 3 container names with memory usage in bytes. Grafana/Prometheus typically consume 200-500 MB.

---

## Validation: Logs Pipeline

### Generate Application Traffic

```bash
# 50 concurrent requests to notes-app
for i in $(seq 1 50); do
  curl -s http://localhost:8000 > /dev/null
  curl -s http://localhost:8000/api/ > /dev/null
  sleep 0.1
done
```

### Loki Explore Queries

**Access**: http://localhost:3000 > Explore > Loki datasource

#### Query 1: All Container Logs
```logql
{job="docker"}
```
**Expected**: Logs from all containers (prometheus, grafana, notes-app, etc.) with labels `container_name`, `job`.

#### Query 2: Notes App Only
```logql
{container_name="notes-app"}
```
**Expected**: Only Django app logs showing HTTP requests, database queries.

#### Query 3: Error Logs Across All Containers
```logql
{job="docker"} |= "error"
```
**Expected**: Filtered logs containing the word "error" (typically few to none in healthy stack).

#### Query 4: HTTP GET Requests from Notes App
```logql
{container_name="notes-app"} |= "GET"
```
**Expected**: HTTP request logs from Django app showing GET requests to `/api/`, `/api/notes/`.

#### Query 5: Log Volume Per Container
```logql
sum by (container_name) (rate({job="docker"}[5m]))
```
**Expected**: Time series showing log ingestion rate (lines/sec) per container. Typically 0.1-1 lines/sec during idle, spikes during traffic.

### Promtail Targets

```bash
curl -s http://localhost:9080/targets | jq '.[] | {job: .job, path: .path, state: .state}' | head -20
```

**Expected Output**:
```json
{
  "job": "docker",
  "path": "/var/lib/docker/containers/*/logs/*/std*.log",
  "state": "running"
}
```

---

## Validation: Traces Pipeline

### Send Test OTLP Trace

```bash
curl -X POST http://localhost:4317/v1/traces \
  -H "Content-Type: application/json" \
  -d '{
    "resourceSpans": [{
      "resource": {
        "attributes": [{
          "key": "service.name",
          "value": { "stringValue": "notes-app" }
        }]
      },
      "scopeSpans": [{
        "spans": [{
          "traceId": "aaaabbbbccccdddd1111222233334444",
          "spanId": "1111222233334444",
          "name": "GET /api/notes",
          "kind": 2,
          "startTimeUnixNano": "1700000000000000000",
          "endTimeUnixNano": "1700000000150000000",
          "attributes": [{
            "key": "http.method",
            "value": { "stringValue": "GET" }
          },
          {
            "key": "http.route",
            "value": { "stringValue": "/api/notes" }
          },
          {
            "key": "http.status_code",
            "value": { "intValue": 200 }
          }],
          "status": { "code": 1 }
        }]
      }]
    }]
  }'
```

### Verify Trace Reception

```bash
docker logs otel-collector 2>&1 | grep -A 10 "GET /api/notes"
```

**Expected Output**:
```
2024-XX-XXT00:00:00.000Z	info	exporters/logging_exporter.go:XX	ResourceSpans #0
Resource labels (key value pairs):
     -> service.name: String(notes-app)
ScopeSpans #0
Span #0
    Trace ID       : aaaabbbbccccdddd1111222233334444
    Parent ID      : 
    ID             : 1111222233334444
    Name           : GET /api/notes
    Kind           : SPAN_KIND_SERVER
    Start time     : 1700000000000000000
    End time       : 1700000000150000000
    Status code    : STATUS_CODE_OK
Attributes:
     -> http.method: String(GET)
     -> http.route: String(/api/notes)
     -> http.status_code: Int(200)
```

---

## Production Overview Dashboard

### Dashboard Configuration

**Name**: Production Overview -- Observability Stack  
**Refresh**: Every 10 seconds (auto-refresh enabled)  
**Time Range**: Last 30 minutes  
**Time Zone**: Browser local time

### Dashboard Layout & Panels

#### Row 1: System Health (Node Exporter + Prometheus)

| Panel | Type | Query | Unit | Thresholds |
|-------|------|-------|------|------------|
| **CPU Usage** | Gauge | `100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)` | percent | Green: <50%, Yellow: 50-80%, Red: >80% |
| **Memory Usage** | Gauge | `(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100` | percent | Green: <60%, Yellow: 60-80%, Red: >80% |
| **Disk Usage** | Gauge | `(1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100` | percent | Green: <70%, Yellow: 70-85%, Red: >85% |
| **Targets UP** | Stat | `sum(up) / count(up)` | none (ratio) | Green: 1.0, Red: <1.0 |

#### Row 2: Container Metrics (cAdvisor)

| Panel | Type | Query | Legend | Format |
|-------|------|-------|--------|--------|
| **Container CPU** | Time Series | `rate(container_cpu_usage_seconds_total{name!=""}[5m]) * 100` | `{{name}}` | Percent (0-100) |
| **Container Memory** | Bar Chart | `container_memory_usage_bytes{name!=""} / 1024 / 1024` | `{{name}}` | Megabytes (MB) |
| **Container Count** | Stat | `count(container_last_seen{name!=""})` | - | Number of active containers |

#### Row 3: Application Logs (Loki)

| Panel | Type | Query (Loki) | Options |
|-------|------|------|---------|
| **App Logs** | Logs | `{container_name="notes-app"}` | Show timestamp, labels: container_name, pod |
| **Error Rate** | Time Series | `sum(rate({job="docker"} \|= "error" [5m]))` | Line chart, legend: "Errors/sec" |
| **Log Volume** | Time Series | `sum by (container_name) (rate({job="docker"}[5m]))` | Stacked area, legend: `{{container_name}}` |

#### Row 4: Service Overview

| Panel | Type | Query | Notes |
|-------|------|-------|-------|
| **Prometheus Scrape Duration** | Time Series | `prometheus_target_interval_length_seconds{quantile="0.99"}` | 99th percentile scrape time in seconds |
| **OTEL Metrics Received** | Stat | `otelcol_receiver_accepted_metric_points` (if available) | Count of metrics accepted by collector |
| **Alertmanager Status** | Stat | `alertmanager_config_last_reload_successful` (after adding alerting) | 1 = OK, 0 = Failed |

### Dashboard Export

To save dashboard as code:

```bash
# In Grafana: Share → Export → Select "Save externally"
# Dashboard JSON is exported for version control

# To re-import:
# Dashboards → New → Import → Paste JSON
```

---

## Configuration Comparison: Build vs Reference

### Day 73-74: Prometheus

**Your config** (days 73-74):
```yaml
# prometheus.yml (built from scratch)
global:
  scrape_interval: 15s
  evaluation_interval: 15s
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['localhost:9100']
```

**Reference config**:
```yaml
# observability-for-devops/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'observability-stack'
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'node-exporter'
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
    relabel_configs:
      - source_labels: [__meta_docker_container_name]
        target_label: instance
  - job_name: 'docker'
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
  - job_name: 'otel-collector'
    static_configs:
      - targets: ['otel-collector:8888']
```

**Differences**:
| Aspect | Your Version | Reference |
|--------|---|---|
| External Labels | ❌ None | ✅ cluster="observability-stack" |
| Service Discovery | ❌ Static IPs | ✅ Docker SD (auto-discovers containers) |
| Job Count | 2 jobs | 4 jobs (adds cAdvisor, OTEL) |
| Scalability | Limited | Scales: new containers auto-discovered |

**Key Insight**: Reference uses Docker SD for auto-discovery. Your manual config works but doesn't scale. In production, use service discovery.

---

### Day 75: Loki Configuration

**Your config** (day 75):
```yaml
# loki-config.yml (built from scratch)
auth_enabled: false
server:
  http_listen_port: 3100
ingester:
  max_chunk_age: 2h
  chunk_retain_period: 0
  chunk_idle_period: 5m
schema_config:
  configs:
    - from: 2024-01-01
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
```

**Reference config**:
```yaml
# observability-for-devops/loki/loki-config.yml
auth_enabled: false
server:
  http_listen_port: 3100
  log_level: info
ingester:
  max_chunk_age: 3h
  chunk_idle_period: 10m
  chunk_retain_period: 1m
limits_config:
  ingestion_rate_mb: 10
  ingestion_burst_size_mb: 20
  retention_period: 720h  # 30 days
schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h
storage_config:
  filesystem:
    directory: /loki/chunks
```

**Differences**:
| Aspect | Your Version | Reference |
|--------|---|---|
| Log Level | ❌ Not set | ✅ info |
| Limits | ❌ Unlimited | ✅ 10 MB/s ingestion, burst 20 MB |
| Retention | ❌ Not set | ✅ 720 hours (30 days) |
| Store | boltdb-shipper (deprecated) | ✅ tsdb (newer) |
| Schema | v11 | ✅ v13 (latest) |
| Index Period | N/A | ✅ 24 hours (better query performance) |

**Key Insight**: Reference includes rate limiting and retention policies—critical for preventing disk bloat. Your version would grow unbounded.

---

### Day 75: Promtail Configuration

**Your config** (day 75):
```yaml
# promtail-config.yml (built from scratch)
server:
  http_listen_port: 9080
clients:
  - url: http://localhost:3100/loki/api/v1/push
scrape_configs:
  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
    relabel_configs:
      - source_labels: [__meta_docker_container_name]
        target_label: container_name
```

**Reference config**:
```yaml
# observability-for-devops/promtail/promtail-config.yml
server:
  http_listen_port: 9080
  log_level: info
clients:
  - url: http://loki:3100/loki/api/v1/push
    batching_config:
      batch_size_bytes: 1048576
      batch_timeout: 5s
      max_retries: 3
positions:
  filename: /tmp/positions.yaml
scrape_configs:
  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
    relabel_configs:
      - source_labels: [__meta_docker_container_name]
        target_label: container_name
      - source_labels: [__meta_docker_container_id]
        target_label: container_id
      - source_labels: [__meta_docker_container_network_mode]
        target_label: network_mode
    pipeline_stages:
      - docker: {}
      - multiline:
          line_start_pattern: '^\d{4}-\d{2}-\d{2}'
```

**Differences**:
| Aspect | Your Version | Reference |
|---|---|---|
| Retry Logic | ❌ No batching | ✅ Batches + 3 retries |
| Positions File | ❌ Not defined | ✅ /tmp/positions.yaml (tracks offset) |
| Pipeline Stages | ❌ None | ✅ Docker parsing + multiline handling |
| Service Name | localhost:3100 | ✅ loki:3100 (Docker DNS) |
| Labels Extracted | 1 label | ✅ 3 labels (container_id, network_mode) |

**Key Insight**: Reference includes batching (efficiency), pipeline stages (parsing), and position tracking (no duplicate logs). Your version works but loses logs during network issues.

---

### Day 76: OTEL Collector Configuration

**Your config** (day 76):
```yaml
# otel-collector-config.yml (built from scratch)
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
exporters:
  logging:
    loglevel: debug
service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [logging]
```

**Reference config**:
```yaml
# observability-for-devops/otel-collector/otel-collector-config.yml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
  prometheus:
    config:
      scrape_configs:
        - job_name: 'otel-collector'
          static_configs:
            - targets: ['localhost:8888']
processors:
  batch:
    send_batch_size: 100
    timeout: 10s
  memory_limiter:
    check_interval: 1s
    limit_mib: 512
exporters:
  logging:
    loglevel: debug
  jaeger:
    endpoint: jaeger:14250
extensions:
  health_check:
    endpoint: 0.0.0.0:13133
service:
  extensions: [health_check]
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch, memory_limiter]
      exporters: [logging, jaeger]
    metrics:
      receivers: [prometheus]
      exporters: [logging]
```

**Differences**:
| Aspect | Your Version | Reference |
|---|---|---|
| Processors | ❌ None | ✅ Batch (throughput) + Memory limiter (stability) |
| Trace Export | logging only | ✅ logging + Jaeger (production trace storage) |
| Metrics Pipeline | ❌ Not defined | ✅ Scrapes collector metrics, exports to Prometheus |
| Health Check | ❌ None | ✅ Endpoint 13133 (k8s readiness probe) |
| Memory Safety | Unbounded | ✅ 512 MB limit + GC pressure |

**Key Insight**: Reference handles production concerns: memory bounds, multi-destination export, health checks. Your version is debug-only; production needs Jaeger or similar.

---

### Day 74: Grafana Datasources (Provisioning)

**Your config** (day 74):
```yaml
# grafana/provisioning/datasources/datasources.yml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://localhost:9090
    isDefault: true
```

**Reference config**:
```yaml
# observability-for-devops/grafana/provisioning/datasources/datasources.yml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
  - name: Jaeger
    type: jaeger
    access: proxy
    url: http://jaeger:16686
```

**Differences**:
| Aspect | Your Version | Reference |
|---|---|---|
| Datasources | 1 (Prometheus) | 3 (Prometheus, Loki, Jaeger) |
| Service Names | localhost:9090 | ✅ Container DNS (prometheus, loki) |
| Logs Support | ❌ No | ✅ Loki |
| Trace Support | ❌ No | ✅ Jaeger |

**Key Insight**: Reference is a complete stack; yours was building incrementally. Auto-provisioning is production-grade: UI changes don't overwrite configs.

---

### Days 73-76: Docker Compose

**Your compose** (built incrementally):
```yaml
version: '3.8'
services:
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
  node-exporter:
    image: prom/node-exporter
    ports:
      - "9100:9100"
  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
```

**Reference compose**:
```yaml
version: '3.8'
networks:
  monitoring:
    driver: bridge
services:
  prometheus:
    image: prom/prometheus
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - monitoring
    depends_on:
      - node-exporter
  
  node-exporter:
    image: prom/node-exporter
    container_name: node-exporter
    restart: unless-stopped
    ports:
      - "9100:9100"
    networks:
      - monitoring
  
  cadvisor:
    image: gcr.io/cadvisor/cadvisor
    container_name: cadvisor
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    ports:
      - "8080:8080"
    networks:
      - monitoring
  
  loki:
    image: grafana/loki
    container_name: loki
    volumes:
      - ./loki/loki-config.yml:/etc/loki/local-config.yaml
      - loki_data:/loki
    ports:
      - "3100:3100"
    networks:
      - monitoring
  
  promtail:
    image: grafana/promtail
    container_name: promtail
    volumes:
      - ./promtail/promtail-config.yml:/etc/promtail/config.yml
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock
    ports:
      - "9080:9080"
    networks:
      - monitoring
    depends_on:
      - loki
  
  otel-collector:
    image: otel/opentelemetry-collector
    container_name: otel-collector
    volumes:
      - ./otel-collector/otel-collector-config.yml:/etc/otelcol/config.yaml
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
      - "8888:8888"   # metrics
    networks:
      - monitoring
  
  grafana:
    image: grafana/grafana
    container_name: grafana
    restart: unless-stopped
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    ports:
      - "3000:3000"
    networks:
      - monitoring
    depends_on:
      - prometheus
      - loki
  
  notes-app:
    build: ./notes-app
    container_name: notes-app
    ports:
      - "8000:8000"
    networks:
      - monitoring
    depends_on:
      - prometheus

volumes:
  prometheus_data:
  loki_data:
  grafana_data:

networks:
  monitoring:
    driver: bridge
```

**Differences**:
| Aspect | Your Version | Reference |
|---|---|---|
| Network | Default bridge | ✅ Custom "monitoring" network (explicit DNS) |
| Container Names | N/A | ✅ Explicit names (predictable) |
| Restart Policy | Default | ✅ unless-stopped (auto-recovery) |
| Volumes | None | ✅ Named volumes for data persistence |
| Dependencies | None | ✅ depends_on for startup order |
| cAdvisor | ❌ Missing | ✅ Included with proper mounts |
| Loki | ❌ Missing | ✅ Full stack |
| Promtail | ❌ Missing | ✅ Full stack |
| OTEL Collector | ❌ Partial | ✅ Complete with health checks |
| Notes App | N/A | ✅ Sample workload to monitor |

**Key Insight**: Reference is production-ready: restart policies, named volumes, explicit networking, dependency ordering. Your incremental build works but needs hardening.

---

## Production Readiness: What's Missing

### Critical for Production Deployment

| Component | Current | Required for Production |
|-----------|---------|------------------------|
| **Alerting** | Alert rules evaluated in Prometheus | ✅ Alertmanager + Slack/PagerDuty/Opsgenie integration |
| **Trace Storage** | Debug exporter (logs only) | ✅ Jaeger or Grafana Tempo (persistent) |
| **High Availability** | Single instance | ✅ Multiple Prometheus/Loki replicas, load balancer |
| **Authentication** | None (admin/admin default) | ✅ OAuth2, LDAP, or RBAC in Grafana; IP allowlists |
| **TLS/HTTPS** | HTTP only | ✅ Certificates, reverse proxy (nginx/traefik) |
| **Data Retention** | 15 days (Prometheus) | ✅ 30+ days, tiered storage, S3/GCS |
| **Backup** | None | ✅ Automated daily backups, tested restore |
| **Resource Limits** | Unlimited | ✅ CPU/memory requests + limits, quotas |
| **Logging** | Stdout/stderr | ✅ Structured logging (JSON), audit logs |
| **Monitoring Stack** | No meta-monitoring | ✅ Monitor the monitors (recursive) |

### Recommended Production Additions

#### 1. Alertmanager + Alert Integration

```yaml
# Add to prometheus.yml
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']
rule_files:
  - 'alert-rules.yml'

# alert-rules.yml examples
groups:
  - name: infrastructure
    rules:
      - alert: HighCPU
        expr: 'cpu_usage > 80'
        for: 5m
        annotations:
          summary: "High CPU on {{ $labels.instance }}"
```

#### 2. Trace Storage (Jaeger/Tempo)

```yaml
# Add to docker-compose.yml
jaeger:
  image: jaegertracing/all-in-one
  ports:
    - "16686:16686"  # UI
    - "14250:14250"  # OTLP gRPC
  volumes:
    - jaeger_data:/badger/

# Update OTEL Collector exporters
exporters:
  jaeger:
    endpoint: jaeger:14250
```

#### 3. Reverse Proxy (Nginx/Traefik)

```yaml
# docker-compose entry for Nginx
nginx:
  image: nginx:alpine
  ports:
    - "80:80"
    - "443:443"
  volumes:
    - ./nginx.conf:/etc/nginx/nginx.conf:ro
    - ./certs:/etc/nginx/certs:ro
  depends_on:
    - grafana
    - prometheus
```

#### 4. Authentication

```yaml
# In grafana/provisioning/datasources/datasources.yml
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    secureJsonData:
      basicAuthPassword: "${PROM_PASSWORD}"
    jsonData:
      basicAuthUser: "promuser"
```

#### 5. Log Retention Policy

```yaml
# Update loki-config.yml
limits_config:
  retention_period: 720h      # 30 days
  max_chunks_per_query: 100000
  cardinality_limit: 100000

# Separate tier for old data
schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: s3       # Use S3, not filesystem
```

#### 6. High Availability (Prometheus)

```yaml
# Use external Thanos coordinator
# Or: Prometheus federation for multi-cluster

global:
  external_labels:
    cluster: "production"
    replica: "1"            # Increment per replica

remote_write:
  - url: "http://thanos-receiver:19291/api/v1/receive"
```

---

## Key Learnings: 5-Day Observability Block

### Day 73: Prometheus Fundamentals
✅ **What You Built**: Prometheus server, scrape jobs, PromQL basics  
🎯 **Core Insight**: "Observability starts with metrics. Prometheus is the most popular open-source solution because it's pull-based (servers pull from exporters), not push-based (exporters flood servers). This prevents cardinality explosion."

### Day 74: System & Container Monitoring
✅ **What You Built**: Node Exporter (host metrics), cAdvisor (container metrics), Grafana dashboards  
🎯 **Core Insight**: "You need both system-level (CPU, disk, network) and container-level (per-service resource usage) metrics. Grafana's visual layers abstract the complexity—dashboards are for humans, panels are for specific questions."

### Day 75: Logs & Correlation
✅ **What You Built**: Loki (log aggregation), Promtail (log shipper), LogQL (log queries)  
🎯 **Core Insight**: "Logs at scale require a different database model. Loki uses label-based indexing (not full-text search), which makes logs 10x cheaper than Elasticsearch. Log-metric correlation (same labels in both) is the bridge between signals."

### Day 76: Distributed Tracing
✅ **What You Built**: OTEL Collector (trace collection), receivers/processors/exporters  
🎯 **Core Insight**: "Traces follow a single request through your entire system. OTEL Collector is vendor-agnostic—you can switch backends (Jaeger → Tempo → Datadog) without changing application instrumentation."

### Day 77: Production Integration
✅ **What You Built**: Full 8-service stack, unified dashboard, production comparison  
🎯 **Core Insight**: "Observability isn't add-on—it's architecture. Three pillars (metrics, logs, traces) + alerting + on-call runbooks + blameless postmortems = production readiness."

### Cross-Cutting Patterns Recognized

1. **Signal Standardization**: Prometheus exports metrics in a standard format; Promtail sends logs in a standard format; OTEL Collector accepts spans in standard format. Swap any component without breaking the pipeline.

2. **Label-Everything Philosophy**: Every metric, log line, and span has labels (tags). These labels are not strings—they're the query API. `{job="docker", container_name="notes-app"}` is more powerful than grepping logs.

3. **Pull vs. Push**: 
   - Prometheus **pulls** (servers ask "give me your metrics")
   - Promtail **pushes** (shipper sends logs to Loki)
   - OTEL Collector **receives** (apps push spans)
   
   Each has tradeoffs. Pull is simpler for discovery (new containers auto-scraped). Push is better for offloading (servers don't scrape—collectors ingest).

4. **Retention vs. Cost**:
   - Prometheus: 15 days (raw metrics storage is cheap per byte but expensive per series due to cardinality)
   - Loki: 30+ days (log storage is compressed; indices are tiny)
   - Traces: 7-14 days (trace storage is expensive; most orgs sample traces or use ring buffers)
   
   Production systems need tiered storage: hot (7 days), warm (30 days), cold (archive).

---

## Architecture Decision Record (ADR)

### ADR-001: Why Open Source Observability?

**Decision**: Build full observability stack using open-source (Prometheus, Loki, OTEL) instead of managed service (Datadog/New Relic).

**Rationale**:
- **Cost**: Open-source is free; Datadog costs $25-100 per host per month at scale
- **Vendor Lock-in**: Own your data; OTEL is multi-backend compatible
- **Learning**: Understand internals; managed services hide complexity
- **On-Prem**: Deploy in airgapped networks without vendor approval

**Tradeoffs**:
- **Operational Burden**: You manage upgrades, backups, troubleshooting
- **Feature Lag**: Managed solutions have newer features (ML anomaly detection, etc.)
- **Support**: Open-source community ≠ vendor SLA support

**For Production**: Use open-source for internal infrastructure + dev/staging; consider managed for critical production (better SLA).

---

### ADR-002: Why OTEL Collector as Middleman?

**Decision**: Use OpenTelemetry Collector between apps and backends instead of direct app-to-backend export.

**Rationale**:
- **Vendor Agnostic**: App doesn't know if traces go to Jaeger, Tempo, or Datadog
- **Reliability**: Collector retries failed exports; app doesn't block on network issues
- **Processing**: Batch, filter, enrich spans in one place
- **Scale**: One collector per datacenter; apps don't overload backend

**Tradeoffs**:
- **Latency**: Additional hop (app → collector → backend adds ~10ms)
- **Complexity**: Another service to operate and monitor

**For Production**: Essential. Protects apps from backend outages.

---

### ADR-003: Prometheus Pull vs. Loki Push

**Decision**: Prometheus scrapes (pull); Loki ingests (push).

**Rationale**:
- **Prometheus Pull**: Automatic service discovery; Prometheus is the source of truth about what exists
- **Loki Push**: Logs are inherently unbounded; apps push when they have data (pressure-relief valve)

**Tradeoffs**:
- Pull requires open ports on every exporter (firewall complexity)
- Push requires apps to know collector address (config management)

**For Production**: This is the community standard. Other configurations are justified only for special cases.

---

## Comparison to Managed Solutions

### Datadog

| Aspect | This Stack | Datadog |
|--------|---|---|
| Cost | $0 (self-hosted) | $25-100/host/month |
| Setup Time | 5 days to understand | 5 minutes (wizard) |
| Features | Metrics, Logs, Traces | ✅ + AI, Synthetics, APM |
| Data Retention | 15-30 days | 365 days (with cost) |
| Compliance | On-premises option | US cloud only (EU available) |
| Vendor Lock-in | None (OTEL portable) | High (custom dashboards, tags) |

**When to Choose Datadog**: SaaS company, high-velocity feature releases, small ops team, budget allows.

### New Relic

| Aspect | This Stack | New Relic |
|--------|---|---|
| Cost | $0 | $15-50/node/month |
| Setup | 5 days | UI-driven onboarding |
| Features | Metrics, Logs, Traces | ✅ + Change Tracking, Incidents |
| Data Retention | 15-30 days | 30 days (free), 365 days (paid) |
| Compliance | On-premises | Separate EU region |
| Vendor Lock-in | None | Medium (NRQL portable, UI not) |

**When to Choose New Relic**: Enterprise with compliance needs, mature incident response, willing to pay for SaaS overhead.

### AWS CloudWatch

| Aspect | This Stack | CloudWatch |
|---|---|---|
| Cost | $0 | Pay-as-you-go (~$5-20/server/month typical) |
| Setup | 5 days | Native if using AWS |
| Features | Metrics, Logs, Traces (Xray) | ✅ + Alarms, Insights, Synthetics |
| Data Retention | 15-30 days | 1 year (logs after 7 days billed less) |
| Compliance | Any | AWS-native compliance (SOC2, PCI, HIPAA regions) |
| Vendor Lock-in | None | Very high (CloudWatch → Lambdas → RDS = ecosystem) |

**When to Choose CloudWatch**: Entire workload on AWS, want single-pane-of-glass for AWS services, compliance mandatory.

### Grafana Cloud

| Aspect | This Stack | Grafana Cloud |
|---|---|---|
| Cost | $0 | $60/month (lite) to $500+ (pro) |
| Setup | 5 days | 1 day (hosted Grafana + Prometheus/Loki backends) |
| Features | Metrics, Logs, Traces | ✅ Same (but hosted) |
| Data Retention | 15-30 days | 30 days (free tier) to 1 year (paid) |
| Compliance | Any | SOC2, HIPAA available |
| Vendor Lock-in | None | Low (portable YAML, exportable dashboards) |

**When to Choose Grafana Cloud**: Want open-source stack but don't want to operate it; budget allows $60-500/month; want Prometheus/Loki without infrastructure.

---

## Handoff Checklist

### For Next Engineer (Runbook)

- [x] **Setup**: Reference repo cloned, `docker compose up -d`, all services UP
- [x] **Access**: Grafana (http://localhost:3000, admin/admin), Prometheus (http://localhost:9090), Loki queries in Explore
- [x] **Health Checks**: All 4 Prometheus scrape jobs UP, Loki receiving logs, OTEL Collector processing traces
- [x] **Dashboard**: "Production Overview" dashboard showing system health, container metrics, logs, traces
- [x] **Alerts**: Alert rules configured in Prometheus (see next steps)
- [x] **Logs**: Run `for i in $(seq 1 50); do curl http://localhost:8000 > /dev/null; done` to generate traffic
- [x] **Traces**: Send OTLP test trace; verify in `docker logs otel-collector`
- [x] **Data Persistence**: Named volumes (prometheus_data, loki_data, grafana_data) survive restarts
- [x] **Retention**: Prometheus 15 days, Loki 30 days by default; configure longer if needed
- [x] **Upgrades**: Image versions pinned in docker-compose.yml; test upgrades in dev first

### Common Issues & Fixes

| Issue | Cause | Fix |
|-------|-------|-----|
| Grafana shows "No data" for Loki | No logs yet OR wrong time range | Run curl loop to generate traffic; expand time range to "Last 1 hour" |
| Prometheus targets show UNKNOWN | Scrape config error OR target down | Check `docker logs prometheus`, verify scrape_configs YAML |
| Promtail can't read Docker logs | Mount permissions | Ensure `/var/run/docker.sock` is readable by container |
| OTEL Collector crashes | Config YAML error | Check `docker logs otel-collector` for parsing errors |
| Disk fills up | Log/metric retention unbounded | Set `retention_period` in Loki, `retention` in Prometheus |

---

## File Structure

```
observability-for-devops/
├── docker-compose.yml                 # 8 services: Prometheus, Node Exporter, cAdvisor, 
│                                       # Grafana, Loki, Promtail, OTEL Collector, Notes App
├── prometheus.yml                     # Prometheus scrape config (4 jobs)
├── alert-rules.yml                    # Prometheus alerting rules (add Alertmanager integration)
├── grafana/
│   ├── provisioning/
│   │   ├── datasources/
│   │   │   └── datasources.yml        # Auto-provision Prometheus, Loki, OTEL
│   │   └── dashboards/
│   │       ├── dashboards.yml         # Dashboard provisioning config
│   │       └── production-overview.json  # "Production Overview" dashboard JSON
│   └── grafana.ini                    # Grafana config (optional)
├── loki/
│   └── loki-config.yml                # Loki storage, schema, retention (30 days)
├── promtail/
│   └── promtail-config.yml            # Docker log collection, batching, pipeline stages
├── otel-collector/
│   └── otel-collector-config.yml      # OTLP receivers, processors, logging exporter
├── notes-app/                         # Sample Django REST API (included in reference repo)
│   ├── Dockerfile
│   ├── requirements.txt
│   └── manage.py
└── day-77-observability-project.md    # This file (handoff documentation)
```

---

## Summary

### What This Stack Provides

✅ **Metrics**: Prometheus scrapes system, container, and application metrics every 15 seconds  
✅ **Logs**: Loki ingests container logs, queryable via LogQL, correlated with metrics via labels  
✅ **Traces**: OTEL Collector receives distributed traces from apps, shows parent-child relationships  
✅ **Visualization**: Grafana unified dashboard combines all three signals  
✅ **Alerting**: Prometheus evaluates alert rules; integrable with Alertmanager → Slack/PagerDuty  
✅ **Portability**: OTEL is vendor-agnostic; swap Loki for Splunk, Prometheus for Cortex, no app changes  

### Why This Matters

Observability is not just monitoring (check if it's up). It's the ability to ask any question about your system without pre-defining dashboards:
- "Why did request latency spike at 14:30?"
- "Which container consumed the most CPU?"
- "Show me all errors from the payment service in the last 5 minutes."
- "Correlate the database slowness with the network latency spike."

This stack answers all of these. That's production readiness.

### Next Steps

1. **Alertmanager**: Add alert routing to Slack/PagerDuty; set up on-call escalation
2. **Jaeger/Tempo**: Replace debug exporter with persistent trace storage
3. **High Availability**: Add second Prometheus replica, setup Loki clustering
4. **HTTPS/TLS**: Reverse proxy all endpoints, enable authentication
5. **Runbook**: Document playbooks for top 10 alerts (high CPU, disk full, etc.)
6. **Learn in Public**: Share the journey (LinkedIn, blog, etc.)

---

## References

- **Repository**: https://github.com/LondheShubham153/observability-for-devops
- **Prometheus Docs**: https://prometheus.io/docs/
- **Grafana Dashboards**: https://grafana.com/grafana/dashboards/
- **Loki LogQL**: https://grafana.com/docs/loki/latest/logql/
- **OpenTelemetry**: https://opentelemetry.io/
- **OTEL Collector Config**: https://opentelemetry.io/docs/collector/configuration/
- **#90DaysOfDevOps**: https://github.com/MichaelCade/90DaysOfDevOps

---

**Completed**: Day 77 of #90DaysOfDevOps  
**Duration**: 5 days (Days 73-77) of intensive observability learning  
**Skills Acquired**: Prometheus, PromQL, Grafana, Loki, LogQL, OpenTelemetry, Docker Compose orchestration, production architecture  
**Next Block**: Kubernetes (Days 78-84) or Infrastructure as Code (Days 49-56 in parallel track)

---

*Handoff prepared for the next engineer. Stack is ready for production deployment with the additions noted above.*
