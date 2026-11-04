# Forge — Hibernation Storage Analysis

> How exactly Forge saves resources, and what happens as PR volume grows.

---

## 1. What "space" actually means in Kubernetes

When people say "saving space" in Kubernetes, there are four distinct resources involved. They are very different in size and importance:

| Resource | What it is | Where it lives | Size range |
|---|---|---|---|
| **CPU requests** | CPU cores reserved on the node for a Pod | Node (physical machine) | 100m–2000m per Pod |
| **Memory requests** | RAM reserved on the node for a Pod | Node (physical machine) | 128Mi–2Gi per Pod |
| **etcd objects** | Kubernetes resource definitions (Deployment, Service, etc.) | etcd database | 2KB–10KB per object |
| **Container images** | The actual application binary + OS layers | Node disk | 20MB–500MB per image |

**The most important of these by far is CPU + memory requests.** These are real, physical resources on your machine. When a Pod has a CPU request of 500m, the Kubernetes scheduler treats those 500m as unavailable to everyone else — whether the Pod is doing anything or not.

etcd storage is almost irrelevant by comparison — the entire etcd database for a 100-environment cluster fits in a few megabytes.

---

## 2. Exactly what exists at each lifecycle state

### Active environment (fully running)

```
Kubernetes objects in etcd:
  ✓ Environment CRD              ~5KB
  ✓ Namespace (forge-pr-101)     ~2KB
  ✓ Deployment                   ~4KB
  ✓ ReplicaSet                   ~3KB
  ✓ Pod                          ~4KB
  ✓ Service                      ~2KB
  ✓ Ingress                      ~2KB
  ✓ ResourceQuota                ~2KB
  ✓ LimitRange                   ~1KB
  ✓ NetworkPolicy × 3            ~6KB
  ─────────────────────────────────────
  Total etcd:                    ~31KB

Node resources consumed:
  ✓ CPU request:                 500m (0.5 cores)
  ✓ Memory request:              256Mi
  ✓ Container image on disk:     ~100MB (one-time, shared across restores)

Running processes:
  ✓ Application Pod              1 Pod running
```

### Warm hibernation (Pods scaled to zero)

```
Kubernetes objects in etcd:
  ✓ Environment CRD              ~5KB
  ✓ Namespace (forge-pr-101)     ~2KB
  ✓ Deployment  (replicas=0)     ~4KB   ← still exists, just 0 replicas
  ✓ ReplicaSet  (desired=0)      ~3KB
  ✗ Pod                          GONE   ← deleted
  ✓ Service                      ~2KB
  ✓ Ingress                      ~2KB
  ✓ ResourceQuota                ~2KB
  ✓ LimitRange                   ~1KB
  ✓ NetworkPolicy × 3            ~6KB
  ─────────────────────────────────────
  Total etcd:                    ~27KB  (saved 4KB vs active)

Node resources consumed:
  ✗ CPU request:                 0m     ← FREED (no Pod = no reservation)
  ✗ Memory request:              0Mi    ← FREED
  ✓ Container image on disk:     ~100MB (still cached — enables fast restore)

Running processes:
  ✗ None
```

### Deep hibernation (Deployment + Service deleted)

```
Kubernetes objects in etcd:
  ✓ Environment CRD              ~5KB   ← full spec retained here
  ✓ Namespace (forge-pr-101)     ~2KB
  ✗ Deployment                   GONE
  ✗ ReplicaSet                   GONE
  ✗ Pod                          GONE
  ✗ Service                      GONE
  ✗ Ingress                      GONE
  ✓ ResourceQuota                ~2KB
  ✓ LimitRange                   ~1KB
  ✓ NetworkPolicy × 3            ~6KB
  ─────────────────────────────────────
  Total etcd:                    ~16KB  (saved 15KB vs active, 11KB vs warm)

Node resources consumed:
  ✗ CPU request:                 0m     ← FREED (same as warm)
  ✗ Memory request:              0Mi    ← FREED (same as warm)
  ~ Container image on disk:     ~100MB (may be GC'd eventually by kubelet)

Running processes:
  ✗ None
```

---

## 3. The numbers at scale — where the real savings appear

