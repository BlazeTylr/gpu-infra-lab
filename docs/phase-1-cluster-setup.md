# Phase 1: Cluster Setup

Installing Ubuntu on the GPU node, joining it to the existing K3s cluster, and preparing the foundation for the GPU stack.

---

## Hardware

| Node                        | Role       | Specs                                                                            |
| --------------------------- | ---------- | -------------------------------------------------------------------------------- |
| controlplane (Raspberry Pi) | K3s server | ARM64, K3s server, runs ArgoCD + monitoring                                      |
| gpu-node (PC)               | GPU worker | Intel i7-7740X, 16GB RAM, NVIDIA RTX 2060 Super 8GB, Ubuntu 24.04 LTS, 500GB SSD |

---

## Installing Ubuntu 24.04 LTS on the PC

The PC originally ran Windows on the C: drive (Samsung NVMe 500GB). Ubuntu 24.04 LTS was installed on the D: drive (WD Blue 500GB SSD) as a dual-boot setup, leaving Windows intact.

### The display problem

During the initial boot from the USB installer, the standard graphics mode produced corrupted rendering. The NVIDIA RTX 2060 Super does not work cleanly with Ubuntu's default `nouveau` open-source driver during installation.

Fix: select **Safe Graphics** mode in the GRUB boot menu when booting the installer. This bypasses the nouveau driver and uses a basic framebuffer instead. Installation completes normally.

After installation, Ubuntu automatically installs the proprietary NVIDIA driver (580.126.09) with CUDA 13.0. The display works correctly from that point on.

### Confirming GPU detection

```bash
$ nvidia-smi
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.126.09             Driver Version: 580.126.09     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
|   0  NVIDIA GeForce RTX 2060 ...    Off |   00000000:02:00.0  On |                  N/A |
|  0%   42C    P8              9W /  175W |     153MiB /   8192MiB |      1%      Default |
+-----------------------------------------+------------------------+----------------------+
```

Driver installed, GPU detected, CUDA available.

---

## Joining the GPU node to the cluster

The Raspberry Pi (controlplane) was already running K3s as the control plane. To join the PC as a worker node, the join token from the Pi is needed.

### Get the join token

On the Pi:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

### Join the cluster

On the PC:

```bash
curl -sfL https://get.k3s.io | \
  K3S_URL=https://192.168.1.48:6443 \
  K3S_TOKEN=<token> \
  sh -
```

This downloads K3s, installs it as an agent, and joins the cluster in one command. The agent connects to the server at `192.168.1.48:6443`, authenticates with the token, and registers the node.

### Label the node

```bash
kubectl label node gpu-node node-role.kubernetes.io/gpu-worker=true
```

This adds a human-readable role label so `kubectl get nodes` shows `gpu-worker` instead of `<none>`. The `nvidia.com/gpu=true` label is added later by the bootstrap workflow and used by the GPU Operator and pod scheduling.

---

## Result

```
$ kubectl get nodes
NAME           STATUS   ROLES                  AGE     VERSION
controlplane   Ready    control-plane,master   85d     v1.32.3+k3s1
gpu-node       Ready    gpu-worker             3m40s   v1.32.3+k3s1
```

Two nodes, both Ready. Multi-architecture cluster — ARM64 (Pi) and AMD64 (PC) running K3s on the same version.

---

## Note on K3s versions

When initially joined, the Pi was on `v1.33.6+k3s1` and the PC installed `v1.34.5+k3s1` (latest at the time). The K3s agent should match the server version, or the server should be equal or newer. To align them:

```bash
# On the Pi — upgrade to latest
curl -sfL https://get.k3s.io | sh -

# Verify both nodes match
kubectl get nodes
```

In the final setup both nodes run `v1.32.3+k3s1`, installed by the bootstrap workflow with a pinned version.

---

## What's next

- **Phase 2** — Deploy NVIDIA GPU Operator and DCGM Exporter
- **Phase 3** — GPU observability dashboards in Grafana
- **Phase 4** — GitOps everything with ArgoCD
- **Phase 5** — Deploy Ollama with a 7B model on GPU
- **Phase 6** — GPU time-slicing
- **Phase 7** — Polish and publish
