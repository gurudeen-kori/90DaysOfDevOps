# Day 73: Introduction to Observability and Prometheus
## Complete Guide & Deliverables

---

## 📋 What You've Completed Today

### ✅ Task 1: Understanding Observability (Conceptual)
- Difference between observability vs. traditional monitoring
- Three pillars: metrics, logs, traces
- Why DevOps engineers need all three
- Architecture overview for the 5-day observability journey

**Output:** `observability_fundamentals.md` + `day-73-observability-prometheus.md`

### ✅ Task 2: Set Up Prometheus with Docker (Hands-On)
- Created `prometheus.yml` configuration
- Created `docker-compose.yml` with Prometheus + Notes-App
- Prometheus running and scraped itself
- Notes-App added as second scrape target

**Output:** `prometheus.yml`, `docker-compose.yml`

### ✅ Task 3: Prometheus Concepts (Learning)
- Scrape targets (pull-based model)
- Metric types: counter, gauge, histogram, summary
- Labels and time series
- Understanding the `up` metric

**Output:** Detailed explanations in `day-73-observability-prometheus.md`

### ✅ Task 4: PromQL Basics (Practical Skills)
- Instant vectors (current values)
- Range vectors (time windows)
- rate() function (converting counters to per-second)
- sum() and topk() functions
- Label filtering
- Arithmetic operations

**Output:** `PROMQL_REFERENCE.md` (quick lookup guide)

### ✅ Task 5: Sample Application Monitoring (Integration)
- Added notes-app to docker-compose.yml
- Updated prometheus.yml with notes-app scrape target
- Verified both targets are UP and healthy

**Output:** Updated `docker-compose.yml` and `prometheus.yml`

### ✅ Task 6: Data Retention & Storage (Advanced)
- Understood TSDB (time-series database)
- Configured retention: 30 days or 1GB (whichever first)
- Understood why volume mounts matter for persistence
- Learned what happens when retention is exceeded

**Output:** Configuration in `docker-compose.yml`

---

## 📁 Files You Have

### Core Configuration Files

#### `prometheus.yml` (642 bytes)
Prometheus configuration defining:
- Global scrape interval: 15 seconds
- Scrape targets: Prometheus + Notes-App
- Data retention: 30 days or 1GB

**Use this for:** Configuring which services Prometheus monitors

#### `docker-compose.yml` (1007 bytes)
Docker Compose configuration for:
- Prometheus service (port 9090)
- Notes-app service (port 8000)
- Volume for data persistence
- Shared network for inter-container communication

**Use this for:** Starting the entire stack with `docker compose up -d`

### Learning Documentation Files

#### `observability_fundamentals.md` (10 KB)
High-level overview of observability covering:
- Observability definition and concepts
- Monitoring vs. Observability comparison
- Three pillars deep dive
- Real-world scenario breakdown
- Architecture diagrams
- Key takeaways and resources

**Use this for:** Understanding the "why" behind observability

#### `day-73-observability-prometheus.md` (23 KB) ⭐ **START HERE**
Comprehensive Day 73 guide covering:
- All 6 tasks with detailed explanations
- Hands-on setup instructions
- Counter vs. Gauge examples with real scenarios
- Complete PromQL tutorial
- Docker commands and verification
- Troubleshooting hints
- Architecture diagram showing Days 73-77

**Use this for:** Main learning reference and task completion

#### `README.md` (12 KB)
Project overview containing:
- Project structure
- File guide for each component
- Quick start instructions
- Key commands (Docker Compose, Docker Exec)
- Important concepts to remember
- What happens in Days 74-77
- Success metrics and verification

**Use this for:** Navigation and quick reference

### Reference & Troubleshooting Guides

#### `PROMQL_REFERENCE.md` (9.5 KB)
Complete PromQL query reference with:
- Query types (instant, range vectors)
- All common functions (rate, sum, topk, histogram_quantile, etc.)
- Label matching and filtering
- Arithmetic operations
- Common patterns (request rates, latency, resource usage)
- Pro tips for writing better queries
- Example dashboard queries
- Debugging tips

**Use this while:** Writing PromQL queries in Prometheus UI

