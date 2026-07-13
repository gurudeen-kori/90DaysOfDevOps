# Day 74 -- Node Exporter, cAdvisor, and Grafana Dashboards

## Overview

Today we expanded the observability stack from basic Prometheus self-monitoring to full-stack monitoring with:
- **Node Exporter** for host-level metrics (CPU, memory, disk, network)
- **cAdvisor** for container-level metrics (Docker resource usage)
- **Grafana** for visualization and dashboarding

This creates a production-ready monitoring solution that provides visibility into both infrastructure and application performance.

---

## Configuration Files

### docker-compose.yml

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    restart: unless-stopped

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    restart: unless-stopped

  grafana:
    image: grafana/grafana-enterprise:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin123
    restart: unless-stopped

  notes-app:
    image: notes-app:latest
    container_name: notes-app
    ports:
      - "5000:5000"
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
```

### prometheus.yml

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]

  - job_name: "cadvisor"
    static_configs:
      - targets: ["cadvisor:8080"]
```

### grafana/provisioning/datasources/datasources.yml

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
```

---

## Node Exporter vs cAdvisor: When to Use Each

### Node Exporter

**What it monitors:** Host-level system metrics

| Metric Category | Examples |
|---|---|
| CPU | CPU time per mode (idle, system, user), load average |
| Memory | Total, available, used, buffers, cached |
| Disk | Filesystem size, available space, I/O operations |
| Network | Bytes sent/received, packets, errors, dropped |
| Process | Number of running processes, file descriptor usage |
| System | Boot time, context switches, interrupts |

**Key characteristics:**
- Runs on the host (via Docker container with volume mounts)
- Exposes `/proc`, `/sys`, and `/` filesystems to gather data
- Metric prefix: `node_*`
- Lightweight, minimal overhead
- Best for infrastructure team visibility

**When to use:**
- Capacity planning (CPU usage trends, memory saturation)
- Troubleshooting host-level performance issues
- Infrastructure health dashboards
- Alerting on resource exhaustion

---

### cAdvisor

**What it monitors:** Container-level resource usage and performance

| Metric Category | Examples |
|---|---|
| CPU | Container CPU usage, CPU throttling |
| Memory | Container memory usage, memory limits, OOM events |
| Network | Per-container bytes sent/received, network errors |
| I/O | Disk read/write operations and bytes per container |
| Filesystem | Container filesystem usage |

**Key characteristics:**
- Runs as a container, monitors all running containers
- Accesses Docker socket and cgroups to gather container stats
- Metric prefix: `container_*`
- Provides per-container isolation of resource usage
- Best for application-level observability

**When to use:**
- Container resource allocation (is container getting enough CPU/memory?)
- Noisy neighbor detection (which container is consuming resources?)
- Cost tracking (resource usage per container/service)
- Per-service alerting

---

### Quick Comparison

| Aspect | Node Exporter | cAdvisor |
|---|---|---|
| **Scope** | Entire host | Individual containers |
| **Use case** | Infrastructure health | Application performance |
| **Data granularity** | Aggregated system level | Per-container isolation |
| **Queries** | `node_cpu_seconds_total` | `container_cpu_usage_seconds_total{name!=""}` |
| **Best for** | Ops/infrastructure teams | Dev teams, SREs |

**In practice:** You need both. Use Node Exporter for "Is the host overloaded?" and cAdvisor for "Which container is causing it?"

---

## Prometheus Targets Verification

After running `docker compose up -d`, verify all targets are healthy:

**URL:** `http://localhost:9090/targets`

**Expected output:**
```
prometheus (localhost:9090) ............ UP
node-exporter (node-exporter:9100) ..... UP
cadvisor (cadvisor:8080) ............... UP
```

All three should show green "UP" status. If any show "DOWN":
1. Check service logs: `docker compose logs <service-name>`
2. Verify port mappings: `docker compose ps`
3. Test connectivity: `curl http://localhost:<port>/metrics`

---

## PromQL Queries for Key Metrics

