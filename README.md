# DevOps / SRE Trial Task — Submission

## Overview

This repository evolves the base infrastructure provided in Feegow’s challenge, focusing on **observability**, **reliability**, and **operational quality**.  
The approach taken was **incremental and signal-driven**, validating each layer (infra → metrics → logs → alerts) before moving on to load, SLOs, and automation.

### Key deliverables

- Working observability (**Prometheus, Grafana, Loki**)
- Complete application dashboards (RPS, errors, latency per route)
- Alert noise reduction
- Log-based alerting (Loki) with route correlation
- Alerts provisioned via **IaC** (no UI dependency)
- Documentation of operational decisions and trade-offs

---

## Setup and initial validation

### 1. Base infrastructure

```bash
make up
```

**Validations performed:**

- Reusable `kind` cluster created  
- Working Ingress at `http://dev.local`  
- Observability stack running (Prometheus, Grafana, Loki, Tempo)  
- Datasources and dashboards automatically provisioned in Grafana  
- Alertmanager configured via file  

---

### 2. Application deploy

```bash
make deploy
```

**Available applications:**

- **Frontend:** `http://dev.local/`  
- **API v1 (Python):** `/v1/appoints/available-schedule`  
- **API v2 (Go):** `/v2/appoints/available-schedule`  

---

## Required technical adjustments

### Metrics API (HPA)

During the initial validation, `kubectl top pods` failed due to the **Metrics API** being unavailable.

**Action taken:**

- Fixed the `metrics-server` installation in the cluster

**Result:**

- CPU and memory metrics available  
- HPAs working correctly  
- Solid baseline for autoscaling and alerting  

---

### Loki failing to start (Ruler)

The **Loki Ruler** failed to start because the chart mounts `/etc/loki` via `Secret` as **read-only**, preventing creation of the rules directory.

**Action taken:**

- Updated `ruler.storage` to use `/tmp/loki/rules`  
- Mounted rules via `ConfigMap` at that path  
- Separated `config.file` and `rule_path`  

**Result:**

- Loki starts correctly  
- Ruler is active and continuously evaluating rules  
- Log alerts fully provisioned via IaC  

---

## Observability — Metrics (Prometheus + Grafana)

### Application dashboard

**Updated file:**

- `dashboards/grafana/app-latency.json`

**Panels implemented:**

- RPS (req/s)  
- 5xx error rate (%)  
- p95 latency per route  
- 5xx percentage per route  

---

## Observability — Logs (Loki)

### Ingestion validation

```logql
{namespace="apps"}
```

---

### Log-based alert — `LogsErrorBurst`

```logql
sum by (app, route) (
  count_over_time(
    {namespace="apps"} |= "" 5"
    | regexp ""(GET|POST|PUT|DELETE|PATCH) (?P<route>/[^ ]+) HTTP/1\.1" (?P<status>5\d\d)"
  [5m])
) > 5
```

## CI/CD — Unit tests

The workflow runs unit tests for **Python (pytest)** and **Go (go test)** in separate jobs, failing the pipeline via exit code if there is an error.  
This reduces regression risk and speeds up feedback by validating logic before build/deploy.  
It can also be reproduced locally with `act`, avoiding push cycles just to validate CI.

### Run CI locally (act)

```bash
act -j unit-tests-python -P ubuntu-latest=ghcr.io/catthehacker/ubuntu:act-latest
act -j unit-tests-go -P ubuntu-latest=ghcr.io/catthehacker/ubuntu:act-latest
```

### Local CI

```bash
cd apps/available-schedules-go && go test ./... -v
cd apps/available-schedules-python && pytest -q
```

---

## Decisions & trade-offs

- I didn’t use k6 at first to validate signals first  
- I fixed the Metrics API before working on HPAs  
- Alerting is based on HTTP status codes, not log level  
- `ticket` severity for logs