#### `SETUP_VERIFICATION.md` (11 KB)
Step-by-step verification guide with:
- Quick start instructions
- 5 detailed verification steps
- Common test queries (copy-paste ready)
- Troubleshooting section (10+ common issues with solutions)
- Network connectivity debugging
- Performance baseline expectations
- Success criteria checklist

**Use this when:** Setting up or debugging your stack

---

## 🚀 Quick Start (5 Minutes)

### 1. Copy Files to Your Project

```bash
# Create observability-stack directory
mkdir -p ~/observability-stack
cd ~/observability-stack

# Copy the configuration files
cp prometheus.yml .
cp docker-compose.yml .
```

### 2. Start the Stack

```bash
docker compose up -d
```

**Expected output:**
```
[+] Running 2/2
 ✓ Container prometheus  Started
 ✓ Container notes-app   Started
```

### 3. Verify Everything Works

```bash
# Check containers are running
docker ps

# Generate some traffic
curl http://localhost:8000 && sleep 1 && curl http://localhost:8000

# Open in browser
open http://localhost:9090
```

### 4. Check Targets

- Go to: http://localhost:9090
- Click: Status → Targets
- You should see: Two targets, both UP ✅

### 5. Run Your First Query

- Click: Graph tab
- Query: `up`
- Click: Execute
- You should see: Two lines showing values of 1 (both healthy)

**Success!** 🎉

---

## 📚 Learning Path

### If You're New to Prometheus

1. **First:** Read `observability_fundamentals.md` (15 minutes)
   - Understand the "why" (observability concepts)
   - See the big picture

2. **Second:** Read `day-73-observability-prometheus.md` (1 hour)
   - Understand the "how" (Prometheus specifics)
   - Complete Tasks 1-3 conceptually

3. **Third:** Follow `SETUP_VERIFICATION.md` (30 minutes)
   - Set up your stack
   - Verify everything works
   - Follow Step 1-5 carefully

4. **Fourth:** Practice with `PROMQL_REFERENCE.md` (1 hour)
   - Write queries in Prometheus UI
   - Experiment with different functions
   - Try the examples provided

5. **Final:** Complete Tasks 4-6 in main document
   - Deep dive into PromQL
   - Understand data retention
   - Know when to use counter vs gauge

### If You're Experienced with Monitoring

1. **Scan:** `observability_fundamentals.md` for novelty concepts
2. **Skim:** `day-73-observability-prometheus.md` Tasks 1-3
3. **Execute:** `SETUP_VERIFICATION.md` Steps 1-5
4. **Reference:** Use `PROMQL_REFERENCE.md` as needed
5. **Deep Dive:** Focus on Tasks 4-6 for Prometheus-specific details

---

## 🔍 Key Concepts at a Glance

### The Three Pillars

| Pillar | Tells You | Example | Tools |
|--------|-----------|---------|-------|
| **Metrics** | **WHAT** is broken | High error rate on `/api/users` | Prometheus |
| **Logs** | **WHY** it broke | "Database connection timeout" | Loki |
| **Traces** | **WHERE** it broke | "Payment service took 12s" | Jaeger |

### Counter vs Gauge

| Type | Behavior | Example | Use rate()? |
|------|----------|---------|-------------|
| **Counter** | Only increases | `http_requests_total` | YES ✅ |
| **Gauge** | Up and down | `cpu_usage_percent` | NO ❌ |

### Essential PromQL Pattern

```promql
# For counters: ALWAYS use rate()
rate(counter_metric[5m])

# For aggregating: sum AFTER rate
sum(rate(counter_metric[5m]))

# NOT the other way around
rate(sum(counter_metric)[5m])  # ❌ Wrong!
```

### Pull vs Push

- **Prometheus (Pull):** Actively scrapes targets → More reliable
- **Other systems (Push):** Targets push data → Less reliable for monitoring

---

## ✅ Verification Checklist

Before moving to Day 74, verify:

