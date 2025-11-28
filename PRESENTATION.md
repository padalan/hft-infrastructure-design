# 30-Minute HFT Infrastructure Deep-Dive
## High-Frequency Trading: DevOps/SRE Perspective

> **TL;DR**: Production crypto trading infrastructure handling 100K+ daily trades across 12+ exchanges. Ultra-low latency (<10ms), 99.99% uptime. Hybrid architecture combining colocated bare-metal (speed) with AWS cloud (reliability).

---

## 📋 Presentation Structure

Problem Context & Requirements

High-Level Architecture

Colocated Infrastructure

AWS Cloud Architecture

Data Flow & Critical Path

DevOps & CI/CD Practices

SRE & Security

Lessons Learned

Ether.fi Connection

Q&A
---

## 🎯 Problem Context & Requirements

### The Business Challenge

| Before | After |
|--------|-------|
| Majority automated trading | Fully Automated trading |
| Seconds of latency | Microseconds latency |
| Missed opportunities | Systematic edge |

### Core Requirements

```
Performance:     <10ms end-to-end (market data -> exchange order)
Reliability:     99.99% uptime (max 1 hours/month downtime)
Safety:          0% position limit breaches (automated controls)
Scale:           100K+ trades/day across 12+ exchanges
Compliance:      Complete audit trail + regulatory requirements
Flexibility:     Multi-strategy support with per-symbol config
```

### Technical Complexity

**Infrastructure Challenges:**
- Multi-exchange coordination (REST APIs, WebSockets, proprietary protocols)
- Nanosecond-level tuning (kernel, networking, CPU scheduling)
- Distributed consistency (positions across geographic regions)
- Network resilience (exchange outages, degradation)
- Automation safety (no human intervention, auto circuit breakers)

**Operational Challenges:**
- 24/7/365 on-call coverage across timezones
- Sub-minute incident response (real money at risk)
- Multiple environments (simulation → paper → production)
- Regulatory compliance requirements

---

## 🏗️ Architecture Overview

### System Design

```
                    ┌──────────────────────────────────┐
                    │  12+ Exchange Feeds              │
                    │  Binance | Coinbase | Gemini     │
                    │  (WebSocket, UDP, REST)          │
                    └────────────────┬─────────────────┘
                                     │
                    ┌────────────────▼──────────────────┐
                    │  COLOCATED BARE-METAL             │
                    │  Equinix: US                      │
                    ├───────────────────────────────────┤
                    │                                   │
                    │  [Market Data] ----──→            │
                    │  (DPDK, <100μs)      |            │
                    │                      │            │
                    │                      ▼            │
                    │               [Strategy Engine]   │
                    │               <1ms decision time  │
                    │                      │            │
                    │  ┌───────────────────▼─────────┐  │
                    │  │        AWS (EKS)            │  │
                    │  │  (Async, 200-500ms OK)      │  │
                    │  └───────────────────┬─────────┘  │
                    │                      │            │
                    │           [Order Gateways]        | 
                    |              |                    |
                    |              |                    |
                    |              ▼                    |
                    |           [Exchanges]             │
                    │                                   │
                    └───────────────────────────────────┘
                                     │
                    ┌────────────────▼──────────────────┐
                    │  AWS CLOUD (Multi-AZ)             │
                    │  us-east-1 Primary Region         │
                    ├───────────────────────────────────┤
                    │                                   │
                    │  Lambda | EC2.metal  │   EKS      │
                    │      │          │         │       │
                    │      └───────-──┘---------|       │
                    │                           │       │
                    │  SQS FIFO ──→ RDS Aurora  │       │
                    │  (Events)  (Trades)       │       │
                    │                           ▼       │
                    │  S3 Archive  │   CloudWatch/Logs  │
                    │  (Compliance)│   (Audit Trail)    │
                    │                                   │
                    └───────────────────────────────────┘
                              ▲              ▲
                              │              │
                    ┌─────────┴──────────────┴──────────┐
                    │  Observability Layer              │
                    │  Prometheus, Loki → Grafana       │
                    │  CloudWatch → AlertManager        │
                    └───────────────────────────────────┘
```

