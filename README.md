# ELK Stack on Kubernetes — Production + Dedicated Monitoring

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Kubernetes Cluster                           │
│                                                                     │
│  ┌─────────────────────────────┐  ┌──────────────────────────────┐  │
│  │   ns: elk-production        │  │   ns: elk-monitoring         │  │
│  │                             │  │                              │  │
│  │  ┌─────────────────────┐   │  │  ┌──────────────────────┐   │  │
│  │  │ Elasticsearch (3n)  │───┼──┼─▶│ Elasticsearch (3n)   │   │  │
│  │  │  - master/data/ingest│  │  │  │  - monitoring data    │   │  │
│  │  └─────────────────────┘   │  │  └──────────────────────┘   │  │
│  │                             │  │                              │  │
│  │  ┌─────────────────────┐   │  │  ┌──────────────────────┐   │  │
│  │  │ Kibana              │   │  │  │ Kibana (monitoring)  │   │  │
│  │  └─────────────────────┘   │  │  └──────────────────────┘   │  │
│  │                             │  │                              │  │
│  │  ┌─────────────────────┐   │  │  ┌──────────────────────┐   │  │
│  │  │ Logstash (2 replicas)│  │  │  │ Metricbeat (DaemonSet)│  │  │
│  │  └─────────────────────┘   │  │  └──────────────────────┘   │  │
│  │                             │  │                              │  │
│  │  ┌─────────────────────┐   │  │  ┌──────────────────────┐   │  │
│  │  │ Filebeat (DaemonSet)│   │  │  │ Heartbeat            │   │  │
│  │  └─────────────────────┘   │  │  └──────────────────────┘   │  │
│  └─────────────────────────────┘  └──────────────────────────────┘  │
│                                                                     │
│  Metricbeat collects from BOTH namespaces:                          │
│  - Elasticsearch metrics (cluster, node, shard, JVM, thread pools)  │
│  - Kibana metrics                                                   │
│  - Logstash metrics (pipeline throughput, latency, errors)          │
│  - System metrics (CPU, disk, memory)                               │
│  All shipped to elk-monitoring Elasticsearch                        │
└─────────────────────────────────────────────────────────────────────┘
```

## What's Monitored

| Category | Metrics |
|----------|---------|
| **Cluster Health** | Status, node count, CPU, heap, GC, disk I/O, watermarks |
| **Shards** | Unassigned count, relocating, initializing |
| **JVM** | Heap usage, old gen, GC time, GC count |
| **Thread Pools** | search, write, bulk, ingest (active, queue, rejected) |
| **Indexing Latency** | Index rate, index time, search rate, search time |
| **Ingest Pipelines** | Latency, errors, throughput per pipeline |

## Alerts Configured

| Alert | Condition | Severity |
|-------|-----------|----------|
| Unassigned Shards | > 0 for 5 min | Critical |
| JVM Heap > 75% | Sustained 5 min | Warning |
| JVM Heap > 90% | Sustained 2 min | Critical |
| Disk > 80% | Any node | Warning |
| Disk > 90% | Any node | Critical |
| Ingest Errors | > 0 errors/min | Warning |
| Search Latency | p95 > 500ms for 5 min | Warning |

## Quick Start

```bash
# 1. Deploy namespaces
kubectl apply -f namespaces/

# 2. Deploy monitoring cluster first
kubectl apply -f monitoring-cluster/

# 3. Wait for monitoring ES to be ready
kubectl -n elk-monitoring rollout status statefulset/elasticsearch-monitoring

# 4. Deploy production cluster
kubectl apply -f production-cluster/

# 5. Wait for production ES to be ready
kubectl -n elk-production rollout status statefulset/elasticsearch-production

# 6. Deploy beats (monitoring agents)
kubectl apply -f beats/

# 7. Import alert rules & dashboards
kubectl apply -f alerts/
kubectl apply -f dashboards/

# OR use the all-in-one script:
chmod +x scripts/deploy.sh
./scripts/deploy.sh
```

## Access

```bash
# Production Kibana
kubectl -n elk-production port-forward svc/kibana-production 5601:5601