### CPU Metrics

**Host CPU Usage Percentage (Gauge)**
```promql
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```
Shows average CPU utilization across all cores. Value ranges 0-100%.

**Per-Core CPU Usage**
```promql
rate(node_cpu_seconds_total{mode="user"}[5m]) * 100
```
Shows CPU time spent in user-space per core.

**Container CPU Usage (Rate)**
```promql
rate(container_cpu_usage_seconds_total{name!=""}[5m]) * 100
```
Shows CPU usage trend for each container over the last 5 minutes.

---

### Memory Metrics

**Host Memory Usage Percentage**
```promql
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
```
Calculates memory utilization as a percentage of total available.

**Host Memory Stats (Bytes)**
```promql
node_memory_MemTotal_bytes      # Total RAM installed
node_memory_MemAvailable_bytes  # RAM available for allocation
node_memory_MemFree_bytes       # Free (not in use)
node_memory_Buffers_bytes       # Kernel buffers
node_memory_Cached_bytes        # Cached data
```

**Container Memory Usage (MB)**
```promql
container_memory_usage_bytes{name!=""} / 1024 / 1024
```
Shows memory consumption per container in megabytes.

**Top 3 Memory-Hungry Containers**
```promql
topk(3, container_memory_usage_bytes{name!=""})
```
Identifies the three containers using the most memory.

---

### Disk Metrics

**Root Filesystem Usage Percentage**
```promql
(1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100
```
Calculates disk utilization for root partition.

**All Filesystems Usage**
```promql
node_filesystem_avail_bytes / node_filesystem_size_bytes
```
Ratio of available to total space per filesystem.

**Disk I/O Rate (Read Bytes/sec)**
```promql
rate(node_disk_read_bytes_total[5m])
```
Shows disk read throughput.

**Disk I/O Rate (Write Bytes/sec)**
```promql
rate(node_disk_written_bytes_total[5m])
```
Shows disk write throughput.

---

### Network Metrics

**Network Receive Rate (Bytes/sec)**
```promql
rate(node_network_receive_bytes_total[5m])
```
Bytes per second received on each network interface.

**Network Transmit Rate (Bytes/sec)**
```promql
rate(node_network_transmit_bytes_total[5m])
```
Bytes per second transmitted on each network interface.

**Container Network Receive (Bytes/sec)**
```promql
rate(container_network_receive_bytes_total{name!=""}[5m])
```
Network input per container.

**Container Network Transmit (Bytes/sec)**
```promql
rate(container_network_transmit_bytes_total{name!=""}[5m])
```
Network output per container.

---

## Grafana Dashboard Configuration

### Dashboard: DevOps Observability Overview

Create a new dashboard with these 5 panels:

#### Panel 1: CPU Usage % (Gauge)