### Architecture Decisions

**Colocated Bare-Metal (Why):**
- ✅ Critical path: market data → strategy → orders (7ms total)
- ✅ Kernel bypass (DPDK): sub-100μs latency
- ✅ CPU-pinned processes: deterministic, no variance
- ✅ Pre-allocated memory: zero GC pauses in hot path
- ❌ Limited redundancy, operational complexity, fixed capacity
- **Decision:** Worth complexity for 5-10x performance gain

**AWS Cloud (Why):**
- ✅ Risk checks: consistency > latency (200-500ms acceptable)
- ✅ Unlimited scalability (Lambda, RDS Multi-AZ)
- ✅ Managed services (backup, security, monitoring)
- ✅ Infrastructure as code (repeatable, auditable)
- ❌ Higher latency (unacceptable for critical path)
- **Decision:** Separate concerns, cloud handles what it's good at

---

## 💾 Colocated Infrastructure Deep-Dive

### Hardware Setup

```
┌──────────────────────────────────────────────────────┐
│ Bare-Metal Server Specifications                     │
├──────────────────────────────────────────────────────┤
│ CPU:      Intel Xeon W9-3495X (56-core, 4.8GHz boost)│
│ RAM:      256GB DDR5 ECC (low-latency tuned)         │
│ Network:  25Gbps SFP28 NIC (Solarflare XtremeScale)  │
│ Storage:  2x 3.2TB NVMe U.2 (RAID0)                  │
│ Latency:  80–150ns NIC tick-to-trade                 │
└──────────────────────────────────────────────────────┘
```

### Kernel Optimization

**Network Performance:**
```bash
net.core.rmem_max=134217728        # 128MB RX buffer
net.core.wmem_max=134217728        # 128MB TX buffer
net.core.netdev_max_backlog=5000   # NIC ring size
net.ipv4.tcp_tw_reuse=1            # Reuse TIME_WAIT
net.ipv4.tcp_sack=0                # Disable SACK overhead
```

**CPU Scheduling:**
```bash
kernel.sched_migration_cost_ns=5000000    # Less migration
kernel.sched_latency_ns=1000000           # Lower latency
kernel.sched_min_granularity_ns=100000    # Fine-grained scheduling
```

**System Stability:**
```bash
vm.swappiness=0                    # NEVER swap
kernel.panic_on_oops=1             # Crash on errors
kernel.core_uses_pid=1             # Generate core dumps
```

### CPU Pinning

| Process | Core | Priority | Purpose |
|---------|------|----------|---------|
| Market Data Collector | 0-2 | High | Async ingest |
| **Strategy Engine** | **3** | **Highest** | **Critical** |
| Order Gateways | 4-7 | High | Exchange comms |
| Monitoring | 8 | Low | Observability |
| Kernel | 9-27 | Varies | System tasks |

**Strategy Core Isolation:**
```bash
# Move all interrupts away from core 3
$ echo 1 > /sys/kernel/irq/*/smp_affinity

# Pin strategy to core 3
$ taskset -p -c 3 <pid>

# Result: 100% CPU time, no context switches
```

### OpenOnload (Kernel Bypass)

| Aspect | Traditional | DPDK |
|--------|-------------|------|
| Packet path | NIC → Kernel → App | NIC → App (direct) |
| Per-packet latency | 50-100μs | <10μs |
| Overhead | OS context switches | Direct memory |
| Savings | — | **100-150μs per trade** |

---

## ☁️ AWS Cloud Architecture

### EC2 Metal
```
┌──────────────────────────────────────────────────────────┐
│ Bare-Metal Server Specifications                         │
├──────────────────────────────────────────────────────────┤
│ CPU:      AMD EPYC 9654 (96-core, 3.7GHz boost)          │
│ RAM:      256GB DDR5                                     │
│ Network:  25–100Gbps NIC (Mellanox ConnectX-6 Dx)        │
│ Storage:  2x 1.92TB NVMe (RAID0)                         │
│ Latency:  1–3µs NIC RTT; 2–8µs user-space packet path    │
└──────────────────────────────────────────────────────────┘
```