# Monitoring Kibana (dashboards + alerts)
kubectl -n elk-monitoring port-forward svc/kibana-monitoring 5602:5601
```

## Customization

- Adjust resource requests/limits in each StatefulSet
- Modify `storageClassName` to match your cluster's storage class
- Scale replicas as needed
- Update alert thresholds in `alerts/` manifests



# ELK Stack — Next Steps

> **Where you are:** Monitoring Elasticsearch is running ✅
> Now deploy the rest of the stack, step by step.

---

## Step 3: Deploy Monitoring Kibana

```bash
kubectl apply -f monitoring-cluster/02-kibana.yaml
```

Wait for it (takes ~1-2 min):
```bash
kubectl -n elk-monitoring get pods -w
```

Wait until `kibana-monitoring-xxx` shows `1/1 Running`, then `Ctrl+C`.

---

## Step 4: Set up ILM (auto-cleanup of old monitoring data)

```bash
kubectl apply -f monitoring-cluster/03-ilm-policy.yaml
```

Check it completed:
```bash
kubectl -n elk-monitoring get jobs
```

Wait for `monitoring-ilm-setup` → `COMPLETIONS: 1/1`.

If it stays at `0/1`, check logs:
```bash
kubectl -n elk-monitoring logs job/monitoring-ilm-setup
```

---

## Step 5: Deploy Production Elasticsearch

```bash
kubectl apply -f production-cluster/01-elasticsearch.yaml
```

Wait:
```bash
kubectl -n elk-production get pods -w
```

Wait until `elasticsearch-production-0` shows `1/1 Running`.

Quick health check:
```bash
kubectl -n elk-production exec elasticsearch-production-0 -- \
  curl -s http://localhost:9200/_cluster/health?pretty
```

You should see `"status" : "green"`.

---

## Step 6: Deploy Production Kibana

```bash
kubectl apply -f production-cluster/02-kibana.yaml
```

Wait:
```bash
kubectl -n elk-production get pods -w
```

---

## Step 7: Deploy Production Logstash

```bash
kubectl apply -f production-cluster/03-logstash.yaml
```

Wait:
```bash
kubectl -n elk-production get pods -w
```

Wait until `logstash-production-xxx` shows `1/1 Running`.

---

## Step 8: Deploy Production Filebeat

```bash
kubectl apply -f production-cluster/04-filebeat.yaml
```

Verify:
```bash
kubectl -n elk-production get daemonsets
```

You should see `filebeat-production` with `DESIRED: 1`, `READY: 1`.

---

## Step 9: Deploy Metricbeat (monitors everything)

This is the key piece — it collects metrics from both clusters and ships to monitoring ES.

```bash
kubectl apply -f beats/01-metricbeat.yaml
```

Verify:
```bash
kubectl -n elk-monitoring get daemonsets
kubectl -n elk-monitoring logs -l app=metricbeat --tail=20
```

You should see logs like `"Elasticsearch cluster is available"` and `"Connected to backoff"`.

---

## Step 10: Deploy Heartbeat (uptime checks)

```bash
kubectl apply -f beats/02-heartbeat.yaml
```

Verify:
```bash
kubectl -n elk-monitoring get pods | grep heartbeat
```

---

## Step 11: Set up Alert Rules (Watcher)

```bash
kubectl apply -f alerts/01-watcher-alerts.yaml
```

This job waits for ES + metricbeat data, then creates 8 alert rules. It can take 2-3 min.

```bash
kubectl -n elk-monitoring get jobs
kubectl -n elk-monitoring logs job/setup-alert-rules
```

Wait for `COMPLETIONS: 1/1`.

---

## Step 12: Set up Dashboards

```bash
kubectl apply -f dashboards/01-setup-dashboards.yaml
```

```bash
kubectl -n elk-monitoring get jobs
```

---

## Step 13: Access the UIs

Open **two separate terminals**:

**Terminal 1 — Production Kibana** (your app data):
```bash
kubectl -n elk-production port-forward svc/kibana-production 5601:5601
```
→ Open **http://localhost:5601**

**Terminal 2 — Monitoring Kibana** (stack health + alerts):
```bash
kubectl -n elk-monitoring port-forward svc/kibana-monitoring 5602:5601
```
→ Open **http://localhost:5602**

> No login needed — security is disabled for local dev.

---

## Step 14: Verify the Monitoring Stack

### In Monitoring Kibana (http://localhost:5602):

1. **Stack Monitoring** → Click ☰ menu → **Stack Monitoring**
   - You should see both ES clusters, Kibana, Logstash, Beats
2. **Discover** → Create index pattern `metricbeat-*`
   - You should see metrics flowing in
3. **Discover** → Create index pattern `heartbeat-*`
   - You should see uptime checks
4. **Discover** → Create index pattern `elk-alerts`
   - Alert history (may be empty if everything is healthy — that's good!)

### In Production Kibana (http://localhost:5601):

1. **Discover** → Create index pattern `logs-*`
   - You should see container logs flowing in from Filebeat → Logstash → ES

---

## Full Status Check (run anytime)

```bash
echo "========== NAMESPACES =========="
kubectl get ns | grep elk

