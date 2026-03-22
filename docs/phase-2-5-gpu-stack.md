# gpu-infra-lab: Phases 2–5

GPU Operator, observability, GitOps, and running a real LLM on Kubernetes.

This document covers what was built, why each piece exists, and how it fits together.

---

## What was built

By the end of these phases, the cluster can:

- Schedule GPU workloads using Kubernetes-native resource requests (`nvidia.com/gpu: 1`)
- Monitor GPU metrics (utilisation, temperature, power, memory) via DCGM and Prometheus
- Manage all applications from Git using ArgoCD — nothing is deployed manually
- Run a quantised 7B language model on the RTX 2060 Super via Ollama

---

## Repository structure

```
gpu-infra-lab/
├── argocd/
│   └── applications/
│       ├── gpu-operator.yaml     # ArgoCD app — deploys GPU Operator from Helm
│       ├── monitoring.yaml       # ArgoCD app — deploys Prometheus + Grafana stack
│       └── ollama.yaml           # ArgoCD app — deploys Ollama LLM server
├── apps/
│   ├── monitoring/
│   │   ├── kustomization.yaml    # Deployment order for monitoring components
│   │   ├── namespace.yaml        # monitoring namespace
│   │   ├── node-exporter.yaml    # DaemonSet + Service for host metrics
│   │   ├── prometheus/
│   │   │   ├── rbac.yaml         # ServiceAccount + ClusterRole for API access
│   │   │   ├── configmap.yaml    # Prometheus scrape config
│   │   │   ├── deployment.yaml   # Prometheus server
│   │   │   └── service.yaml      # NodePort 30090
│   │   └── grafana/
│   │       ├── configmaps.yaml   # Datasource + dashboard provisioning config
│   │       ├── deployment.yaml   # Grafana server
│   │       └── service.yaml      # NodePort 30030
│   └── ollama/
│       ├── kustomization.yaml    # Deployment order for Ollama
│       ├── namespace.yaml        # ollama namespace
│       ├── deployment.yaml       # Ollama pod — GPU-scheduled, nvidia runtime
│       └── service.yaml          # NodePort 30434
└── bootstrap/
    └── root-app.yaml             # ArgoCD App of Apps — the entry point for GitOps
```

---

## Phase 2: NVIDIA GPU Operator

### What problem it solves

Running GPU workloads on Kubernetes requires several NVIDIA software components to be installed and configured correctly on the node. Without them, Kubernetes has no visibility of the GPU and cannot schedule pods onto it.

Doing this manually on each node is error-prone and doesn't scale. The GPU Operator automates the entire NVIDIA software stack as Kubernetes-managed components.

### What the GPU Operator installs

| Component                   | What it does                                                          |
| --------------------------- | --------------------------------------------------------------------- |
| Container Toolkit           | Patches containerd so GPU-enabled containers can access the GPU       |
| Device Plugin               | Exposes `nvidia.com/gpu` as a schedulable resource in Kubernetes      |
| DCGM                        | NVIDIA Data Center GPU Manager — the GPU monitoring engine            |
| DCGM Exporter               | Exposes DCGM metrics in Prometheus format on port 9400                |
| GPU Feature Discovery (GFD) | Adds GPU labels to nodes (e.g. `nvidia.com/gpu.model=RTX-2060-Super`) |

### Why driver management is disabled

The host already has the NVIDIA driver installed (580.126.09, installed automatically by Ubuntu during OS setup). If `driver.enabled: true`, the GPU Operator would try to install its own driver and conflict with the existing one. Setting `driver.enabled: false` tells the Operator to skip driver management and use whatever is already on the host.

### The K3s containerd socket problem

The GPU Operator's container toolkit needs to patch containerd's config and signal it to reload. By default, it looks for the containerd socket at `/run/containerd/containerd.sock`. K3s ships its own bundled containerd and places the socket at `/run/k3s/containerd/containerd.sock`.

Without telling the toolkit where to find the socket, it fails after 6 retry attempts:

```
unable to dial: dial unix /runtime/sock-dir/containerd.sock: connect: no such file or directory
```

The fix is to pass the correct paths via environment variables in the Helm values:

```yaml
toolkit:
  enabled: true
  env:
    - name: CONTAINERD_SOCKET
      value: /run/k3s/containerd/containerd.sock
    - name: CONTAINERD_CONFIG
      value: /var/lib/rancher/k3s/agent/etc/containerd/config.toml
    - name: CONTAINERD_RUNTIME_CLASS
      value: nvidia
    - name: CONTAINERD_SET_AS_DEFAULT
      value: 'true'
```