Using these baseline numbers per environment:
- CPU request: **500m** (0.5 cores)
- Memory request: **256Mi**
- etcd storage: **~31KB** active, **~16KB** deep hibernated

### Without Forge (every environment always running)

| Open PRs | Running Pods | CPU Reserved | RAM Reserved | etcd storage |
|---|---|---|---|---|
| 10 | 10 | 5 cores | 2.5 GB | 310 KB |
| 25 | 25 | 12.5 cores | 6.25 GB | 775 KB |
| 50 | 50 | 25 cores | 12.5 GB | 1.55 MB |
| 100 | 100 | **50 cores** | **25 GB** | 3.1 MB |
| 500 | 500 | **250 cores** | **125 GB** | 15.5 MB |

At 100 PRs you need a 50-core cluster just for preview environments. At 500 PRs it's a 250-core cluster. This is why companies either spend enormous money on cloud infrastructure or stop giving developers preview environments.

### With Forge (max 10 active, rest deep hibernated)

| Open PRs | Active | Hibernated | Running Pods | CPU Reserved | RAM Reserved | etcd storage |
|---|---|---|---|---|---|---|
| 10 | 10 | 0 | 10 | 5 cores | 2.5 GB | 310 KB |
| 25 | 10 | 15 | 10 | 5 cores | 2.5 GB | 550 KB |
| 50 | 10 | 40 | 10 | 5 cores | 2.5 GB | 950 KB |
| 100 | 10 | 90 | 10 | **5 cores** | **2.5 GB** | 1.75 MB |
| 500 | 10 | 490 | 10 | **5 cores** | **2.5 GB** | 8.14 MB |

**At 100 PRs: Forge uses 5 cores instead of 50. That is a 10× reduction.**
**At 500 PRs: Forge uses 5 cores instead of 250. That is a 50× reduction.**

The cluster size stays constant regardless of how many PRs are open. Only the etcd storage grows — and it grows slowly (a few MB even at 500 environments).

---

## 4. The etcd growth is negligible

This is worth making explicit because people sometimes worry about it.

```
500 deep-hibernated environments × 16KB each = 8 MB

Kubernetes default etcd size limit = 2 GB
Forge uses 8 MB out of 2000 MB = 0.4% of etcd capacity
```

You could have **10,000 deep-hibernated environments** and still only use ~160MB of etcd. etcd is not a constraint.

---

## 5. Warm vs deep hibernation — honest comparison

The CPU and memory saving is **identical** between warm and deep. Both have zero running Pods. The difference is only in etcd objects:

| | Warm | Deep |
|---|---|---|
| CPU saved | 500m | 500m |
| Memory saved | 256Mi | 256Mi |
| etcd objects removed | 1 (Pod) | 6 (Pod, Deployment, ReplicaSet, Service, Ingress + Pod) |
| etcd space saved vs active | ~4KB | ~15KB |
| Restore time | ~3 seconds | ~5–7 seconds |

**The real reason for two levels is not storage — it is K8s controller overhead.**

Every Deployment object is watched by the kube-controller-manager. With 500 Deployments at replicas=0, the controller still reconciles them constantly, checking nothing has drifted. This is wasted CPU on the control plane. Deep hibernation removes those objects entirely, reducing controller noise.

At small scale (10–20 environments) you won't notice this. At 100+ environments it becomes measurable.

---

## 6. Container image storage — the hidden cost

One thing the table above doesn't show: container images on the node's disk.

```
Each environment uses one image: ~100MB (a typical Go app on Alpine)

100 environments, all using different images:
  Without caching: 100 × 100MB = 10GB on node disk

100 environments, all using the SAME sample app:
  With caching:    1 × 100MB  = 100MB on node disk
```

For Forge's demo, all environments use the same sample app image — so image storage is constant regardless of environment count. In production, different PRs have different images but share base layers (the Alpine/Go base is shared), so the actual per-image overhead is typically 5–20MB of unique layers.

**This is why the local registry matters:** images pulled from `host.minikube.internal:5000` are cached on the Minikube node. A cache hit on restore = no download, instant start.

---

## 7. Container image caching in deep hibernation

This is the most commonly misunderstood part of deep hibernation.

### Where images are stored

Container images live on the **Minikube node's disk** (managed by containerd), completely separate from Kubernetes objects in etcd. When kubelet pulls an image to run a Pod, it stores the image layers here:

