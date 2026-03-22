# gpu-infra-lab

Multi-node Kubernetes cluster with NVIDIA GPU scheduling, observability, and GitOps deployment.

Built as a homelab to learn and demonstrate AI infrastructure engineering patterns
used in production GPU clouds like Nscale and CoreWeave.

## Hardware

| Node | Role | Specs |
|---|---|---|
| controlplane (Raspberry Pi) | Control plane | ARM64, K3s server, ArgoCD |
| gpu-node (PC) | GPU worker | Intel i7-7740X, 16GB RAM, RTX 2060 Super 8GB |

## Stack

- **Kubernetes** — K3s, multi-arch cluster (ARM64 + AMD64)
- **GitOps** — ArgoCD App of Apps pattern
- **GPU** — NVIDIA GPU Operator, DCGM Exporter
- **Observability** — Prometheus, Grafana, DCGM metrics
- **Workloads** — Ollama serving a 7B model on GPU
