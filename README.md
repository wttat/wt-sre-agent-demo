# Grubify - Food Delivery Demo App

A modern food delivery application built with React TypeScript frontend and .NET 9 Web API backend, deployed on a self-managed K3s cluster with full observability stack.

## Application

### Backend API (.NET 9)
- **Tech:** ASP.NET Core Web API
- **Endpoints:**
  - `GET /api/restaurants` — Restaurant listings
  - `GET /api/fooditems` — Menu items
  - `POST /api/cart/{userId}/items` — Add to cart
  - `GET /api/orders` — Order history

### Frontend (React TypeScript)
- **Tech:** React 18 + TypeScript + Material-UI
- **Features:** Browse restaurants, add to cart, checkout, order tracking

## Deployment

### Prerequisites
- K3s cluster
- Azure Container Registry (for image builds)
- Loki + Prometheus + Grafana (observability)

### Build Images (ACR)
```bash
# Backend
az acr build -r <acr-name> -t grubify-api:latest --file GrubifyApi/Dockerfile GrubifyApi/

# Frontend
az acr build -r <acr-name> -t grubify-frontend:latest \
  --build-arg REACT_APP_API_BASE_URL=http://<api-endpoint>/api \
  --file grubify-frontend/Dockerfile grubify-frontend/
```

### Deploy to K3s
```bash
kubectl create namespace grubify

# Create ACR pull secret
kubectl create secret docker-registry acr-secret \
  --docker-server=<acr>.azurecr.io \
  --docker-username=<user> \
  --docker-password=<pass> \
  -n grubify

# Deploy API
kubectl create deployment grubify-api \
  --image=<acr>.azurecr.io/grubify-api:latest \
  -n grubify
kubectl set env deployment/grubify-api \
  ASPNETCORE_URLS=http://+:8080 \
  AllowedOrigins__0=http://<frontend-url> \
  -n grubify
kubectl expose deployment grubify-api --type=NodePort --port=80 --target-port=8080 -n grubify

# Deploy Frontend
kubectl create deployment grubify-frontend \
  --image=<acr>.azurecr.io/grubify-frontend:latest \
  -n grubify
kubectl expose deployment grubify-frontend --type=NodePort --port=80 --target-port=80 -n grubify
```

## Observability

### Log Collection
- **Promtail** (DaemonSet on all K3s nodes) collects container logs
- Pushes to **Loki** for storage and querying via LogQL

### Metrics
- **Prometheus** scrapes K3s API Server, Kubelet, cAdvisor
- Container memory, CPU, restart counts available

### Visualization & Alerting
- **Grafana** connects to both Loki and Prometheus as datasources
- Alert rules monitor application health (e.g., HTTP errors, resource usage)

## Project Structure

```
├── GrubifyApi/              # .NET 9 Web API backend
│   ├── Controllers/         # API controllers
│   ├── Models/              # Data models
│   ├── Dockerfile           # Backend container image
│   └── Program.cs           # App entry point
├── grubify-frontend/        # React TypeScript frontend
│   ├── src/                 # React components & pages
│   ├── nginx.conf           # Nginx config for SPA
│   └── Dockerfile           # Frontend container image
├── knowledge-base/          # Operations documentation
│   ├── architecture.md      # System architecture
│   └── http-500-errors-k8s.md  # Troubleshooting runbook
├── infra/                   # Azure Bicep templates (reference)
└── scripts/                 # Utility scripts
```

## License

Based on [Grubify](https://github.com/dm-chelupati/grubify).
