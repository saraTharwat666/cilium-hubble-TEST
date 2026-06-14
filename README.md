# Cilium & Hubble Lab 🐝

A Kubernetes security and observability lab using Cilium as CNI and Hubble for traffic visibility.

## Architecture

[Frontend] ──► [Backend] ──► [Redis]


cilium-hubble-lab/
├── kind-config.yaml        # Kind cluster config
├── manifests/
│   ├── frontend/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── backend/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── redis/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── policies/
│       ├── frontend-policy.yaml
│       ├── backend-policy.yaml
│       └── redis-policy.yaml
└── README.md






## What This Lab Demonstrates

- Cilium as a CNI replacing the default kindnet
- eBPF-based networking instead of kube-proxy
- Network Policies controlling traffic between services
- Hubble UI for real-time traffic visualization

## Prerequisites

- Docker
- kind
- kubectl
- Cilium CLI
- Hubble CLI

## Quick Start

### 1. Create the Cluster
```bash
kind create cluster --config kind-config.yaml --name cilium-lab
```

### 2. Install Cilium
```bash
cilium install --version 1.15.5
cilium hubble enable --ui
```

### 3. Load Images
```bash
docker pull --platform linux/amd64 docker.io/library/nginx:alpine
docker pull --platform linux/amd64 docker.io/kennethreitz/httpbin:latest
docker pull --platform linux/amd64 docker.io/library/redis:7-alpine

kind load docker-image docker.io/library/nginx:alpine --name cilium-lab
kind load docker-image docker.io/kennethreitz/httpbin:latest --name cilium-lab
kind load docker-image docker.io/library/redis:7-alpine --name cilium-lab
```

### 4. Deploy Services
```bash
kubectl apply -f manifests/redis/
kubectl apply -f manifests/backend/
kubectl apply -f manifests/frontend/
```

### 5. Apply Network Policies
```bash
kubectl apply -f manifests/policies/
```

### 6. Open Hubble UI
```bash
cilium hubble ui
```

## Network Policies

| Source    | Destination | Allowed |
|-----------|-------------|---------|
| Frontend  | Backend     | ✅      |
| Backend   | Redis       | ✅      |
| Frontend  | Redis       | ❌      |
| Any       | Any         | ❌      |

## Viewing Traffic in Hubble

Use these filters in the Hubble UI:

- `verdict:dropped` - Show blocked traffic
- `source:default/frontend` - Show frontend traffic
- `destination:default/redis` - Show Redis traffic