**Metric:**
```promql
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**Configuration:**
- Visualization Type: Gauge
- Title: "CPU Usage %"
- Max: 100
- Thresholds:
  - Green: 0-60
  - Yellow: 60-80
  - Red: 80-100
- Unit: Percent (0-100)

---

#### Panel 2: Memory Usage % (Gauge)

**Metric:**
```promql
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
```

**Configuration:**
- Visualization Type: Gauge
- Title: "Memory Usage %"
- Max: 100
- Thresholds:
  - Green: 0-60
  - Yellow: 60-80
  - Red: 80-100
- Unit: Percent (0-100)

---

#### Panel 3: Container CPU Usage (Time Series)

**Metric:**
```promql
rate(container_cpu_usage_seconds_total{name!=""}[5m]) * 100
```

**Configuration:**
- Visualization Type: Time series
- Title: "Container CPU Usage"
- Legend: {{name}}
- Y-Axis Label: "CPU %"
- Show legend: Always

---

#### Panel 4: Container Memory Usage (Bar Chart)

**Metric:**
```promql
container_memory_usage_bytes{name!=""} / 1024 / 1024
```

**Configuration:**
- Visualization Type: Bar chart
- Title: "Container Memory (MB)"
- Legend: {{name}}
- X-Axis: Container names
- Y-Axis Label: "Memory (MB)"

---

#### Panel 5: Disk Usage % (Stat)

**Metric:**
```promql
(1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100
```

**Configuration:**
- Visualization Type: Stat
- Title: "Disk Usage %"
- Decimals: 1
- Thresholds:
  - Green: 0-70
  - Yellow: 70-85
  - Red: 85-100
- Unit: Percent (0-100)

---

### Importing Community Dashboards

**Dashboard ID 1860 (Node Exporter Full)**
- Navigate to: Dashboards > New > Import
- Enter ID: 1860
- Select Prometheus datasource
- Import

This dashboard includes 50+ panels covering:
- CPU metrics (per-core, load average, context switches)
- Memory (usage, swap, cache, buffers)
- Disk I/O (read/write rates, operations)
- Network (bytes/packets sent/received)
- Filesystem (inode usage, mount points)
- System (processes, file descriptors, uptime)

**Dashboard ID 193 (Docker monitoring via cAdvisor)**
- Navigate to: Dashboards > New > Import
- Enter ID: 193
- Select Prometheus datasource
- Import

This dashboard provides:
- Per-container CPU and memory usage
- Network I/O per container
- Container count and status
- Resource limits and actual usage comparison

---

## Datasource Provisioning via YAML

### Why Provision Datasources via YAML?

**Manual UI Configuration Issues:**
- ❌ Not repeatable (easy to misconfigure on next deployment)
- ❌ Not version-controlled (can't track changes)
- ❌ Doesn't scale (tedious for multiple environments)
- ❌ No infrastructure-as-code (manual, error-prone)

**YAML Provisioning Benefits:**
- ✅ Repeatable (same config in dev, staging, prod)
- ✅ Version-controlled (git history, code review)
- ✅ Scalable (same file for all environments)
- ✅ Infrastructure-as-code (automated deployments)
- ✅ Idempotent (safe to apply repeatedly)

### How It Works

1. **Create provisioning directory structure:**
```bash
mkdir -p grafana/provisioning/datasources
mkdir -p grafana/provisioning/dashboards
```

2. **Write datasources.yml** (Grafana auto-discovers this file on startup)

3. **Mount in docker-compose.yml:**
```yaml
volumes:
  - ./grafana/provisioning:/etc/grafana/provisioning
```

4. **Grafana automatically:**
   - Reads `/etc/grafana/provisioning/datasources/datasources.yml`
   - Creates datasources without UI interaction
   - Prevents manual edits (sets `editable: false` in config)

### Example: Adding a Second Datasource

To add Loki (logs) alongside Prometheus, just update `datasources.yml`:

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
    isDefault: false
    editable: false
```

Restart Grafana: `docker compose up -d grafana`

No UI clicks needed—Grafana provisions both datasources automatically.

---

## Verification Steps

### 1. Check All Services Running
```bash
docker compose ps
```
Expected: All services showing "Up"

### 2. Test Node Exporter Metrics
```bash
curl http://localhost:9100/metrics | head -20
```
Expected: Output shows `node_*` metrics

### 3. Test cAdvisor Metrics
```bash
curl http://localhost:8080/metrics | head -20
```
Expected: Output shows `container_*` metrics

### 4. Verify Prometheus Scrape Targets
Visit: `http://localhost:9090/targets`
Expected: All three targets (prometheus, node-exporter, cadvisor) show UP

### 5. Test a PromQL Query
Visit: `http://localhost:9090/graph`
Paste: `node_cpu_seconds_total{mode="idle"}`
Expected: Green line/values in graph

### 6. Verify Grafana Datasource
Visit: `http://localhost:3000`
- Login: admin / admin123
- Go to Connections > Data Sources
- Expected: "Prometheus" should be listed and "Successfully queried the Prometheus API"

### 7. Test Custom Dashboard
Visit: Dashboards > DevOps Observability Overview
Expected: All 5 panels show data (gauges display percentages, time series show lines, bar chart shows containers)

---

## Troubleshooting

### "Prometheus targets show DOWN"
**Symptoms:** Targets page shows DOWN status
**Solution:**
1. Check service is running: `docker compose ps`
2. Check logs: `docker compose logs node-exporter`
3. Verify port is open: `curl http://localhost:9100/metrics`
4. Restart service: `docker compose restart node-exporter`

### "Grafana panels show 'No data'"
**Symptoms:** Dashboard panels are empty
**Solution:**
1. Verify datasource is configured: Connections > Data Sources
2. Test query in Prometheus UI first
3. Check time range (upper right in dashboard)
4. Verify containers are actually running: `docker compose ps`

### "cAdvisor metrics not appearing"
**Symptoms:** container_* queries return no results
**Solution:**
1. Ensure Docker socket is mounted: Check volumes in docker-compose.yml
2. Verify cAdvisor can see containers: `docker compose logs cadvisor`
3. Run a container if none exist: `docker run -d nginx`
4. Wait 30 seconds for first scrape

### "Grafana won't start"
**Symptoms:** Container exits immediately
**Solution:**
1. Check logs: `docker compose logs grafana`
2. Ensure volume permissions: `sudo chown -R 472:472 grafana_data/`
3. Restart: `docker compose down && docker compose up -d`

---

## Key Learnings

### Observability Layers

| Layer | Tool | What It Monitors |
|---|---|---|
| Metrics | Prometheus + Node Exporter + cAdvisor | Numeric time-series data (CPU, memory, disk) |
| Visualization | Grafana | Dashboards and alerts based on metrics |
| Infrastructure | Node Exporter | Host/kernel level resource usage |
| Containers | cAdvisor | Application-level resource isolation |

### Metric Naming Conventions

- **Node Exporter:** `node_<category>_<metric>_<unit>`
  - Example: `node_cpu_seconds_total`, `node_memory_MemTotal_bytes`

- **cAdvisor:** `container_<category>_<metric>_<unit>`
  - Example: `container_cpu_usage_seconds_total`, `container_memory_usage_bytes`

### PromQL Rate Function

```promql
rate(metric_name[5m])
```
- Calculates per-second rate of change over the last 5 minutes
- Essential for gauges (metrics that keep growing)
- Smooths out spikes and trends

### Container Filtering

```promql
{name!=""}
```
- Filters out aggregated/system-level entries
- Only shows named containers
- Always use when querying `container_*` metrics

---

## Next Steps

### Short-term
- [ ] Add alert rules (high CPU, low disk space)
- [ ] Create more specific dashboards (per-service, per-team)
- [ ] Set up Prometheus retention policies

### Medium-term
- [ ] Add Loki for centralized log aggregation
- [ ] Integrate with PagerDuty for on-call alerting
- [ ] Build runbooks linked to alerts

### Long-term
- [ ] Implement SLOs and error budgets
- [ ] Build custom exporters for app-specific metrics
- [ ] Deploy observability stack across multiple clusters

---

## References

- [Node Exporter Documentation](https://github.com/prometheus/node_exporter)
- [cAdvisor Documentation](https://github.com/google/cadvisor)
- [Grafana Dashboards](https://grafana.com/grafana/dashboards)
- [PromQL Query Language](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [Observability for DevOps Repo](https://github.com/LondheShubham153/observability-for-devops)

---

## Screenshots Checklist

- [ ] Prometheus Targets page (all 3+ targets UP)
- [ ] Custom Grafana dashboard "DevOps Observability Overview"
- [ ] Imported Node Exporter Full dashboard (ID 1860)
- [ ] Imported Docker monitoring dashboard (ID 193)
- [ ] Dashboard filters/variables (optional but recommended)

---

**Completion Date:** [Add date when completed]

**Status:** ✅ Complete

---

*"Observability isn't about having lots of metrics. It's about asking questions about your system and having the data to answer them."*
