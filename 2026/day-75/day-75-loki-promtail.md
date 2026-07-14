# Day 75: Log Management with Loki and Promtail

## Overview

Metrics tell you **what** is broken. Logs tell you **why**. Today I added the second pillar of observability to my DevOps stack: **logs**.

I set up Grafana Loki (log aggregation) and Promtail (log collection agent) to:
- Collect all Docker container logs
- Ship them to Loki for storage and indexing
- Query them with LogQL in Grafana
- Correlate metrics and logs side by side

By the end of this day, my Grafana instance shows both metrics and logs together for complete observability.

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│                     OBSERVABILITY STACK                         │
│                                                                 │
├──────────────────┬──────────────────┬──────────────────┐        │
│                  │                  │                  │        │
│   CONTAINERS     │    METRICS       │      LOGS        │        │
│   (Your Apps)    │    PIPELINE      │      PIPELINE    │        │
│                  │                  │                  │        │
└──────────┬───────┴──────────┬───────┴──────────┬───────┘        │
           │                  │                  │                 │
           │                  │                  │                 │
    ┌──────▼──────┐    ┌──────▼──────┐    ┌──────▼──────┐        │
    │   Docker    │    │  cAdvisor   │    │  Promtail   │        │
    │ Containers  │    │  Node Exp.  │    │  (Agent)    │        │
    └──────┬──────┘    └──────┬──────┘    └──────┬──────┘        │
           │                  │                  │                 │
           │                  │                  │                 │
           │          ┌───────▼────────┐         │                 │
           │          │  Prometheus    │         │                 │
           │          │  (Metrics DB)  │         │                 │
           │          └────────┬───────┘         │                 │
           │                   │                 │                 │
           └───────────────────┼─────────────────┤                 │
                               │                 │                 │
                     ┌─────────▼──────────┐      │                 │
                     │      Grafana       │      │                 │
                     │  (Dashboards &     │      │                 │
                     │   Visualization)   │◄─────┘                 │
                     │                    │                        │
                     └──────────┬─────────┘                        │
                                │                                   │
                          ┌──────▼──────┐                          │
                          │    Loki     │                          │
                          │  (Logs DB)  │                          │
                          └─────────────┘                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Data Flow:**
1. Docker containers write logs to `/var/lib/docker/containers/`
2. **Promtail** reads these log files, adds labels (container_name, job), and pushes to Loki
3. **Loki** stores logs and indexes them by labels (not full-text)
4. **Grafana** queries Loki with LogQL to display logs and correlate with metrics
5. User can now debug by seeing metrics AND logs at the same time

---

## Why Loki Instead of ELK?

### Loki's Key Design Decision: Label-Based Indexing

Loki does **NOT** index full log text like Elasticsearch does. Instead:
- ✅ **Only labels are indexed** (container_name, job, filename, severity)
- ✅ Much **cheaper** to operate (smaller index size)
- ✅ **Simpler** to run (no complex tuning)
- ✅ **Log retention** is less expensive
- ❌ Full-text search is slower (but still works)
- ❌ Not ideal for searching arbitrary log content

**Trade-off:** Simplicity + Cost vs. Search Power

This is the same design philosophy as Prometheus:
- Prometheus indexes by **labels**, not all possible values
- Loki indexes by **labels**, not all possible text
- Both are operational nightmares to run at scale without this constraint

---

## Loki Configuration

### File: `loki/loki-config.yml`

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory
  replication_factor: 1
  path_prefix: /loki

schema_config:
  configs:
    - from: 2020-10-24
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

### Configuration Explanation

| Section | Key Setting | Purpose |
|---------|-------------|---------|
| `auth_enabled` | `false` | Single-tenant mode (no auth needed for learning) |
| `server.http_listen_port` | `3100` | HTTP API port for Promtail and Grafana |
| `ring.kvstore.store` | `inmemory` | Single-node setup (no distributed coordination) |
| `replication_factor` | `1` | Single instance, no replication (fine for learning) |
| `schema_config.store` | `tsdb` | Time-series database for fast label indexing |
| `object_store` | `filesystem` | Store log chunks on local disk |
| `index.period` | `24h` | Create new index every 24 hours |
| `storage_config.filesystem.directory` | `/loki/chunks` | Where to persist log data |