### Microservices Components

```
┌──────────────────────────────────────────────┐
│ AWS Services Layer (Multi-AZ)                │
├──────────────────────────────────────────────┤
│                                              │
│  [S1, S2, S3]    [S4, S5, S6, S7]            │
│  Lambda            EKS                       │
│  512MB | 1000x     2-10 tasks                │
│  ├─ Position limits    ├─ Aggregation        │
│  ├─ Leverage check     ├─ P&L calculation    │
│  ├─ Concentration      ├─ Reporting          │
│  └─ Drawdown limits    └─ Balance mgmt       │
│       │                     │                │
│       └──────────┬──────────┘                │
│                  │                           │
│          [SQS FIFO Queue]                    │
│          Event bus (ordered)                 │
│          14-day retention                    │
│                  │                           │
│      ┌───────────┼───────────┐               │
│      ▼           ▼           ▼               │
│  [RDS Aurora] [TimescaleDB] [S3 Archive]     │
│  Multi-AZ     Time-series   Compliance       │
│  PostgreSQL   Metrics       Audit Logs       │
│  (Durable)    (Analytics)   (Long-term)      │
│                                              │
└──────────────────────────────────────────────┘
```

### Service Responsibilities

**Risk Service (Lambda):**
- Position limits (long + short + pending)
- Leverage ratio (borrowed capital)
- Concentration risk (% of portfolio per symbol)
- Daily drawdown thresholds
- Correlation risk (multi-exchange)
- **Latency:** 200-500ms (acceptable, not critical)
- **SLA:** 99.9% (fallback to cached decision)

**Portfolio Management (ECS Fargate):**
- Aggregate positions across strategies
- Real-time P&L (mark-to-market)
- Exposure reporting (per-exchange, per-symbol)
- Balance management (funding, withdrawals)
- **Latency:** 10-100ms (monitoring, not critical)
- **Scaling:** Auto-scale 1-10 instances

**Persistence (RDS Aurora):**
- trades (100M+ rows, 1TB+)
- positions (10K rows)
- orders (50M+ rows)
- risk_config (per-account)
- audit_log (immutable, 10B+ rows)
- **HA:** Multi-AZ, 30-day backups

---

## ⚡ Data Flow & Critical Path

### The 7-Millisecond Journey

| Time | Component | Action | Latency |
|------|-----------|--------|---------|
| 0μs | Exchange | Price published | — |
| 20μs | Market Handler | Receive | 20μs |
| 50μs | Ring Buffer | Enqueue (lock-free) | 30μs |
| 100μs | Strategy | Read buffer | 50μs |
| 200μs | Greeks | Calculate | 100μs |
| 300μs | Signal | Generate | 100μs |
| 500μs | gRPC (async) | Risk check sent | 200μs |
| 700μs | Formatter | Build order | 200μs |
| 1000μs | Fallback | Use cache if timeout | 300μs |
| 1500μs | Submit | Send to exchange | 500μs |
| **External** | **Exchange** | **Process** | **3-50ms** |

**Total: <2ms internal | 3-50ms external | P95: <10ms**

### Latency Breakdown

**What's Fast (<1ms):**
- Market data ingestion (Solarflare OpenOnload)
- Ring buffer IPC (shared memory)
- Strategy calculation (pre-compiled)
- Order formatting (templates)

**What's Acceptable (<500ms):**
- Risk service (Lambda, async)
- RDS persistence (durable)
- SQS event publishing

**Fallback Strategy:**
- Risk timeout? → Use cached decision
- Exchange down? → Circuit breaker stops orders
- Position mismatch? → Manual review + stop

---

## 🔧 DevOps & CI/CD Practices

### Testing Strategy (3-Tier Pyramid)

```
        ┌─────────────────────────┐
        │   Production (Real $)   │  1% bugs
        │   Small positions       │  Real incidents
        │   1-2 week rollout      │
        └─────────────────────────┘
            ▲
            │
        ┌─────────────────────────┐
        │  Paper Trading (Live)   │  10% bugs
        │  No money, 2-3 days     │  Integration
        └─────────────────────────┘
            ▲
            │
        ┌─────────────────────────┐
        │  Simulation (Replay)    │  90% bugs
        │  5+ years history       │  Unit + replay
        │  1 hour runtime         │
        └─────────────────────────┘
```

