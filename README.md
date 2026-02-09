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