### Why This Config?

- **Single-tenant**: No user isolation needed in a dev environment
- **In-memory kvstore**: Acceptable for one node; scales to distributed with etcd/Consul
- **TSDB + Filesystem**: Perfect for laptops/single VMs; replaces the old BoltDB
- **24h index rotation**: Prevents any single index from getting too large

---

## Promtail Configuration

### File: `promtail/promtail-config.yml`

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker
    static_configs:
      - targets:
          - localhost
        labels:
          job: docker
          __path__: /var/lib/docker/containers/*/*-json.log
    pipeline_stages:
      - docker: {}
```

### Configuration Explanation

| Section | Key Setting | Purpose |
|---------|-------------|---------|
| `server.http_listen_port` | `9080` | Promtail metrics endpoint (for monitoring Promtail itself) |
| `positions.filename` | `/tmp/positions.yaml` | Bookmark file: tracks which logs have been shipped |
| `clients[0].url` | `http://loki:3100/...` | Where to push logs (Loki endpoint) |
| `scrape_configs[0].job_name` | `docker` | Job label applied to all collected logs |
| `__path__` | `/var/lib/docker/containers/*/*-json.log` | Glob pattern to find Docker log files on host |
| `pipeline_stages.docker` | `{}` | Parse Docker JSON format; extract timestamp, stream, message |

### Key Points

- **Positions file**: Acts like a bookmark—tracks which lines have been shipped. Delete it to replay all logs.
- **__path__**: Special label that Promtail uses as a glob pattern (not shipped to Loki)
- **docker pipeline stage**: Automatically extracts fields from Docker's JSON log format
- **Static config**: Single localhost target; container discovery happens via docker.sock

### Docker Compose Configuration

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_INSTALL_PLUGINS=grafana-piechart-panel
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"
    restart: unless-stopped

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    restart: unless-stopped

  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - "3100:3100"
    volumes:
      - ./loki/loki-config.yml:/etc/loki/loki-config.yml
      - loki_data:/loki
    command: -config.file=/etc/loki/loki-config.yml
    restart: unless-stopped

  promtail:
    image: grafana/promtail:latest
    container_name: promtail
    volumes:
      - ./promtail/promtail-config.yml:/etc/promtail/promtail-config.yml
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock
    command: -config.file=/etc/promtail/promtail-config.yml
    restart: unless-stopped

  notes-app:
    image: your-notes-app:latest
    container_name: notes-app
    ports:
      - "8000:5000"
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
  loki_data:
```

### Why These Volume Mounts for Promtail?

| Mount | Reason |
|-------|--------|
| `/var/lib/docker/containers:/var/lib/docker/containers:ro` | Contains all container log files in JSON format |
| `/var/run/docker.sock:/var/run/docker.sock` | Allows Promtail to query Docker daemon for container metadata |
| `./promtail/promtail-config.yml:/etc/promtail/...` | Config file for Promtail |

---

## Loki as a Grafana Datasource

### Provisioning Method (Recommended)