echo ""
echo "========== MONITORING PODS =========="
kubectl -n elk-monitoring get pods

echo ""
echo "========== PRODUCTION PODS =========="
kubectl -n elk-production get pods

echo ""
echo "========== DAEMONSETS =========="
kubectl -n elk-monitoring get ds
kubectl -n elk-production get ds

echo ""
echo "========== JOBS =========="
kubectl -n elk-monitoring get jobs

echo ""
echo "========== MONITORING ES HEALTH =========="
kubectl -n elk-monitoring exec elasticsearch-monitoring-0 -- \
  curl -s http://localhost:9200/_cluster/health?pretty

echo ""
echo "========== PRODUCTION ES HEALTH =========="
kubectl -n elk-production exec elasticsearch-production-0 -- \
  curl -s http://localhost:9200/_cluster/health?pretty

echo ""
echo "========== WATCHER ALERTS STATUS =========="
kubectl -n elk-monitoring exec elasticsearch-monitoring-0 -- \
  curl -s http://localhost:9200/_watcher/stats?pretty
```

**Expected:** All pods `Running`, both clusters `green`, Watcher `started`.

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| Pod stuck `Pending` | `kubectl describe pod <name> -n <ns>` — usually not enough memory. Give Docker Desktop more RAM. |
| Pod `CrashLoopBackOff` | `kubectl logs <pod> -n <ns>` — check for config errors |
| Metricbeat can't connect | `kubectl -n elk-monitoring logs -l app=metricbeat --tail=50` — check service DNS |
| Jobs stuck at `0/1` | `kubectl -n elk-monitoring logs job/<job-name>` — usually ES not ready yet, delete and reapply |
| No data in Kibana | Wait 2-3 min for first data, then refresh. Check Metricbeat/Filebeat logs. |

---

## What You Get When Done

| Component | Where | What it does |
|-----------|-------|-------------|
| **Production ES** | `elk-production` | Stores application logs |
| **Production Kibana** | http://localhost:5601 | Browse app logs |
| **Logstash** | `elk-production` | Ingests logs from Filebeat |
| **Filebeat** | `elk-production` (DaemonSet) | Collects container logs |
| **Monitoring ES** | `elk-monitoring` | Stores all monitoring metrics + alerts |
| **Monitoring Kibana** | http://localhost:5602 | Stack Monitoring dashboards + alert history |
| **Metricbeat** | `elk-monitoring` (DaemonSet) | Collects metrics from ES, Kibana, Logstash, system |
| **Heartbeat** | `elk-monitoring` | Uptime checks on all endpoints |
| **Watcher Alerts** | `elk-monitoring` | 8 alert rules (shards, JVM, disk, ingest, latency, thread pools) |