### MIG manager is disabled

MIG (Multi-Instance GPU) splits high-end datacenter GPUs like the A100 or H100 into smaller isolated instances. The RTX 2060 Super does not support MIG, so `migManager.enabled: false`.

### The ArgoCD application

`argocd/applications/gpu-operator.yaml` is an ArgoCD `Application` resource. It tells ArgoCD:

- Where to get the Helm chart: `https://helm.ngc.nvidia.com/nvidia`
- Which chart: `gpu-operator`
- Which version: `v26.3.0`
- What Helm values to apply (all the component toggles above)
- Where to deploy it: the `gpu-operator` namespace on the local cluster

When ArgoCD syncs this, it runs the Helm install and keeps the deployed state reconciled with what is in Git.

### Verifying GPU Operator is working

After the toolkit pod goes Running, the device plugin registers the GPU with Kubernetes:

```bash
kubectl get node gpu-node -o jsonpath='{.status.allocatable}' | tr ',' '\n'
```

You should see `nvidia.com/gpu: 1` in the output. This is the signal that GPU scheduling is available.

---

## Phase 3: Observability

### Why a custom stack rather than a managed one

This project uses raw Kubernetes manifests (Prometheus, Grafana, Node Exporter) rather than the Prometheus Operator or kube-prometheus-stack Helm chart. The reason is deliberate: deploying the components individually means understanding what each one does. The Helm chart bundles everything and hides the wiring.

### Components

**Node Exporter** is a DaemonSet — it runs one pod on every node automatically. It exposes host-level metrics from `/proc` and `/sys`: CPU usage, memory, disk I/O, network traffic. The pod mounts these host paths read-only. It uses `hostNetwork: true` so it binds directly to the node's network interface.

**Prometheus** scrapes metrics from configured targets on a 15-second interval. It stores time-series data with a 7-day retention window. It needs a `ServiceAccount` and `ClusterRole` because it uses the Kubernetes API to discover nodes and pods — that is what the `kubernetes_sd_configs` block in the configmap does.

The Prometheus configmap includes a scrape job for `dcgm-exporter`:

```yaml
- job_name: 'dcgm-exporter'
  static_configs:
    - targets: ['nvidia-dcgm-exporter.gpu-operator:9400']
```

This tells Prometheus to scrape GPU metrics from the DCGM Exporter service in the `gpu-operator` namespace. GPU Operator manages the DCGM Exporter — there is no separate DCGM deployment.

**Grafana** connects to Prometheus as a datasource (configured via the `grafana-datasources` ConfigMap) and serves dashboards on port 3000. The datasources and dashboard provisioning config are supplied as ConfigMaps mounted into the container — Grafana picks them up automatically on startup without needing manual UI configuration.

### Kustomization

`apps/monitoring/kustomization.yaml` lists every manifest and controls deployment order. ArgoCD uses Kustomize by default when it finds a `kustomization.yaml` in the target path. This ensures the namespace exists before anything else is deployed into it, and RBAC is in place before the Prometheus pod starts.

### Access

| Service    | How to reach it                                |
| ---------- | ---------------------------------------------- |
| Prometheus | `http://<controlplane-ip>:30090`               |
| Grafana    | `http://<controlplane-ip>:30030` (admin/admin) |

Both use `NodePort` services so they are reachable directly from the home network without port-forwarding.

---

## Phase 4: GitOps with ArgoCD

### What GitOps means here

GitOps means Git is the single source of truth for what runs in the cluster. ArgoCD watches the repo and reconciles the cluster state with what is in `main`. If you change a manifest and push, ArgoCD detects the diff and applies it. If someone makes a manual change to a resource in the cluster, ArgoCD reverts it (this is what `selfHeal: true` does).

### App of Apps pattern

`bootstrap/root-app.yaml` is the entry point. It is an ArgoCD `Application` that points at `argocd/applications/` in the repo. That directory contains three more `Application` resources: `gpu-operator.yaml`, `monitoring.yaml`, and `ollama.yaml`.

When root-app syncs, ArgoCD creates those three child applications. Each child application then syncs its own resources. This is the App of Apps pattern — one application manages other applications.

The flow is:

```
root-app.yaml (applied manually once)
  -> gpu-operator.yaml (ArgoCD app)
  -> monitoring.yaml (ArgoCD app)
  -> ollama.yaml (ArgoCD app)
        -> apps/ollama/* (actual Kubernetes resources)
```