Updated `grafana/provisioning/datasources/datasources.yml`:

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    editable: false
```

**Advantages:**
- Automatic provisioning when Grafana starts
- No manual UI configuration needed
- Reproducible setup

**Restart Grafana:**
```bash
docker compose restart grafana
```

---

## LogQL Queries

### Query 1: All error logs from notes-app container

```logql
{container_name="notes-app"} |= "error"
```

**Result:** Shows all log lines from the notes-app container that contain the word "error" (case-sensitive).

**Use case:** Quickly find exceptions or error messages in a specific container.

---

### Query 2: Count error lines per minute

```logql
count_over_time({container_name="notes-app"} |= "error"[1m])
```

**Result:** Returns a time series showing how many error lines appeared in each 1-minute window over the last hour.

**Use case:** Track error frequency over time; spot when errors became frequent.

---

### Query 3: Top 5 containers by log volume

```logql
topk(5, sum by (container_name) (rate({job="docker"}[5m])))
```

**Result:** Shows the 5 containers producing the most log lines per second over the last 5 minutes.

**Use case:** Identify which container is generating excessive logs (possible infinite loop or debug logging).

---

### Query 4: HTTP 4xx and 5xx errors

```logql
{job="docker"} |~ "status=[45]\\d{2}"
```

**Result:** All log lines containing HTTP status codes in the 400-599 range.

**Use case:** Find application errors when you know they include HTTP status codes in logs.

---

### Query 5: Exclude health check noise

```logql
{job="docker"} != "health" != "ping" != "readiness"
```

**Result:** All logs EXCEPT those containing "health", "ping", or "readiness" keywords.

**Use case:** Reduce noise from health checks and focus on application logs.

---

## Screenshots

### Screenshot 1: Grafana Explore with Loki

**[INSERT SCREENSHOT: Grafana > Explore > Loki datasource > query {job="docker"}]**

Shows:
- Datasource selector set to "Loki"
- Query: `{job="docker"}`
- Timeline graph showing log volume over time
- Log panel below showing individual log lines with timestamps

---

### Screenshot 2: Metrics and Logs Side by Side

**[INSERT SCREENSHOT: Grafana Explore with split view enabled]**

Shows:
- **Left panel**: Prometheus metric `rate(container_cpu_usage_seconds_total{name="notes-app"}[5m])`
- **Right panel**: Loki query `{container_name="notes-app"}`
- Time ranges synchronized so CPU spike correlates with log output

**Key observation:** When CPU spiked at 14:32, the logs show a burst of "processing" messages, indicating the app was under load.

---

## Loki vs ELK Stack

### Comparison Table

| Aspect | Loki | Elasticsearch (ELK) |
|--------|------|-------------------|
| **Indexing Strategy** | Label-based (like Prometheus) | Full-text indexed |
| **Storage Cost** | ✅ Low (indexes only labels) | ❌ High (indexes all text) |
| **Operational Complexity** | ✅ Simple | ❌ Complex (tuning, sharding) |
| **Full-Text Search** | ❌ Slow (not indexed) | ✅ Fast (fully indexed) |
| **Search Latency** | Fast for label queries | Fast for full-text queries |
| **Log Retention** | ✅ Cheap (high retention) | ❌ Expensive (limited retention) |
| **Integration with Prometheus** | ✅ Native (same label model) | ❌ Separate ecosystem |
| **Cardinality Handling** | ✅ Strict limits prevent meltdown | ❌ Easy to create cardinality bombs |
| **Learning Curve** | ✅ Easy (especially from Prometheus) | ❌ Steep |
| **Scalability** | Single-node to cluster | Enterprise-grade distributed |

### When to Use Loki

✅ **Use Loki when:**
- You're already using Prometheus and Grafana
- You need simple, reliable log storage
- Your organization wants low operational overhead
- You primarily search logs by container/service/environment (labels)
- You want cheap long-term log retention
- You're running on a budget (small team, limited infrastructure)

**Example:** DevOps team running 20-30 microservices in Kubernetes. Labels are container_name, namespace, pod, service. Most queries are "show me logs from service X" or "show me errors in namespace Y". Perfect fit for Loki.

---

### When to Use ELK Stack

✅ **Use ELK when:**
- You need powerful full-text search across all log content
- Your organization has log analysis as a primary requirement
- You search logs by arbitrary keywords or message content
- You need advanced security/compliance log analysis
- You have the ops team to maintain Elasticsearch
- You need SIEM-like capabilities

**Example:** Large financial institution needs to search transaction logs for "user_id=12345" or "amount > $1M" across billions of events. Needs compliance audits of exact log queries. ELK's full-text search is essential.

---

### When to Use Both

Some organizations run **both**:
- **Loki**: For real-time operational logs (container, application, system)
- **ELK**: For compliance, security, and deep forensic analysis

**Trade-off principle:**
- Loki = "Prometheus for logs" → simple, cheap, operational
- ELK = "Google for logs" → powerful, expensive, comprehensive

---

## Key Learnings

### 1. Labels are Everything in Loki

Unlike traditional logging (where you search by text), Loki searches by **labels**.

```logql
# Good: using labels that already exist
{container_name="app", job="docker"}