### CI/CD Pipeline (7 Phases)

**Phase 1: Code Quality (3 min)**
- Lint: ruff, black, mypy
- Unit tests: pytest with coverage
- Result: 90% of bugs caught

**Phase 2: Integration Tests (5 min)**
- Database, Redis, services
- Real environment simulation
- Result: 8% of bugs caught

**Phase 3: Replay Tests (1+ hour)**
- 5+ years market data
- Historical scenarios
- Result: 2% of bugs caught

**Phase 4: Security Scanning**
- Dependency check (Snyk)
- Static analysis (Bandit)

**Phase 5: Build & Push (5 min)**
- Docker images
- ECR push (OIDC auth)

**Phase 6: Deploy Staging (5 min)**
- Terraform apply
- Smoke tests

**Phase 7: Deploy Production (5 min)**
- Blue/green deployment
- Auto-rollback on errors

**Total Time:** 20-30 minutes code → production

### Infrastructure as Code

**Terraform (AWS):**
```
terraform/
├─ main.tf, vpc.tf, rds.tf
├─ lambda.tf, iam.tf
├─ monitoring.tf
└─ environments/
   ├─ staging.tfvars
   └─ production.tfvars
```

**Ansible (Bare-Metal):**
```
ansible/
├─ playbooks/
│  ├─ bootstrap.yml
│  ├─ kernel-tuning.yml
│  ├─ dpdk-setup.yml
│  └─ monitoring.yml
└─ inventory/
   ├─ singapore.ini
   ├─ tokyo.ini
   └─ us-east.ini
```

**Result:**
- ✅ Everything version-controlled
- ✅ Entire stack deployed in 30 minutes
- ✅ Staging = Production (exact replicas)
- ✅ Disaster recovery fully automated

---

## 📊 SRE & Security

### Observability Stack

```
Prometheus/Loki (collect)
├─ 15-sec scrape intervals
├─ 200+ metrics
└─ Latency, errors, resources

    ↓

Grafana (visualize)
├─ System health dashboard
├─ Trading metrics dashboard
├─ Latency heatmaps
└─ Incident timeline

    ↓

AlertManager (alert)
├─ P99 latency > 50ms
├─ Error rate > 1%
├─ Position mismatch
├─ Exchange connectivity loss
└─ → Slack, PagerDuty, Runbook
```

### Incident Response (MTTR)

```
Alert Fires → Investigation → Fix → Deploy → Recovery
(instant)    (1-2 min)      (3-5) (5 min)   (5-10 min)

Total: 15-30 minutes

Keys to fast MTTR:
├─ Dashboards (understand story in 1 min)
├─ Runbooks (copy-paste solutions)
├─ Circuit breakers (contain impact)
├─ Auto-rollback (recover <5 min)
└─ On-call rotation (prevent burnout)
```

### Security: 5 Layers

**Layer 1: Authentication**
- ✅ GitHub OIDC (no static credentials)
- ✅ Temporary role assumption (15-min TTL)
- ✅ API keys rotated quarterly
- ✅ Secrets in AWS Secrets Manager

**Layer 2: Authorization**
- ✅ IAM least-privilege (per service)
- ✅ VPC private subnets (Lambda, RDS)
- ✅ Security groups (explicit rules)
- ✅ Network ACLs (between subnets)

**Layer 3: Data Protection**
- ✅ RDS encryption at rest (KMS)
- ✅ TLS 1.3 in transit
- ✅ S3 versioning (recover deleted)
- ✅ Audit logs immutable (append-only)

**Layer 4: Runtime Security**
- ✅ gRPC + mTLS (service auth)
- ✅ Input validation (exchange data)
- ✅ Rate limiting (APIs)
- ✅ Unprivileged containers (no root)

