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

## Known Limitations
- `container_memory_working_set_bytes{namespace="grubify"}` instant query may return empty. Use `container="grubify-api"` filter + range query with `stepSeconds` to get data.
- `kube_pod_container_status_restarts_total` and other `kube_*` metrics are NOT available (kube-state-metrics is not deployed). Check pod restart info via Loki logs instead: `{namespace="grubify"} |~ "(?i)(restart|started|back-off)"`.
- `kube_pod_status_phase` and `kube_pod_container_status_last_terminated_reason` are also unavailable for the same reason.

---

## Phase 1: Container Memory and CPU Metrics (Check First)

### 1.1 Container Memory Usage
```
Tool: query_prometheus
datasourceUid: cfh5p2u9zvke8c
expr: container_memory_working_set_bytes{namespace="grubify", container="grubify-api"}
startTime: now-1h
queryType: range
stepSeconds: 60
```

### 1.2 Container CPU Usage
```
Tool: query_prometheus
datasourceUid: cfh5p2u9zvke8c
expr: rate(container_cpu_usage_seconds_total{namespace="grubify", container="grubify-api"}[5m])
startTime: now-1h
queryType: range
stepSeconds: 60
```

### 1.3 Memory Usage Trend (Detect Growth)
```
Tool: query_prometheus
datasourceUid: cfh5p2u9zvke8c
expr: container_memory_working_set_bytes{namespace="grubify", container="grubify-api"}
startTime: now-30m
queryType: range
stepSeconds: 30
```

### Resource Thresholds Reference
| Metric | Warning | Critical | Action |
|--------|---------|----------|--------|
| Memory | > 75% of limit | > 90% of limit | Investigate memory issue, restart pod |
| CPU | > 70% sustained | > 90% sustained | Check for hot loops, scale replicas |

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

### 2.5 High-Volume API Request Patterns
```
Tool: query_loki_logs
datasourceUid: bfh5p2h9cxczkd
logql: {namespace="grubify", container="grubify-api"} |= "Added"
limit: 20
```

### 2.6 Cache / Memory Growth Indicators
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

### 4.2 Pod Restart / Crash Evidence (via Loki)
Since kube-state-metrics is not deployed, check for restart indicators in logs:
```
Tool: query_loki_logs
datasourceUid: bfh5p2h9cxczkd
logql: {namespace="grubify"} |~ "(?i)(restart|started|back-off|OOMKilled|CrashLoop)"
limit: 20
```

### 4.3 Container Process Start Events
```
Tool: query_loki_logs
datasourceUid: bfh5p2h9cxczkd
logql: {namespace="grubify", container="grubify-api"} |= "Content root path"
limit: 10
```
Note: .NET apps log "Content root path" on startup. Multiple entries in short time = restarts.

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
- [Prometheus metrics showing resource usage]
- [Application log patterns]

## Root Cause
[Detailed root cause with code reference]

## Remediation
- Immediate: [what was done]
- Long-term: [what should be done]
```