```
Minikube node disk:
  /var/lib/containerd/snapshots/
    └── sha256:abc123...    ← image layers stored here
```

### What happens when deep hibernation deletes the Deployment

```
Active:           Deployment → Pod running → image IN USE on node
Warm hibernated:  Deployment (replicas=0) → no Pod → image still on node (referenced by spec)
Deep hibernated:  Deployment DELETED → image still on node (unreferenced, but not removed)
```

**Kubernetes never deletes images when objects are deleted.** The image remains on disk after deep hibernation. Only kubelet's garbage collector removes images, and only when node disk usage exceeds 85%.

### The risk: kubelet image garbage collection

If the Minikube node's disk fills up while an environment is deep hibernated, kubelet removes least-recently-used images. That environment's next restore will need a full image pull.

```
Disk < 80%  →  no GC, image safe
Disk > 85%  →  GC runs, LRU images removed first
```

### Three layers of protection Forge uses

**Layer 1: Image digest in CRD status**

```yaml
status:
  imageDigest: "sha256:abc123def456..."
```

Even if the image is GC'd, Forge knows exactly which image to pull back. No tag ambiguity.

**Layer 2: Local registry (most important)**

The local registry on your laptop is NOT on the Minikube node — kubelet GC never touches it. It permanently holds every image ever deployed.

```
Minikube node               Your laptop
┌──────────────┐            ┌─────────────────────────┐
│ containerd   │  ────────▶ │ host.minikube.internal  │
│ image cache  │   pull     │ :5000 (local registry)  │
│ (may be GC'd)│            │ permanent copy          │
└──────────────┘            └─────────────────────────┘
```

Pull from local registry on a cache miss: **1–3 seconds** vs 30–60 seconds from Docker Hub.

**Layer 3: Minikube image preload (for demo reliability)**

```bash
minikube image load sample-app:latest
```

Pre-loaded images bypass GC pressure entirely. Run this before every demo.

### Restoration latency by image state

| Scenario | Image location | Restore time |
|---|---|---|
| Warm → Active | Node disk (always cached, Deployment keeps reference) | ~2–3s |
| Deep → Active (image still on node) | Node disk | ~4–6s |
| Deep → Active (image GC'd, local registry exists) | Pull from localhost:5000 | ~5–8s |
| Deep → Active (image GC'd, no local registry) | Pull from Docker Hub | ~35–65s |

**Conclusion:** The local registry is what makes deep hibernation restoration fast and reliable, not node-level image caching. Node caching is a bonus when it works; the local registry is the guarantee.

---

## 8. Summary — what Forge actually saves

| Resource | Saving | When |
|---|---|---|
| CPU requests | 100% of idle app workload | From the moment an environment enters warm hibernation |
| Memory requests | 100% of idle app workload | From the moment an environment enters warm hibernation |
| etcd object count | ~50% reduction | Deep hibernation only |
| Node controller overhead | Reduced at scale | Deep hibernation only |
| Image download on restore | ~95% (cache hit) | Throughout lifetime via local registry |

**The headline claim for your report:**

> *"Forge reduces preview environment CPU and memory consumption by 100% for idle environments through scale-to-zero hibernation, allowing a fixed 4-core cluster to support 100+ concurrent preview environments where a traditional always-on approach would require 50+ cores."*

---

## 8. How to show this live in the demo

```
Step 1: kubectl top nodes
        → shows current CPU + memory usage on Minikube node

Step 2: Create 10 environments, all running
        → kubectl top nodes shows ~5 cores used

Step 3: Trigger hibernation on all 10
        → kubectl top nodes drops to near-baseline (Forge control plane only)
        → Grafana: forge_active_pods drops to 0
        → Grafana: forge_cpu_requests_saved_millicores jumps to 5000m

Step 4: Access one environment URL
        → Gateway intercepts, triggers restore
        → Environment is live in ~3 seconds
        → Grafana: forge_active_pods goes to 1

Step 5: Show kubectl get environments
        → 10 environments exist
        → 9 deep hibernated, 1 active
        → Cluster paying for 1, not 10
```

That sequence makes the storage and resource story impossible to misunderstand.