**Layer 5: Compliance & Audit**
- ✅ CloudTrail (all AWS calls)
- ✅ VPC Flow Logs (network traffic)
- ✅ Application logs (with PII redaction)
- ✅ 10B+ audit entries (who, what, when)

---

## 💡 Lessons Learned

### What Went Well ✅

**Architecture Split** (Colocated + AWS)
- Hit <10ms latency target
- Risk service scaled independently
- Different strategies per layer
- Prevented cascade failures

**Observability First**
- Caught incidents <1 minute
- Root cause in <5 minutes
- Dashboards told the story
- MTTR: hours → minutes

**Infrastructure as Code**
- Reproducible (staging = production)
- Disaster recovery (30 min rebuild)
- Version control for infrastructure
- Code review before deployment

**Testing Strategy**
- 80% bugs caught before production
- Replay tests caught race conditions
- Paper trading validated strategies
- Production incidents: 5% of bugs

**Automation Over Heroics**
- Circuit breakers contained failures
- Auto-rollback prevented impact
- Runbooks made response consistent
- Solo incident handling (no escalation)

### Improvements Needed 🔄

**Chaos Engineering** → Weekly tests (kill service, delay network)

**Blast Radius** → Per-strategy sandboxing (resource limits)

**On-Call Workload** → Dedicated SRE rotation (not dev)

**Design** → Microservices from day 1 (not refactored later)

### Core SRE Principles

```
1. OBSERVABILITY FIRST
   Measure everything → Alert on SLOs

2. IDEMPOTENCY BY DEFAULT
   Every op survives failure → Unique IDs + query-after-failure

3. AUTOMATION OVER HEROICS
   Humans don't scale → Circuit breakers, auto-rollback

4. FAILURE IS INEVITABLE
   Design for degradation → Fallbacks, <5 min MTTR

5. SEPARATION OF CONCERNS
   Latency ≠ Consistency → Colocated for speed, cloud for reliability

6. SECURITY IN LAYERS
   One breach ≠ compromise → Defense in depth
```

---

## 🎯 Connection to Ether.fi

### ETher.fi Challenges Match HFT

| HFT | Ether.fi |
|-----|----------|
| Money in motion (trading) | Money in motion (staking) |
| Exchange coordination | Protocol coordination |
| Settlement risk | Cross-chain risk |
| Regulatory audit trail | User-facing reliability |

### Patterns That Apply

- Hybrid architecture
- Idempotent operations
- Comprehensive monitoring
- Security in layers 
- Infrastructure as code
- Automated Incident response
- CI/CD

### Technologies in Common

```
Cloud:          AWS
IaC:            Terraform, Ansible
Deployment:     GitHub Actions, blue/green
Observability:  Prometheus, Grafana, CloudWatch
```

---

## ✅ Key Takeaways

**This Infrastructure Demonstrates:**
- ✅ Hybrid design (optimize each layer)
- ✅ DevOps excellence (fully automated)
- ✅ SRE best practices (observability + resilience)
- ✅ Security depth (layered defense)
- ✅ Operational maturity (sub-minute response)

**Principles Applicable Everywhere:**
1. Measure everything
2. Automate everything
3. Design for failure
4. Separate concerns
5. Layer your security

---

## Q&A


### **LinkedIn Articles on Ultra-Low-Latency Engineering**

* **[Memory Tuning on Linux for Ultra-Low-Latency Trading](https://www.linkedin.com/pulse/memory-tuning-linux-ultra-low-latency-trading-nikhil-goud-ckp8c/)**
* **[Storage I/O Tuning on Linux for Ultra-Low-Latency Trading](https://www.linkedin.com/pulse/storage-io-tuning-linux-ultra-low-latency-trading-nikhil-goud-fiwic/)**
* **[Achieving Ultra-Low Latency in Trading - A Practical Guide for Engineers](https://www.linkedin.com/pulse/achieving-ultra-low-latency-trading-guide-engineers-nikhil-goud-6cfjc/)**
* **[CPU Optimization on Linux for Ultra-Low-Latency Trading](https://www.linkedin.com/pulse/cpu-optimization-linux-ultra-low-latency-trading-nikhil-goud-5rezc/)**

