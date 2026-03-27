# K3s Cluster Architecture

## Overview

A K3s-based monitoring and observability platform deployed on Azure VMs, with SRE Agent integration via MCP protocol.

- Resource Group: k8s (eastasia)
- Network: VNet k3s-vnet (10.0.0.0/16), Subnet k3s-subnet (10.0.1.0/24), NSG k3s-nsg

## VMs

### K3s Cluster Nodes

- **k3s-master** (10.0.1.4, D2s_v6) — K3s control plane
  - Runs: API Server (:6443), etcd, CoreDNS, Traefik (Ingress), metrics-server, local-path-provisioner, Promtail

- **k3s-worker-1** (10.0.1.5, D2s_v6) — K3s worker node
  - Runs: Promtail, application pods (scheduled by K3s)

- **k3s-worker-2** (10.0.1.6, D2s_v6) — K3s worker node
  - Runs: Promtail, application pods (scheduled by K3s)

### Standalone Service VMs

- **prometheus** (10.0.1.7, D2s_v6) — Metrics collection
  - Runs: Prometheus (:9090)
  - Scrapes: K3s API Server, Kubelet, cAdvisor, application metrics

- **loki-server** (10.0.1.8, D2s_v6) — Log storage
  - Runs: Docker → Loki (:3100)
  - Storage: local filesystem, retention 168h (7 days), schema tsdb v13

- **mcp-server** (10.0.1.9, D2s_v5) — MCP protocol gateway
  - Runs: Nginx (:8080, Bearer Token auth) → mcp-grafana (:8000, streamable-http)
  - Provides: Loki log queries, Prometheus metric queries, Dashboard management via MCP protocol

- **grafana-server** (10.0.1.10, D2s_v5) — Visualization
  - Runs: Docker → Grafana (:3000, v12.4.2)
  - Datasources: Loki (10.0.1.8:3100), Prometheus (10.0.1.7:9090)

## Application: Grubify

Deployed in K3s namespace `grubify`. Source: github.com/wttat/wt-sre-agent-demo

- **grubify-api** — .NET 9 Web API backend
  - Image: k8sacr17598.azurecr.io/grubify-api:latest
  - Exposed: NodePort :30081
  - Endpoints: /api/restaurants, /api/cart/{userId}/items, /api/orders, /api/fooditems
  - Known issue: in-memory cart has no eviction, can cause memory leak under load

- **grubify-frontend** — React TypeScript frontend
  - Image: k8sacr17598.azurecr.io/grubify-frontend:latest
  - Exposed: NodePort :30083
  - Connects to API via REACT_APP_API_BASE_URL

## Service Dependencies

### Log Pipeline
- Promtail (runs on every K3s node as DaemonSet) → pushes logs to → Loki (10.0.1.8:3100)

### Metrics Pipeline
- Prometheus (10.0.1.7:9090) → scrapes metrics from → K3s nodes (API Server, Kubelet, cAdvisor)

### Visualization
- Grafana (10.0.1.10:3000) → queries → Loki (10.0.1.8:3100) via LogQL
- Grafana (10.0.1.10:3000) → queries → Prometheus (10.0.1.7:9090) via PromQL

### MCP Integration
- SRE Agent → authenticates with Bearer Token → Nginx on mcp-server (:8080) → proxies to → mcp-grafana (:8000) → calls → Grafana API (10.0.1.10:3000) → queries → Loki / Prometheus

## NSG Rules

| Port | Protocol | Purpose |
|------|----------|---------|
| 22 | TCP | SSH |
| 6443 | TCP | K8s API |
| 10250 | TCP | Kubelet |
| 8472 | UDP | Flannel VXLAN |
| 3000 | TCP | Grafana |
| 3100 | TCP | Loki |
| 8080 | TCP | MCP Server |
| 9090 | TCP | Prometheus |
| 30081 | TCP | Grubify API (NodePort) |
| 30083 | TCP | Grubify Frontend (NodePort) |
