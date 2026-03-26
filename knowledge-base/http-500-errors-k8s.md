# HTTP 500 Error Investigation Runbook

## Trigger Keywords
`500 error`, `internal server error`, `HTTP 500`, `server error`, `application error`, `unresponsive`, `OOMKilled`, `CrashLoopBackOff`

## Scope
Application deployed on Kubernetes cluster. Logs collected by Promtail into Loki, metrics scraped by Prometheus.

## IMPORTANT: Use MCP Tools First
All log queries and metric queries MUST be performed through the Grafana MCP Server tools. Do NOT use kubectl, curl, or direct API calls to Loki/Prometheus. The MCP tools provide authenticated, structured access to all observability data:
- **Logs** → use `query_loki_logs`, `list_loki_label_names`, `list_loki_label_values`
- **Metrics** → use `query_prometheus`, `list_prometheus_label_values`, `list_prometheus_metric_names`
- **Dashboards** → use `search_dashboards`, `get_dashboard_by_uid`

Only fall back to direct access if MCP tools are unavailable.

## MCP Connection
- Endpoint: http://40.83.74.188:8080/mcp
- Transport: streamable-http
- Auth: Bearer Token
- Loki datasource UID: bfh5p2h9cxczkd
- Prometheus datasource UID: cfh5p2u9zvke8c

## Important: Parameter Names
- Loki queries use `logql` parameter (NOT `logQuery`)
- Prometheus queries use `expr` parameter and require `startTime` (e.g., `now-1h`)
- Loki time range uses `startRfc3339` / `endRfc3339`
- Prometheus time range uses `startTime` / `endTime`

---

## Phase 1: Container Memory and CPU Metrics (Check First)

### 1.1 Container Memory Usage
```
Tool: query_prometheus
datasourceUid: cfh5p2u9zvke8c
expr: container_memory_working_set_bytes{namespace="grubify"}
startTime: now-1h
queryType: instant
```

### 1.2 Container CPU Usage
```
Tool: query_prometheus
datasourceUid: cfh5p2u9zvke8c
expr: rate(container_cpu_usage_seconds_total{namespace="grubify"}[5m])
startTime: now-1h
queryType: instant
```

### 1.3 Container Restart Count (OOM Indicator)
```
Tool: query_prometheus
datasourceUid: cfh5p2u9zvke8c
expr: kube_pod_container_status_restarts_total{namespace="grubify"}
startTime: now-1h
queryType: instant
```

### 1.4 Memory Usage Over Time (Range Query)
```
Tool: query_prometheus
datasourceUid: cfh5p2u9zvke8c
expr: container_memory_working_set_bytes{namespace="grubify", container="grubify-api"}
startTime: now-1h
queryType: range
stepSeconds: 60
```

### Resource Thresholds Reference
| Metric | Warning | Critical | Action |
|--------|---------|----------|--------|
| Memory | > 75% of limit | > 90% of limit | Investigate memory leak, restart pod |
| CPU | > 70% sustained | > 90% sustained | Check for hot loops, scale replicas |
| Restarts | > 0 in 1h | > 3 in 1h | Check OOMKilled, CrashLoopBackOff |

---

## Phase 2: Application Logs (Loki)

### 2.1 Recent Error Logs
```
Tool: query_loki_logs
datasourceUid: bfh5p2h9cxczkd
logql: {namespace="grubify"} |= "error" or |= "Error" or |= "exception" or |= "Exception"
limit: 20
```

### 2.2 HTTP 500 Errors in API Logs
```
Tool: query_loki_logs
datasourceUid: bfh5p2h9cxczkd
logql: {namespace="grubify", container="grubify-api"} |= "500"
limit: 20
```

### 2.3 OOM / Memory Pressure Indicators
```
Tool: query_loki_logs
datasourceUid: bfh5p2h9cxczkd
logql: {namespace="grubify"} |~ "(?i)(OutOfMemory|OOM|memory pressure|GC|heap|Cache size)"
limit: 20
```

### 2.4 Stack Traces
```
Tool: query_loki_logs
datasourceUid: bfh5p2h9cxczkd
logql: {namespace="grubify", container="grubify-api"} |~ "(?i)(exception|stack trace|at .*\\()"
limit: 10
```

### 2.5 Cart API Request Volume (Memory Leak Indicator)
```
Tool: query_loki_logs
datasourceUid: bfh5p2h9cxczkd
logql: {namespace="grubify", container="grubify-api"} |= "cart" |= "Added"
limit: 20
```

### 2.6 Cache Size Growth (Specific to Grubify Memory Leak)
```
Tool: query_loki_logs
datasourceUid: bfh5p2h9cxczkd
logql: {namespace="grubify", container="grubify-api"} |= "Cache size"
limit: 20
```

---

## Phase 3: Identify Error Patterns

### 3.1 Error Rate by Log Pattern
```
Tool: query_loki_logs
datasourceUid: bfh5p2h9cxczkd
logql: sum(count_over_time({namespace="grubify", container="grubify-api"} |= "error" [5m]))
limit: 1
```

### 3.2 All Available Namespaces (Verify Scope)
```
Tool: list_loki_label_values
datasourceUid: bfh5p2h9cxczkd
labelName: namespace
```

### 3.3 All Pods in grubify Namespace
```
Tool: list_loki_label_values
datasourceUid: bfh5p2h9cxczkd
labelName: pod
```

### 3.4 Prometheus Target Health
```
Tool: query_prometheus
datasourceUid: cfh5p2u9zvke8c
expr: up
startTime: now-1h
queryType: instant
```

---

## Phase 4: Cluster Health

### 4.1 Node Memory Pressure
```
Tool: query_prometheus
datasourceUid: cfh5p2u9zvke8c
expr: node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100
startTime: now-1h
queryType: instant
```

### 4.2 Pod Status (Running vs Not Running)
```
Tool: query_prometheus
datasourceUid: cfh5p2u9zvke8c
expr: kube_pod_status_phase{namespace="grubify"}
startTime: now-1h
queryType: instant
```

### 4.3 Container OOMKilled Events
```
Tool: query_prometheus
datasourceUid: cfh5p2u9zvke8c
expr: kube_pod_container_status_last_terminated_reason{namespace="grubify", reason="OOMKilled"}
startTime: now-1h
queryType: instant
```

---

## Phase 5: Remediation

### 5.1 Immediate Mitigation
- **Restart the affected pod:** `kubectl rollout restart deployment grubify-api -n grubify`
- This clears in-memory state and restores service

### 5.2 Root Cause: Memory Leak in Cart API
- **File:** `GrubifyApi/Controllers/CartController.cs`
- **Issue:** Static `_carts` dictionary accumulates cart items without eviction
- **Evidence:** "Cache size" log entries showing steady MB growth
- **Fix:** Add TTL-based cache eviction or maximum cart size limit

### 5.3 Long-term Prevention
- Set memory limits on deployment: `resources.limits.memory: 512Mi`
- Configure Prometheus alerting rule for memory threshold
- Add health check endpoint with memory reporting
- Implement cart cleanup/expiry logic in application code

---

## Investigation Summary Template

When reporting findings, use this structure:

```
## Summary
[One-line description of the incident]

## Impact
- Duration: [start time] to [end time]
- Affected services: [list]
- User impact: [description]

## Timeline
- [time] Alert triggered
- [time] Investigation started
- [time] Root cause identified
- [time] Mitigation applied

## Evidence
- [Loki logs showing errors]
- [Prometheus metrics showing memory growth]
- [Cache size progression from logs]

## Root Cause
[Detailed root cause with code reference]

## Remediation
- Immediate: [what was done]
- Long-term: [what should be done]
```