- [ ] Docker containers running (`docker ps`)
- [ ] Prometheus web UI accessible (http://localhost:9090)
- [ ] Both targets show UP (Status → Targets)
- [ ] Can query metrics (`rate(prometheus_http_requests_total[5m])`)
- [ ] Understand counter vs gauge (with real examples)
- [ ] Understand rate() function and when to use it
- [ ] Know the three pillars and why each is needed
- [ ] Can explain observability vs monitoring
- [ ] Data persists across container restarts
- [ ] prometheus.yml and docker-compose.yml match examples

**All checked?** You're ready for Day 74! 🚀

---

## 🛠️ Common Commands

### Docker Compose

```bash
# Start stack
docker compose up -d

# View logs
docker compose logs -f prometheus
docker compose logs -f notes-app

# Stop stack
docker compose down

# Restart specific service
docker compose restart prometheus

# Clean everything (remove volumes)
docker compose down -v
```

### Docker Exec

```bash
# Check containers
docker ps

# View container logs
docker logs prometheus

# Run commands in container
docker exec prometheus du -sh /prometheus
docker exec notes-app curl http://localhost:8000
```

### Prometheus Web UI

| Page | URL | Purpose |
|------|-----|---------|
| Graph | http://localhost:9090/graph | Write and execute queries |
| Status | http://localhost:9090/status | See configuration and targets |
| Targets | http://localhost:9090/targets | Check scrape status |
| TSDB | http://localhost:9090/tsdb-status | See data storage info |
| Metrics | http://localhost:9090/metrics | Raw Prometheus metrics |

---

## 📞 Troubleshooting Quick Reference

### Containers won't start?
```bash
docker compose logs  # Check error messages
docker system df     # Check disk space
lsof -i :9090        # Check if port is in use
```

### Targets showing DOWN?
```bash
# Is container running?
docker ps

# Can Prometheus reach it?
docker exec prometheus curl http://notes-app:8000

# Check config has correct hostname
cat prometheus.yml
```

### No metrics data?
```bash
# Wait 15 seconds (first scrape)
sleep 15

# Try simpler query
# up (should always work)

# Check targets are UP
# http://localhost:9090/targets
```

### More issues?
→ Read `SETUP_VERIFICATION.md` Troubleshooting section (comprehensive)

---

## 📊 Architecture: What You Built

```
┌─────────────────────────────────────────────────────┐
│           Your Observability Stack                  │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Data Producers:                                    │
│  ├─ Prometheus (scrapes itself)                    │
│  └─ Notes-App (exposes metrics at :8000/metrics)   │
│                                                     │
│  ↓ (Prometheus pulls every 15 seconds)              │
│                                                     │
│  Metrics Storage:                                   │
│  └─ TSDB (/prometheus directory)                   │
│     └─ Stores time-series data                     │
│     └─ Retention: 30 days or 1GB                   │
│                                                     │
│  ↓ (Query via PromQL)                              │
│                                                     │
│  Visualization:                                     │
│  └─ Prometheus Web UI (:9090)                      │
│     └─ Graphs, tables, alerts                      │
│     └─ (Grafana coming in Day 74)                  │
│                                                     │
└─────────────────────────────────────────────────────┘

Tomorrow: Add Node Exporter + cAdvisor → Full infrastructure monitoring
```

---

## 🎓 Learning Outcomes

After completing Day 73, you should understand:

✅ What observability is and why it matters  
✅ The three pillars: metrics, logs, traces  
✅ How Prometheus works (pull-based scraping)  
✅ Metric types: counter, gauge, histogram, summary  
✅ Labels and time series  
✅ Basic PromQL queries  
✅ rate() function (the most important function!)  
✅ How to configure Prometheus  
✅ Docker Compose for multi-container apps  
✅ Data persistence and retention  

---

## 📈 What's Next?

### Day 74: Complete Infrastructure Monitoring
- Add **Node Exporter** for host metrics
- Add **cAdvisor** for Docker container metrics
- Build **Grafana dashboards** for visualization
- Correlate multiple data sources

### Day 75: Log Aggregation
- Set up **Loki** for log storage
- Configure **Promtail** as log shipper
- Query logs in Grafana
- Understand the "why" pillar

### Day 76: Distributed Tracing
- Configure **OpenTelemetry Collector**
- Instrument applications for tracing
- Visualize request flow across services
- Understand the "where" pillar

### Day 77: Complete Observability
- Build comprehensive Grafana dashboards
- Create alert rules
- **Correlate all three pillars**
- Develop troubleshooting workflows

---

## 🎯 Success Criteria

Complete Day 73 when you can answer these questions:

1. **What is observability?** (More than monitoring)
2. **What are the three pillars?** (Metrics, Logs, Traces)
3. **Why use rate() on counters?** (Convert totals to velocity)
4. **How does Prometheus collect data?** (Pull-based scraping)
5. **What's the difference between counter and gauge?** (Monotonic vs snapshot)
6. **How do you ensure Prometheus data persists?** (Volume mounts)
7. **What's a scrape target?** (An endpoint Prometheus scrapes)
8. **How do labels work?** (Add dimensions to metrics)
9. **When should you use sum(rate()) vs rate(sum())?** (Sum after rate!)
10. **What's TSDB?** (Time-series database where metrics are stored)

**If you can answer all 10, you've mastered Day 73!** 🏆

---

## 📚 Additional Resources

### Official Documentation
- **Prometheus Docs:** https://prometheus.io/docs/
- **PromQL Guide:** https://prometheus.io/docs/prometheus/latest/querying/basics/
- **Best Practices:** https://prometheus.io/docs/practices/naming/

### Reference Repository
- **90 Days of DevOps:** https://github.com/LondheShubham153/90DaysOfDevOps
- **Observability Stack:** https://github.com/LondheShubham153/observability-for-devops

### Next Learning Materials
- These will be added on Day 74 (Grafana setup)
- And Day 75 (Loki configuration)
- And Day 76 (OTEL Collector)

---

## 📝 Share Your Progress

### LinkedIn Post Template

> "Started the observability block today -- learned the three pillars (metrics, logs, traces), set up Prometheus in Docker, wrote my first PromQL queries, and started monitoring a sample app. Observability is what separates running services from actually understanding them.
>
> Day 73/90 complete ✅ 
> 
> #90DaysOfDevOps #DevOpsKaJosh #TrainWithShubham"

### What to Include
- Screenshot of Prometheus Targets page (both UP)
- Screenshot of a PromQL query result
- Brief description of what you learned
- What you'll do next (Node Exporter + cAdvisor)

### Hashtags
- #90DaysOfDevOps
- #DevOpsKaJosh
- #TrainWithShubham
- #Prometheus
- #Observability
- #DevOps
- #Infrastructure
- #Monitoring

---

## 💡 Pro Tips for Learning

1. **Don't memorize PromQL** — Use `PROMQL_REFERENCE.md` as a lookup
2. **Experiment freely** — Prometheus web UI is safe to query anything
3. **Break things intentionally** — Then fix them (best learning!)
4. **Take screenshots** — For your portfolio and LinkedIn
5. **Write your own examples** — Adapt the counter/gauge examples to your work
6. **Explain to others** — Teaching solidifies learning
7. **Build on this foundation** — Days 74-77 stack on Day 73

---

## 🎉 You're Ready!

Everything you need is in this directory:

| File | Purpose | Read Time |
|------|---------|-----------|
| `README.md` | Project overview | 10 min |
| `observability_fundamentals.md` | Observability concepts | 15 min |
| `day-73-observability-prometheus.md` | Complete tutorial | 60 min |
| `SETUP_VERIFICATION.md` | Hands-on setup | 30 min |
| `PROMQL_REFERENCE.md` | Query reference | As needed |
| `prometheus.yml` | Config file | Copy-paste |
| `docker-compose.yml` | Stack config | Copy-paste |

**Start with:** `day-73-observability-prometheus.md`  
**Set up with:** `SETUP_VERIFICATION.md`  
**Reference:** `PROMQL_REFERENCE.md` while querying  

---

## ✨ Final Checklist

Before moving on to Day 74:

- [ ] I understand what observability is
- [ ] I understand the three pillars
- [ ] I know why DevOps engineers need all three
- [ ] Prometheus is running in Docker
- [ ] Both scrape targets are UP
- [ ] I can write basic PromQL queries
- [ ] I understand counter vs gauge
- [ ] I can explain the rate() function
- [ ] My configuration files match the examples
- [ ] I've created documentation for my learning

**All done?** You're ready for Day 74! 🚀

---

**Happy Learning!** 🎓

*Observability is not just about monitoring — it's about understanding your systems deeply enough to ask any question and get an answer from your data.*

---

Generated: July 13, 2025  
Course: 90 Days of DevOps  
Day: 73/90  
Topic: Introduction to Observability and Prometheus