# Bad: attempting to search non-label fields (slow)
{} |= "user_id=12345"

# Best: Add user_id as a label in Promtail if you search it frequently
```

**Lesson:** Plan your labels carefully. Low cardinality labels = fast queries.

---

### 2. Positions.yaml is Your State Machine

Promtail tracks which logs it has shipped in `positions.yaml`:

```bash
# Delete positions.yaml to replay all logs from the start
rm /tmp/positions.yaml
docker compose restart promtail
```

This is useful for debugging ("why didn't my error show up?") or re-indexing after config changes.

---

### 3. Correlation Solves Real Incidents

The power of having metrics and logs in one place:

1. **See metric anomaly** → CPU spike at 14:32
2. **Click on the spike** → Time range narrows
3. **Look at logs** → See "OutOfMemory" exception at 14:32:15
4. **Root cause found** → Memory leak in garbage collection

This would take 10 minutes across separate systems (Datadog + Splunk). Takes 30 seconds in Grafana.

---

### 4. Loki is Schemaless (But Plan Anyway)

You don't need to pre-define log schemas in Loki, but you should:
- Standardize label names (container_name, not containerName)
- Keep label cardinality low (container_name ✅, user_id ❌)
- Document what each label means for your team

---

## Troubleshooting

### Promtail Shows No Logs in Grafana

**Checklist:**
1. Is Promtail running? `docker ps | grep promtail`
2. Is Loki running? `curl http://localhost:3100/ready` (should return "ready")
3. Does `/var/lib/docker/containers/` exist on your host?
4. Check Promtail targets: `curl http://localhost:9080/targets` (should show docker job)
5. Check Promtail logs: `docker logs promtail`

**Common issue on macOS:** Docker Desktop stores container logs inside the Docker VM. Promtail must run as a container (not on host) to access them.

---

### Query Returns No Results

1. Verify logs exist: `{job="docker"}` (should show something)
2. Check label names: `{container_name="prometheus"}` vs `{container="prometheus"}`
3. Case sensitivity: `|= "error"` won't match "ERROR". Use `|~ "(?i)error"` for case-insensitive.
4. Verify time range: Logs older than Promtail started won't exist.

---

### High Cardinality Label Error

If you see "too many streams" errors:

```logql
# Bad (too many unique values for this label)
{user_id="*"}

# Good (low cardinality)
{service="api", environment="prod"}
```

Cardinality = number of unique values for a label. Loki enforces limits to prevent memory exhaustion.

---

## Summary

Today I completed the second pillar of observability:

✅ Set up **Loki** as a log storage backend  
✅ Configured **Promtail** to collect all Docker container logs  
✅ Added Loki to Grafana as a datasource  
✅ Wrote **LogQL queries** to filter and analyze logs  
✅ Correlated **metrics and logs side by side** in Grafana  
✅ Understood the **trade-offs** between Loki and ELK  

Now when something breaks:
- Prometheus + Grafana shows **what** broke (metrics)
- Loki + Grafana shows **why** it broke (logs)
- Same dashboard = faster incident response

---

## Next Steps (Day 76+)

- Add alerting based on log patterns (e.g., "alert if 10+ errors in 1 minute")
- Set up log aggregation labels for multi-tenant scenarios
- Explore Promtail pipeline stages for structured logging (JSON parsing)
- Add retention policies to Loki to save storage costs


