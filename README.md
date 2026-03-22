# gpu-infra-lab

Multi-node Kubernetes cluster with NVIDIA GPU scheduling, observability, and GitOps deployment.

Built as a homelab to learn and demonstrate AI infrastructure engineering patterns used in production GPU clouds like Nscale and CoreWeave.

## Hardware

| Node                        | Role                           | Specs                                                          |
| --------------------------- | ------------------------------ | -------------------------------------------------------------- |
| controlplane (Raspberry Pi) | K3s server, ArgoCD, monitoring | ARM64, 4GB RAM                                                 |
| gpu-node (PC)               | K3s agent, GPU workloads       | Intel i7-7740X, 16GB RAM, RTX 2060 Super 8GB, Ubuntu 24.04 LTS |

## Stack

| Layer         | Technology                                                |
| ------------- | --------------------------------------------------------- |
| Kubernetes    | K3s — lightweight, multi-arch (ARM64 + AMD64)             |
| GitOps        | ArgoCD — App of Apps pattern                              |
| GPU           | NVIDIA GPU Operator v26.3.0, DCGM Exporter                |
| Observability | Prometheus, Grafana, Node Exporter                        |
| Workloads     | Ollama — LLM inference on GPU                             |
| CI/CD         | GitHub Actions — bootstrap, teardown, manifest validation |

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Home Network                         │
│                                                             │
│  ┌─────────────────────┐       ┌─────────────────────────┐  │
│  │    controlplane     │       │       gpu-node          │  │
│  │    (Pi, ARM64)      │◄─────►│       (PC, AMD64)       │  │
│  │                     │  K3s  │                         │  │
│  │  K3s Server         │       │  K3s Agent              │  │
│  │  ArgoCD             │       │  NVIDIA GPU Operator    │  │
│  │  Prometheus         │       │  DCGM Exporter          │  │
│  │  Grafana            │       │  Ollama (LLM server)    │  │
│  │  Node Exporter      │       │  RTX 2060 Super 8GB     │  │
│  └─────────────────────┘       └─────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

The Pi handles cluster management, GitOps, and observability. The PC handles GPU workloads. This mirrors how production AI clusters separate control plane concerns from compute.

## Repository structure

```
gpu-infra-lab/
├── .github/workflows/
│   ├── bootstrap.yml       # Deploys cluster from scratch via GitHub Actions
│   ├── teardown.yml        # Destroys cluster (reverse of bootstrap)
│   ├── validate.yml        # Validates manifests on every PR
│   └── build-push.yml      # Builds and pushes custom images
├── argocd/applications/
│   ├── gpu-operator.yaml   # ArgoCD App — NVIDIA GPU Operator (Helm)
│   ├── monitoring.yaml     # ArgoCD App — Prometheus + Grafana stack
│   └── ollama.yaml         # ArgoCD App — Ollama LLM server
├── apps/
│   ├── monitoring/         # Prometheus, Grafana, Node Exporter manifests
│   └── ollama/             # Ollama deployment, service, namespace
└── bootstrap/
    └── root-app.yaml       # App of Apps entry point — applied once to activate GitOps
```

## How it works

### Bootstrap

The cluster is deployed entirely through a GitHub Actions workflow. Nothing is installed manually.

```
GitHub Actions (bootstrap.yml)
  1. Validate inputs
  2. Manual approval gate (cluster-approval environment)
  3. Install K3s server on controlplane (self-hosted runner on Pi)
  4. SSH into gpu-node → install K3s agent → join cluster
  5. Install ArgoCD → apply root-app → GitOps takes over
```

After step 5, everything is managed from Git. No further manual intervention needed.

### GitOps flow

```
Git push to main
  → ArgoCD detects diff
    → Applies changed manifests to cluster
      → Cluster state matches Git
```

`root-app.yaml` is an ArgoCD Application that watches `argocd/applications/`. Every YAML in that directory becomes a child ArgoCD Application. Each child syncs its own set of resources. Adding a new application is a single Git commit.

All applications use `automated: {prune: true, selfHeal: true}` — changes in Git are applied automatically, and manual changes to the cluster are reverted.

### GPU stack

The NVIDIA GPU Operator manages the full software stack on gpu-node:

- **Container Toolkit** — patches containerd to enable the NVIDIA runtime
- **Device Plugin** — registers `nvidia.com/gpu` as a schedulable Kubernetes resource
- **DCGM** — GPU monitoring daemon (utilisation, memory, temperature, power)
- **DCGM Exporter** — exposes DCGM metrics to Prometheus on port 9400
- **GPU Feature Discovery** — labels nodes with GPU properties

K3s uses a non-standard containerd socket path (`/run/k3s/containerd/containerd.sock`). The GPU Operator Helm values include the correct socket path override — without this, the container toolkit fails to reload containerd.

GPU workloads require three things in the pod spec:

```yaml
nodeSelector:
  nvidia.com/gpu: 'true'
runtimeClassName: nvidia
resources:
  requests:
    nvidia.com/gpu: '1'
  limits:
    nvidia.com/gpu: '1'
```

### Observability

Prometheus scrapes metrics from:

- Node Exporter (host-level: CPU, memory, disk, network) on both nodes
- Kubernetes node discovery via the API
- DCGM Exporter (GPU metrics) from the gpu-operator namespace

Grafana connects to Prometheus as a datasource via Kubernetes DNS (`http://prometheus:9090`) and serves dashboards on NodePort 30030.

### LLM inference

Ollama runs on gpu-node, scheduled exclusively via `nodeSelector: nvidia.com/gpu: "true"`. On startup it detects the RTX 2060 Super via CUDA and logs:

```
inference compute library=CUDA compute=7.5 description="NVIDIA GeForce RTX 2060 SUPER" total="8.0 GiB"
```

Running a model:

```bash
kubectl exec -n ollama <pod-name> -- ollama run llama3.2:3b "your prompt"
```

Llama 3.2 3B (Q4_K_M, ~2GB) fits comfortably within 8GB VRAM.

## Access

| Service    | URL                                                         |
| ---------- | ----------------------------------------------------------- |
| Grafana    | `http://<controlplane-ip>:30030`                            |
| Prometheus | `http://<controlplane-ip>:30090`                            |
| Ollama API | `http://<gpu-node-ip>:30434`                                |
| ArgoCD UI  | `kubectl port-forward svc/argocd-server -n argocd 8080:443` |

## Phases

- **Phase 1** — OS install, K3s cluster, node join (documented in `docs/phase-1-cluster-setup.md`)
- **Phase 2** — NVIDIA GPU Operator, DCGM Exporter
- **Phase 3** — Prometheus, Grafana, GPU observability dashboards
- **Phase 4** — ArgoCD GitOps, App of Apps
- **Phase 5** — Ollama GPU workload, LLM inference
- **Phase 6** — GPU time-slicing (in progress)
- **Phase 7** — Polish and publish

## What's next

- Fix DCGM Exporter hostengine connectivity so GPU metrics flow into Prometheus
- Import NVIDIA DCGM Grafana dashboard (ID 12239)
- Add persistent storage for Ollama models (PVC backed by local storage on gpu-node)
- GPU time-slicing for multi-tenant scheduling
