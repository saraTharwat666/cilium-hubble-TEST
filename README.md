![Architecture](https://imgs.search.brave.com/msFHxgewWk-UT_jwcHfNoGHw3S_WpRGHlSGk-A7gp8I/rs:fit:860:0:0:0/g:ce/aHR0cHM6Ly9naXRo/dWIuY29tL2NpbGl1/bS9odWJibGUvcmF3/L21haW4vRG9jdW1l/bnRhdGlvbi9pbWFn/ZXMvaHViYmxlX2Fy/Y2gucG5n)

# Cilium & Hubble Lab 🐝

A Kubernetes security and observability lab using Cilium as CNI and Hubble for real-time traffic visibility.

## Architecture

```
[Frontend] ──► [Backend] ──► [Redis]
```

- **Frontend**: traefik/whoami — returns request info
- **Backend**: kennethreitz/httpbin — returns HTTP responses
- **Redis**: redis:7-alpine — in-memory data store

## Project Structure

```
cilium-hubble-lab/
├── kind-config.yaml
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
```

## What This Lab Demonstrates

- Cilium as a CNI replacing the default kindnet
- eBPF-based networking instead of kube-proxy
- Zero-trust Network Policies controlling traffic between services
- Hubble UI for real-time traffic visualization and policy verification

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
cilium status --wait
```

### 3. Load Images

> kind v0.33+ has issues loading multi-platform images directly.
> Always pull with `--platform linux/amd64` first.

```bash
docker pull --platform linux/amd64 docker.io/traefik/whoami:latest
docker pull --platform linux/amd64 docker.io/kennethreitz/httpbin:latest
docker pull --platform linux/amd64 docker.io/library/redis:7-alpine

kind load docker-image docker.io/traefik/whoami:latest --name cilium-lab
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

### 6. Verify Everything is Running

```bash
kubectl get pods
kubectl get networkpolicies
cilium status
```

### 7. Open Hubble UI

```bash
cilium hubble ui
```

## Images Used

| Service  | Image                       | Port |
|----------|-----------------------------|------|
| Frontend | traefik/whoami:latest       | 80   |
| Backend  | kennethreitz/httpbin:latest | 80   |
| Redis    | redis:7-alpine              | 6379 |

## Network Policies

| Source   | Destination | Allowed | Reason                     |
|----------|-------------|---------|----------------------------|
| Frontend | Backend     | ✅      | Allowed by frontend-policy |
| Backend  | Redis       | ✅      | Allowed by backend-policy  |
| Frontend | Redis       | ❌      | No direct DB access        |
| Any Pod  | Any         | ❌      | Default deny all           |
| External | Frontend    | ✅      | Public facing service      |

## Viewing Traffic in Hubble

Use these filters in the Hubble UI search bar:

```
verdict:dropped              # show blocked traffic only
verdict:forwarded            # show allowed traffic only
source:default/frontend      # show frontend traffic
destination:default/redis    # show redis traffic
```

Or use the Hubble CLI:

```bash
# watch all flows live
hubble observe

# watch dropped flows only
hubble observe --verdict DROPPED

# watch traffic from frontend only
hubble observe --from-label app=frontend
```

## Testing the Policies

Run a debug pod and try to reach each service:

```bash
kubectl run attacker \
  --image=docker.io/library/redis:7-alpine \
  --rm -it --restart=Never \
  -- sh -c "
    echo '=== Trying Redis (should be DROPPED) ===' &&
    redis-cli -h redis -p 6379 PING &&
    echo '=== Trying Backend (should be DROPPED) ===' &&
    wget -qO- backend:80 &&
    echo '=== Trying Frontend (should be FORWARDED) ===' &&
    wget -qO- frontend:80
  "
```

Watch the results live in Hubble UI — blocked traffic shows in 🔴 red, allowed in ⬜ grey.

## Known Issues

- kind v0.33+ has issues loading multi-platform images directly
- Solution: use `--platform linux/amd64` when pulling images
- If `imagePullPolicy: Never` causes `ErrImageNeverPull`, verify the image name matches exactly what is inside the cluster using:

```bash
docker exec cilium-lab-control-plane crictl images
```