After `root-app.yaml` is applied to the cluster once, everything else is driven from Git.

### Sync policy

All applications use:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

`prune: true` means if you delete a manifest from Git, ArgoCD deletes the corresponding resource from the cluster. `selfHeal: true` means if someone manually edits a resource in the cluster, ArgoCD reverts it back to what Git says.

### ServerSideApply for GPU Operator

The GPU Operator CRDs (Custom Resource Definitions) are large — they exceed the annotation size limit that client-side apply uses to track changes. `ServerSideApply=true` moves the field management to the API server, which avoids the annotation size limit.

---

## Phase 5: GPU Workload — Ollama

### What Ollama is

Ollama is an LLM inference server. It exposes a REST API on port 11434, manages model downloads, and handles GPU scheduling internally. For this project it serves as proof that the GPU stack works end-to-end: a model runs, a request comes in, the GPU does the inference.

### GPU scheduling in the deployment

Three things in `apps/ollama/deployment.yaml` are required for GPU scheduling to work:

**nodeSelector** pins the pod to gpu-node:

```yaml
nodeSelector:
  nvidia.com/gpu: 'true'
```

**Resource request** tells the Kubernetes scheduler this pod needs a GPU:

```yaml
resources:
  requests:
    nvidia.com/gpu: '1'
  limits:
    nvidia.com/gpu: '1'
```

Without this, the pod would be scheduled but would not have access to the GPU device.

**runtimeClassName** tells containerd to use the NVIDIA runtime for this pod:

```yaml
runtimeClassName: nvidia
```

The NVIDIA runtime is what injects the GPU libraries and devices into the container. The container toolkit (installed by the GPU Operator) configures containerd with this runtime. Without `runtimeClassName: nvidia`, the pod runs with the default runtime and cannot see the GPU.

### The emptyDir limitation

The models volume uses `emptyDir`:

```yaml
volumes:
  - name: models
    emptyDir: {}
```

This means model files are stored in temporary pod storage. When the pod restarts, all downloaded models are deleted and must be re-downloaded. For a production setup this would be replaced with a PersistentVolumeClaim backed by local storage on gpu-node. This is a known gap to address in a later phase.

### Verifying GPU inference is working

After the pod is Running, check the logs:

```bash
kubectl logs -n ollama <pod-name>
```

The key line to look for:

```
inference compute library=CUDA name=CUDA0 description="NVIDIA GeForce RTX 2060 SUPER" total="8.0 GiB"
```

This confirms Ollama detected the GPU and is using CUDA for inference.

Run a model to confirm end-to-end:

```bash
kubectl exec -n ollama <pod-name> -- ollama run llama3.2:3b "say hello in one sentence"
```

This downloads the 3B model (~2GB) and runs it on the GPU. Llama 3.2 3B fits comfortably within the 8GB VRAM of the RTX 2060 Super.

---

## How it all connects

```
Git (github.com/BlazeTylr/gpu-infra-lab)
  |
  | ArgoCD watches main branch
  v
ArgoCD (running on controlplane / Raspberry Pi)
  |
  |-- gpu-operator app --> GPU Operator (on gpu-node)
  |                           |-- Container Toolkit  (patches containerd)
  |                           |-- Device Plugin      (registers nvidia.com/gpu)
  |                           |-- DCGM Exporter      (GPU metrics on :9400)
  |                           |-- GFD               (node labels)
  |
  |-- monitoring app  --> Prometheus (scrapes metrics from all targets)
  |                    -> Grafana    (dashboards)
  |                    -> Node Exporter (host metrics from both nodes)
  |
  |-- ollama app      --> Ollama (LLM server on gpu-node, using CUDA runtime)
```

Prometheus scrapes: Node Exporter (host metrics), the Kubernetes API (node discovery), and DCGM Exporter (GPU metrics). Grafana visualises all of it.

---

## What is not done yet

- **DCGM Exporter** is failing to connect to the DCGM hostengine. GPU metrics are not yet flowing into Prometheus. This needs to be debugged.
- **GPU dashboards in Grafana** — the NVIDIA DCGM Grafana dashboard (ID 12239) needs to be imported once DCGM is working.
- **Persistent model storage** — the Ollama models volume needs a PVC so models survive pod restarts.
- **Phase 6: GPU time-slicing** — sharing the GPU across multiple workloads simultaneously.
- **Phase 7: Documentation and publish** — full write-up of all phases for the portfolio.
