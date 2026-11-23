# Forge — High-Level Design

> Living document · Last updated: 2026-11-02
> Update this file as design decisions change. Add a note under the relevant section with the date and what changed.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [System Architecture](#2-system-architecture)
3. [Component Deep-Dives](#3-component-deep-dives)
4. [Environment Lifecycle State Machine](#4-environment-lifecycle-state-machine)
   - 4a. [Hibernation Storage & Restoration Latency](#4a-hibernation-storage--restoration-latency)
5. [CI/CD Pipeline Design](#5-cicd-pipeline-design)
6. [Technology Stack & Rationale](#6-technology-stack--rationale)
7. [Security Model & Resource Governance](#7-security-model--resource-governance)
8. [Observability & Metrics](#8-observability--metrics)
9. [Test Strategy](#9-test-strategy)
10. [Scope, Phases & Milestones](#11-scope-phases--milestones)
11. [Open Questions & Future Extensions](#12-open-questions--future-extensions)

---

## 1. Project Overview

Forge is a Kubernetes-native control plane that manages the full lifecycle of ephemeral preview environments — the temporary deployments created for feature branches and pull requests.

### The Problem

As AI-assisted development accelerates code output, teams open more PRs and create more preview environments. Without automation:

- Environments accumulate and consume cluster resources indefinitely
- Idle environments hold CPU and memory that new deployments need
- Manual cleanup does not scale with development velocity
- Stateful environments are hard to recover once deleted

### What Forge Does

| Capability | Description |
|---|---|
| Auto-provisioning | Creates namespace, Deployment, Service, Ingress on PR open |
| Inactivity detection | Monitors HTTP traffic to detect idle environments |
| Warm hibernation | Scales Pods to zero after 15 min idle; keeps routing alive |
| Deep hibernation | Removes Deployment + Service after 2h idle; retains metadata |
| Traffic-triggered wake | Incoming request to hibernated URL triggers automatic restore |
| Auto-termination | Cleans up all resources when PR closes or TTL expires |
| Image scanning | Trivy scans every image for CVEs before deployment |

### What Forge Is Not

- Not a general-purpose PaaS (not replacing Render, Heroku, Fly.io)
- Not a production environment manager
- Not a multi-cluster or multi-cloud system
- Not a process checkpoint system (no CRIU)

### SPE Capstone Context

Forge is also the SPE capstone project. The platform itself is delivered through a fully automated CI/CD pipeline satisfying all mandatory DevOps stages. The Platform Engineering lifecycle controller is the cutting-edge extension. Trivy image scanning adds a DevSecOps layer.

---

## 2. System Architecture

### High-Level Component Map

> Diagram: [docs/diagrams/system-architecture.md](diagrams/system-architecture.md)

### Design Choice: Separate Control Plane and Data Plane

**Decision:** The control plane (operator, API, lifecycle manager) is always running. The data plane (app Pods) scales to zero or disappears during hibernation.

**Why:** If control plane components hibernated too, there would be no system left to wake environments. Forge's infrastructure is shared and permanent; the preview environments are temporary.

### Design Choice: Shared Gateway Instead of Per-Environment Ingress

**Decision:** One Forge Gateway handles routing for all environments, rather than creating individual Nginx/Traefik Ingress resources per environment.

**Why:** Ingress controllers cannot trigger application wake-up on their own. The gateway needs custom logic to (a) detect hibernation state, (b) trigger restore, (c) wait for readiness, (d) coalesce concurrent requests. A shared Go gateway gives full control over this logic. Individual Ingress resources are removed during deep hibernation anyway.

---

## 3. Component Deep-Dives

### 3.1 Webhook Receiver

**Responsibility:** Receive GitHub PR webhook events and translate them into Environment API calls.

**Design choices:**
- Validates GitHub webhook signatures (HMAC-SHA256) before processing
- Handles `opened`, `reopened`, `synchronize`, `closed` event types
- Idempotent: duplicate webhook delivery for the same event produces the same result
- Runs as a separate Go HTTP server (not merged into the main API) so it can be secured independently

**Events handled:**

| GitHub event | Forge action |
|---|---|
| PR opened | Create environment |
| PR synchronize (new commit) | Update image tag, redeploy |
| PR closed / merged | Terminate environment |

### 3.2 Environment API

**Responsibility:** Accept environment lifecycle requests and create/update Environment CRDs in Kubernetes.

**Design choices:**
- REST API, not gRPC — simpler for a local-cluster project, easier to test with curl
- Writes to Kubernetes CRDs directly; does not maintain its own database
- Kubernetes is the source of truth for environment state
- Validates requests before writing (name format, resource limits, TTL bounds)

**Key endpoints:**

```
POST   /api/v1/environments          Create environment
GET    /api/v1/environments          List environments
GET    /api/v1/environments/:name    Get environment status
DELETE /api/v1/environments/:name    Terminate environment
POST   /api/v1/environments/:name/pin   Protect from hibernation
```

### 3.3 Forge Operator (Kubernetes Controller)

**Responsibility:** The central reconciliation loop. Watches Environment CRDs and ensures actual Kubernetes state matches desired state.

**Design choices:**
- Built with `controller-runtime` (the standard Go library for Kubernetes operators)
- Reconciliation is idempotent: running it multiple times on the same state is safe
- **Environment CRD is cluster-scoped** (not namespace-scoped) — a namespaced resource cannot own objects in a different namespace, so owner references would silently fail. Making it cluster-scoped lets the CRD own the Namespace itself; deleting the Namespace cascades to everything inside it. This decision must be made before generating the CRD because changing scope later requires regenerating all manifests and migrating existing objects.
- Uses Kubernetes owner references: the cluster-scoped Environment CRD owns the Namespace; everything inside the namespace (Deployment, Service, Ingress, PV) is owned by the namespace and cleaned up automatically when it is deleted.
- Uses finalizers to ensure cleanup logic (stopping traffic, snapshot if needed) runs before the CRD is permanently removed
- Persists lifecycle state in the CRD's `status` subresource — survives controller restarts
- CRD fields validated with kubebuilder markers and CEL rules (name format, TTL bounds) — the API server rejects bad specs at admission time, not in controller code

**Reconciliation loop:**

> Diagram: [docs/diagrams/operator-reconciliation.md](diagrams/operator-reconciliation.md)

### 3.4 Lifecycle Manager

**Responsibility:** Monitors environment activity and drives state transitions based on inactivity timeouts and TTLs.

**Design choices:**
- Runs as a goroutine within the operator process (not a separate binary) — simplifies deployment
- Uses HTTP request timestamps recorded by the Gateway as the inactivity signal
- Periodic reconciliation loop (every 60s) checks all active environments against their policies
- TTL check is hard: expired environments terminate regardless of pin status
- **Restore grace period:** when an environment is restored, `lastActivityAt` is reset to `now()` and a minimum 5-minute "stay awake" window is enforced. Without this, the 60s inactivity loop would re-hibernate immediately because `lastActivityAt` still held the pre-hibernation value.

**Inactivity policy (configurable per environment):**

```
Last HTTP request > 15 min ago  →  Warm hibernation  (unless in grace period)
Last HTTP request > 2h ago      →  Deep hibernation   (unless in grace period)
PR closed OR TTL expired        →  Termination        (overrides grace period)
Manually pinned                 →  Skip inactivity check
Restored < 5 min ago            →  Grace period: skip inactivity check
```

### Adaptive hibernation policy

Rather than applying identical timers to all environments, Forge uses a rule-based priority system to decide how quickly environments hibernate and which ones are evicted first under capacity pressure.

| Priority | Warm after (idle) | Deep after (idle) | Eviction order |
|---|---|---|---|
| `pinned` | Never | Never | Never evicted automatically |
| `high` | 30 min | 4 h | Last evicted under pressure |
| `normal` | 15 min | 2 h | Middle |
| `low` | 5 min | 30 min | First evicted under pressure |

Priority is set per environment via `spec.lifecycle.priority`. The default is `normal`.

**Capacity pressure eviction:** When a new environment (or a restore request) needs a slot and all active slots are full, the capacity worker picks the lowest-priority, least-recently-used active environment to hibernate first. A `pinned` environment is never evicted. If all active environments are `pinned`, the wake request returns `503 Capacity Limit Reached`.

**What this enables:** Teams can mark critical review environments `high` to protect them from automatic eviction while bot-triggered or low-importance preview environments are tagged `low` and yield their slots immediately when anything higher-priority needs them. This costs nothing — it is purely policy in the lifecycle manager, no new infrastructure.

### 3.5 Forge Gateway

**Responsibility:** Shared reverse proxy that routes traffic to all environments, detects hibernation, and triggers wake-up.

**Design choices:**
- Written in Go using `net/http/httputil.ReverseProxy`
- Routes by hostname: `pr-101.dev.local` → environment `pr-101`
- **State is read via a controller-runtime informer (watch), not a TTL cache.** A TTL cache means the gateway could forward to a scaled-down Service for several seconds after hibernation begins. An informer is event-driven — it is notified the moment the Environment CRD status changes — so the gateway's view is always current with zero polling overhead on the API server.
- Wake-up deduplication via a `sync.Map` of per-environment mutexes: first request triggers restore, subsequent concurrent requests wait on the same mutex
- Readiness-aware: does not forward until the app passes its readiness probe
- Hard timeout on restoration: if the app doesn't become ready in 60s, returns `503 Service Unavailable`
- **Browser clients** receive a "waking up" HTML page that auto-refreshes every 3 seconds rather than a raw 503, so reviewers see feedback instead of a generic error while the environment starts
- Gateway's RBAC: has permission only to `get`/`watch`/`list` Environments and to `patch` the `status` subresource — nothing else

**Request flow:**

> Diagram: [docs/diagrams/gateway-request-flow.md](diagrams/gateway-request-flow.md)

### 3.6 Resource Manager

**Responsibility:** Enforces per-environment and global resource limits.

**Design choices:**
- Uses native Kubernetes `ResourceQuota` and `LimitRange` per namespace — no custom scheduler
- Global limit: maximum 10 active environments (configurable)
- When at capacity for a **new provision**: hibernate the least-recently-used active environment first, then provision the new one
- **When a hibernated environment is woken and all 10 slots are already active:** evict the lowest-priority, least-recently-used active environment to warm hibernation to free a slot. If no eviction candidate exists (all 10 are `pinned`), the wake request returns `503 Capacity Limit Reached` with a message naming the constraint.
- **Capacity check is serialized:** a single goroutine (capacity worker) owns all admission decisions. Without this, two simultaneous creates can both pass the cap check before either increments the counter, resulting in 11 active environments. The capacity worker processes one admission at a time.
- Simple FIFO queue for pending provisioning requests

### Bounded restoration queue

Concurrent restoration requests across different environments are bounded globally, not just deduplicated per environment.

```
Incoming restoration requests
            |
            v
    Restoration semaphore (max 3 concurrent)
            |
      Slot available?
         /       \
       Yes        No
       |           |
   Restore      Bounded queue (max 5 pending)
       |           |
       v         Full?
    Active       /   \
              Yes     No
               |       |
           429 TooMany  Wait → Restore
```

- **Max concurrent restorations** (default 3): prevents 10 simultaneous deep hibernation restores from saturating the node's image-pull bandwidth and memory on a laptop cluster.
- **Max pending queue** (default 5): when both concurrent slots and queue are full, the gateway returns `429 Too Many Requests` with a `Retry-After: 10` header. This provides backpressure rather than accumulating unbounded requests in memory.
- **High-priority environments** skip the queue and are granted a slot immediately if any active non-`pinned` environment can be evicted to make room.
- Both limits are configurable via Helm (`gateway.maxConcurrentRestorations`, `gateway.maxPendingRestorations`).

---

## 4. Environment Lifecycle State Machine

> Diagram: [docs/diagrams/lifecycle-state-machine.md](diagrams/lifecycle-state-machine.md)

**State persistence:** All state is stored in the Environment CRD's `status.lifecycleState` field. If the operator restarts mid-transition, it re-reads the CRD and continues reconciliation from the persisted state.

### Event × State Matrix

What happens when an event arrives in the "wrong" state. This is also the primary test-case list.

| Event | PROVISIONING | ACTIVE | WARM_HIBERNATED | DEEP_HIBERNATED | RESTORING | TERMINATING |
|---|---|---|---|---|---|---|
| PR opened (duplicate) | No-op (idempotent) | No-op | No-op | No-op | No-op | Queue new after termination |
| PR synchronize (new commit) | Update image tag; re-provision | Redeploy with new image | Update CRD spec; restore then redeploy | Update CRD spec; restore then redeploy | Queue redeploy after restore | No-op (termination wins) |
| PR closed | Cancel provisioning → TERMINATING | TERMINATING | TERMINATING | TERMINATING | Cancel restore → TERMINATING | No-op (already terminating) |
| TTL expired | TERMINATING | TERMINATING | TERMINATING | TERMINATING | Cancel restore → TERMINATING | No-op |
| HTTP request arrives | Queue; return 202 | Forward | Trigger restore | Trigger restore | Wait on mutex | Return 410 Gone |
| Inactivity timeout fires | No-op (not yet active) | → WARM_HIBERNATED | → DEEP_HIBERNATED | No-op | No-op (grace period after restore) | No-op |
| Manual pin | Mark pinned; continue | Mark pinned | Mark pinned; no-op on inactivity | Mark pinned | Mark pinned | No-op |
| Restore succeeds | — | — | → ACTIVE | → ACTIVE | → ACTIVE | → TERMINATING (restart cleanup) |
| Restore times out | — | — | — | — | → RESTORATION_FAILED | — |

---

## 4a. Hibernation Storage & Restoration Latency

### What is actually stored per hibernation level

| Level | Stored where | What is kept | What is deleted | Storage cost |
|---|---|---|---|---|
| Warm | Kubernetes etcd (native) | Deployment (replicas=0), Service, Ingress, CRD | Nothing | ~5–10 KB per environment in etcd |
| Deep | Kubernetes etcd (CRD spec only) | Environment CRD (full spec + image digest), Namespace, ResourceQuota, LimitRange, NetworkPolicy, PersistentVolume (if stateful) | Deployment, Service, Ingress | ~3–5 KB per environment in etcd |
| Terminated | Nothing | Nothing | Everything | 0 |

No external database. No filesystem snapshots for stateless apps. Kubernetes etcd is the only store. 100 deep-hibernated environments occupy ~500 KB in etcd — negligible.

For stateful environments with a PersistentVolume: the PV is retained through both warm and deep hibernation and deleted only on termination, according to the environment's storage retention policy.

### Wake-up latency breakdown

**Warm hibernation wake:**
1. Patch Deployment replicas `0 → 1` — ~100ms (K8s API call)
2. Kubernetes scheduler places Pod on node — ~200–500ms
3. Kubelet starts container:
   - Image cached on node → ~500ms–1s
   - Image not cached, pulled from local registry → ~1–3s
   - Image not cached, pulled from Docker Hub → ~30–60s
4. App readiness probe passes — depends on app (~1–5s for a Go HTTP service)

**Estimated total (cached image): 2–5 seconds**

**Deep hibernation wake:**
1. Read full spec from CRD (operator keeps an in-memory cache of all CRDs via controller-runtime's informer — effectively 0ms read cost)
2. Recreate Deployment + Service + Ingress in parallel — ~300ms
3. Same Pod start sequence as warm

**Estimated total (cached image): 3–7 seconds**

The difference between warm and deep restoration latency is small (~2s). The dominant variable in both cases is **image cache hit vs miss**, not storage retrieval.

### Image caching strategy (most important latency lever)

| Technique | How | Effect |
|---|---|---|
| Local Docker registry | `localhost:5000` — images stored on same machine | Cache miss pull: 1–3s instead of 30–60s |
| Immutable image digest | CRD stores digest (`sha256:...`), never a mutable tag | Same layers reused across restores; no re-download |
| Warm hibernation default | Keeps Deployment object alive at replicas=0 | Node keeps image referenced longer before GC |
| Minikube image preload | `minikube image load forge-operator:latest` | Bypasses registry for Forge's own images |

### Wake-up deduplication (prevents thundering herd)

If 20 reviewers hit a hibernated environment URL simultaneously, Forge must not trigger 20 parallel restore operations. The gateway uses a `sync.Map` of per-environment mutexes:

```
Request 1  →  acquire lock  →  trigger restore  →  wait for readiness  →  forward  →  release lock
Request 2  →  wait on lock  ──────────────────────────────────────────────────────────▶  forward
Request 3  →  wait on lock  ──────────────────────────────────────────────────────────▶  forward
...
```

Exactly one restore operation fires per environment regardless of concurrent traffic. All waiters are served once the single restore completes.

### Bounded restoration

A restore operation has a hard 60-second timeout. If the app never passes its readiness probe:
- Gateway returns `503 Service Unavailable`
- Environment state is set to `RESTORATION_FAILED`
- No request is held open indefinitely
- Operator retries restoration on the next reconcile cycle

**Important rules:**
- A pinned environment skips inactivity transitions but still respects hard TTL
- A single restoration operation per environment — the gateway's mutex prevents duplicates
- Duplicate webhook events (PR opened twice) are idempotent: second create is a no-op if environment already exists

---

## 5. CI/CD Pipeline Design

This pipeline serves two purposes: (1) building and deploying Forge itself, and (2) demonstrating the SPE capstone DevOps stages.

### Pipeline Stages

> Diagram: [docs/diagrams/cicd-pipeline.md](diagrams/cicd-pipeline.md)

### Design Choices

**Jenkins vs GitHub Actions:** Jenkins runs locally on the same machine as Minikube — no external service dependency, satisfies the "laptop-runnable" constraint. GitHub Actions is an acceptable alternative if Jenkins setup is time-consuming.

**Local registry:** `docker run -d -p 5000:5000 registry:2` — avoids DockerHub rate limits and works fully offline. Minikube is configured to trust this registry.

**Trivy in CI:** `trivy image --exit-code 1 --severity HIGH,CRITICAL` fails the build on high/critical CVEs. This is intentionally strict for the demo — shows the pipeline actively enforcing security. For the actual demo, use a clean Alpine-based image that passes the scan.

**Helm chart for Forge:** Forge's own components (operator, gateway, API server, webhook receiver) are packaged as a Helm chart under `charts/forge/`. This satisfies the IaC/config management requirement alongside Ansible.

**Ansible roles:** Ansible handles machine setup using modular **roles** (not flat playbooks). Each role is independently reusable:

```
ansible/
  roles/
    docker/        ← installs Docker engine
    kubectl/       ← installs kubectl + kubeconfig
    minikube/      ← installs + starts Minikube
    helm/          ← installs Helm
    registry/      ← starts local Docker registry with persistent volume
    vault/         ← installs Vault CLI, starts Vault dev server, seeds initial secrets
    elk/           ← installs Filebeat on the host (ELK runs in K8s)
  site.yml         ← runs all roles in order
```

**Docker Compose (local dev):** A `docker-compose.yml` at the repo root lets developers run all 4 Forge components + Prometheus + Grafana + ELK on a laptop without Kubernetes. Used for fast local iteration; not used in the Jenkins pipeline or production deploy.

**Helm and CRDs:** CRD manifests placed under `charts/forge/crds/` are installed by `helm install` but **not upgraded by `helm upgrade`**. The `deploy` pipeline stage always runs `kubectl apply -f config/crd/` before `helm upgrade` to ensure the API server has the latest schema before the new operator version starts. A schema mismatch between operator and CRD will cause immediate reconciliation failures. After CRD apply and before operator rollout, the pipeline verifies the CRD version matches what the new operator image expects (checked via a `forge-crd-check` init container or a simple `kubectl get crd` version label check).

**Pipeline stages (9 total):**
1. Lint + vet
2. Test (envtest, go test -race)
3. Build Docker images (operator, gateway, API, webhook)
4. Trivy scan — all 4 images; fail on HIGH/CRITICAL
5. Push to Docker Hub (tagged with Git SHA — never `latest`)
6. Push to local registry (same images, for Minikube pull)
7. Apply CRDs (`kubectl apply -f config/crd/`)
8. Helm deploy (`helm upgrade --install forge ...`)
9. Rollout verify — all 4 components + post-deploy smoke test

**Docker Hub push (stage 5):** Jenkins reads `DOCKER_HUB_CREDENTIALS` from Vault (see §7 Secrets Management). Images are pushed as `{dockerhub-user}/forge-{component}:{git-sha}`. Local registry (stage 6) is kept as the pull source for Minikube since Docker Hub rate limits can break CI.

**Rollout verification (stage 8):**
```bash
kubectl rollout status deployment/forge-operator  -n forge-system --timeout=120s
kubectl rollout status deployment/forge-gateway   -n forge-system --timeout=120s
kubectl rollout status deployment/forge-api       -n forge-system --timeout=120s
kubectl rollout status deployment/forge-webhook   -n forge-system --timeout=120s
# Smoke test: create a test environment, wait for ACTIVE, then delete it
./hack/smoke-test.sh
```
If any rollout times out or the smoke test fails, the pipeline stage fails and the previous release remains active.

---

## 6. Technology Stack & Rationale

| Layer | Technology | Why |
|---|---|---|
| Main language | Go | Kubernetes ecosystem is Go-native; controller-runtime requires Go; goroutines suit concurrent reconciliation |
| Kubernetes operator | controller-runtime | Standard library for K8s operators; handles watch/reconcile loop boilerplate, leader election, metrics |
| Local cluster | Minikube | Laptop-runnable; satisfies SPE constraint; easy to reset between demos |
| CI/CD | Jenkins | Runs locally; no external dependency; satisfies SPE default toolchain |
| Containerization | Docker | Standard; multi-stage builds keep Forge images small |
| Config management | Ansible | SPE default; playbooks configure the laptop environment (Minikube, kubectl, Helm, registry) |
| K8s packaging | Helm | Templates Forge's own K8s manifests; parameterises image tags per pipeline run |
| Monitoring | Prometheus + Grafana | SPE default; Forge exposes custom metrics via `/metrics` endpoint using `prometheus/client_golang` |
| Logging | ELK Stack (Elasticsearch + Logstash + Kibana) | Mandatory SPE requirement; Forge components emit structured JSON logs to stdout; Filebeat sidecar ships them to Logstash |
| Secret management | HashiCorp Vault | Stores Docker Hub credentials, Forge API token, GitHub webhook secret; Vault Agent sidecar injects secrets as files at runtime |
| Local dev | Docker Compose | `docker-compose.yml` at repo root; spins up all 4 Forge components + Prometheus + Grafana + ELK on a laptop without Kubernetes |
| Image scanning | Trivy | Single binary; zero config; scans in CI with `--exit-code 1`; DevSecOps extension |
| Image registry | Local Docker registry via `host.minikube.internal:5000` | `localhost:5000` on the laptop is not `localhost` inside the Minikube node VM. Use `host.minikube.internal:5000` as the registry address inside the cluster, and start the registry with `--insecure-registry host.minikube.internal:5000` flag on Minikube. Alternatively use the Minikube registry addon (`minikube addons enable registry`). |
| Gateway | Go `net/http` | Full control over routing, hibernation detection, wake-up deduplication logic |

### Why Not…

| Rejected option | Reason |
|---|---|
| Argo CD (GitOps) | Adds complexity; Helm + Jenkins achieves the same outcome for this scope |
| Istio service mesh | Heavy; overkill for local Minikube; would consume most of the laptop's RAM |
| Crossplane | Designed for cloud provisioning, not local K8s lifecycle management |
| Backstage | Frontend portal; Forge's value is the controller logic, not a UI portal |
| CRIU process checkpointing | Out of scope; stateless apps don't need it; adds weeks of complexity |

---

## 7. Security Model & Resource Governance

### Container Security (per environment)

Every managed application environment runs with:

```yaml
securityContext:
  runAsNonRoot: true
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
```

**Why:** Preview environments run code from feature branches — potentially untested or experimental. Dropping all capabilities and enforcing non-root reduces blast radius if an app behaves badly.

**`readOnlyRootFilesystem` caveat:** Many frameworks write to `/tmp` or internal cache directories at startup. With `readOnlyRootFilesystem: true` these writes fail silently or crash the app. Fix: mount an `emptyDir` volume at `/tmp` in every pod template. The sample application used for the demo must be verified to start correctly under the full securityContext before the demo.

```yaml
volumes:
  - name: tmp
    emptyDir: {}
volumeMounts:
  - name: tmp
    mountPath: /tmp
```

### Namespace Isolation & NetworkPolicy

Each environment gets its own Kubernetes namespace. A default-deny NetworkPolicy is applied to each namespace:

```yaml
# Applied to every environment namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```

**Required allow rules (must be added or isolation breaks the product):**

| Rule | Why |
|---|---|
| Ingress from Forge gateway namespace | Gateway must reach app Pods to forward traffic |
| Egress to `kube-dns` (port 53 UDP/TCP) | App containers need DNS resolution; default-deny blocks it |
| Egress to app's own dependencies (if any) | Database, external APIs — configure per environment |

Without the gateway ingress rule and the DNS egress rule, default-deny succeeds as isolation but the environment never serves traffic.

**Limitation to state explicitly:** Kubernetes namespaces are not a hard security boundary against hostile code. Security boundary for this project: isolation between preview environments. We do not claim isolation from the cluster control plane.

### Forge Component RBAC

| Component | Permissions needed | Why |
|---|---|---|
| Operator | `ClusterRole`: full CRUD on Environments, Namespaces, Deployments, Services, Ingresses, ResourceQuotas | Manages cluster-wide resources |
| Gateway | `ClusterRole`: `get`/`watch`/`list` Environments; `patch` Environment status | Reads state, records last-activity timestamp |
| Webhook receiver | `ClusterRole`: `create`/`patch` Environments | Translates GitHub events to CRD writes |
| API server | Same as webhook receiver | REST façade over CRD operations |

Principle: each component gets only the verbs it actually needs. The gateway cannot create or delete anything.

### Secrets Management

All secrets are stored in **HashiCorp Vault** (deployed to `forge-system` namespace in dev mode for the demo). Vault Agent sidecars inject secrets as files into each component's Pod at startup — no secrets in Helm values, no secrets committed to git.

| Secret | Vault path | Consumer |
|---|---|---|
| GitHub webhook HMAC secret | `secret/forge/github-webhook` | Webhook receiver |
| Forge API bearer token | `secret/forge/api-token` | API server + Gateway |
| Docker Hub credentials | `secret/forge/dockerhub` | Jenkins (read via Vault CLI in pipeline) |
| Local registry credentials | `secret/forge/registry` | Kubernetes `imagePullSecret` |

**How it works at runtime:**
```
Pod starts → Vault Agent init container authenticates with K8s ServiceAccount
           → pulls secrets from Vault
           → writes them as files to /vault/secrets/
           → main container reads files (never env vars)
```

**Ansible role** (`roles/vault/`) handles Vault installation and initial secret seeding on the laptop.

### API Authentication

The REST API and webhook receiver are reachable from outside the cluster (via Minikube tunnel). Both require authentication:

- **REST API:** static bearer token (`Authorization: Bearer <token>`). Token stored in a Kubernetes Secret. For the demo this is a single pre-shared token; production would use OIDC.
- **Webhook receiver:** GitHub HMAC-SHA256 signature validation on every request. Reject any request without a valid `X-Hub-Signature-256` header.

### Resource Governance

```yaml
# Per-environment ResourceQuota (in each namespace)
# Requests match the LLD CRD defaults (200m CPU / 256Mi memory) so a single
# default pod fits within the quota without violating it.
hard:
  requests.cpu: "200m"
  requests.memory: "256Mi"
  limits.cpu: "500m"
  limits.memory: "512Mi"
  pods: "5"

# LimitRange — applies to pods that don't specify requests/limits explicitly
defaultRequest:
  cpu: "200m"
  memory: "256Mi"
default:
  cpu: "500m"
  memory: "512Mi"
```

**Global limits:**
- Maximum 10 active environments (configurable)
- Maximum 3 environments per developer
- New environments queue if at capacity; eligible idle ones are hibernated first (lowest priority, LRU)
- Maximum 3 concurrent restorations (configurable)
- Maximum 5 pending restoration requests before 429 is returned

### Image Scanning (Trivy)

All images are scanned in CI before deployment. HIGH and CRITICAL CVEs block the pipeline. This applies to both Forge's own images and (in the full implementation) images submitted for managed environments.

---

## 8. Observability & Metrics

### Prometheus Metrics (custom, exposed by Forge)

> **Prometheus naming:** `_total` suffix is for Counters only. Gauges have no suffix.
> **Cardinality:** Never use PR number or environment name as a metric label — that creates one time series per PR and will make Prometheus slow. Use bounded labels only: `state`, `result`, `type`, `reason`.

| Metric | Type | Labels | Notes |
|---|---|---|---|
| `forge_environments` | Gauge | `state` | Count of environments per state. Read from CRD list, not inside reconcile. |
| `forge_provisioning_duration_seconds` | Histogram | `result` | Time from acceptance to ACTIVE. |
| `forge_provisioning_failures_total` | Counter | `reason` (timeout/image_pull/quota/other) | Incremented **only on the actual transition** to PROVISIONING_FAILED. |
| `forge_restoration_duration_seconds` | Histogram | `from` (warm/deep), `result` | Measured from `RestorationStartedAt` to ACTIVE. Report p50/p95/p99 in Grafana. |
| `forge_restoration_failures_total` | Counter | `reason` | Incremented only on the actual transition to RESTORATION_FAILED. |
| `forge_hibernations_total` | Counter | `type` (warm/deep) | Incremented only when the state transition actually occurs, not on every reconcile. |
| `forge_terminations_total` | Counter | `reason` (ttl/pr_closed/manual) | — |
| `forge_environment_evictions_total` | Counter | `evicted_priority` | LRU evictions under capacity pressure. |
| `forge_active_pods` | Gauge | — | Ready Pods only (not just Running). |
| `forge_environment_ready` | Gauge | — | Environments in ACTIVE with all Pods ready. |
| `forge_cpu_requests_saved_millicores` | Gauge | — | Sum of `spec.resources.cpuRequest` for hibernated environments. From CRD specs, not node metrics. |
| `forge_memory_requests_saved_bytes` | Gauge | — | Same approach for memory. |
| `forge_restoration_queue_depth` | Gauge | — | Pending restoration requests. |
| `forge_restoration_queue_full_total` | Counter | — | 429s returned due to full queue. |
| `forge_restoration_queue_wait_seconds` | Histogram | — | Time waiting for a restoration slot. |
| `forge_image_pull_duration_seconds` | Histogram | `result` | Time for image pull during restoration. |
| `forge_gateway_requests_total` | Counter | `state` (active/hibernated/unknown) | All inbound gateway requests. |
| `forge_gateway_upstream_errors_total` | Counter | `reason` | Errors forwarding to live environments. |
| `forge_orphaned_resources` | Gauge | `kind` | K8s objects with Forge labels but no matching Environment CRD. |

### Grafana Dashboards

**Dashboard 1: Environment Lifecycle**
- Current environments by state (pie chart)
- Provisioning duration over time (p50/p95/p99)
- Restoration latency by hibernation type (warm vs deep) — p50/p95/p99 — key benchmark
- Provisioning and restoration failure counts by reason
- Eviction count over time

**Dashboard 2: Resource Efficiency**
- Ready Pod count over time (not just running)
- CPU/memory requests reclaimed by hibernated environments (from CRD specs)
- Environments created/terminated per hour
- Restoration queue depth and wait time

**Dashboard 3: Pipeline Health** (for SPE demo)
- Pipeline run duration
- Trivy scan results (CVEs found/blocked)
- Deployment success rate

### Design Choice: Custom Metrics Over Just Infra Metrics

Standard Prometheus + cAdvisor gives CPU/memory of running Pods. But they don't tell you how many environments are hibernated or what the restoration latency is. Forge exposes domain-specific metrics that directly measure the platform's value proposition.

---

### ELK Stack (Logging)

Forge uses **ELK** (Elasticsearch + Logstash + Kibana) for centralised log aggregation — mandatory SPE requirement alongside Prometheus for metrics.

**How logs flow:**

```
Forge Pod (stdout JSON logs)
  → Filebeat sidecar container
  → Logstash (port 5044) — parses + enriches with component/environment labels
  → Elasticsearch (stores + indexes)
  → Kibana (visualises)
```

**Deployment:** ELK runs in the `forge-logging` namespace on Minikube, deployed via Helm (`elastic/elasticsearch`, `elastic/logstash`, `elastic/kibana` charts).

**What Forge logs (structured JSON on stdout):**

| Component | Key log events |
|---|---|
| Operator | Reconcile start/end, state transitions, provisioning errors |
| Gateway | Request received, wake triggered, restore complete, 429 returned |
| API server | Environment created/deleted, restore triggered |
| Webhook | PR event received, environment create/delete dispatched |

**Filebeat sidecar (added to all 4 Helm Deployment templates):**
```yaml
- name: filebeat
  image: docker.elastic.co/beats/filebeat:8.11.0
  args: ["-c", "/etc/filebeat/filebeat.yml", "-e"]
  volumeMounts:
  - name: filebeat-config
    mountPath: /etc/filebeat
```

**Kibana dashboards:**
- **Dashboard 1:** Forge component logs by level (INFO/WARN/ERROR) over time
- **Dashboard 2:** State transition log stream per environment (trace a single PR through its lifecycle)
- **Dashboard 3:** Error log count by component — shows which component has most failures

**Metrics vs Logs distinction:**
- Prometheus answers "how many environments are hibernated right now?"
- Kibana answers "why did environment `myrepo-pr-42` fail to restore at 14:32?"

---

## 9. Test Strategy

Without tests, state-machine bugs only surface during the live demo. Two layers are required.

### Layer 1: Operator unit + integration tests (envtest)

`controller-runtime` ships `envtest` — a local Kubernetes API server that runs in-process. It is the standard way to test operators without a real cluster.

```
pkg/
  controller/
    environment_controller_test.go   ← envtest-based tests
  lifecycle/
    manager_test.go                  ← pure unit tests
  gateway/
    handler_test.go                  ← pure unit tests (mock informer)
```

**Operator test cases (map directly to the event × state matrix):**

| Test | What it verifies |
|---|---|
| Provision happy path | CRD created → namespace + Deployment + Service + Ingress exist |
| Duplicate create | Second create of same name is a no-op |
| Warm hibernation | After inactivity threshold, Deployment replicas = 0 |
| Deep hibernation | After deep threshold, Deployment + Service deleted; CRD spec retained |
| Restore from warm | Replicas patched back to 1; state → ACTIVE |
| Restore from deep | Deployment + Service recreated from CRD spec; state → ACTIVE |
| PR closed mid-restore | State → TERMINATING; all resources deleted |
| Synchronize while hibernated | CRD spec updated; restore + redeploy on next wake |
| TTL expiry | Environment terminated regardless of pin |
| Controller restart | Restart envtest operator; reconcile loop recovers correct state |
| Finalizer cleanup | Delete CRD; verify namespace gone |
| Orphan detection | Create Deployment with no owner Environment; verify it is cleaned up |

### Layer 2: Gateway unit tests

Pure Go tests, no Kubernetes required. Use a mock informer to inject environment state.

| Test | What it verifies |
|---|---|
| Route active environment | Request forwarded to correct backend |
| Route hibernated → wake | Restore triggered; request forwarded after readiness |
| Concurrent wake deduplication | 20 goroutines hit same hibernated env; exactly 1 restore fires |
| Wake timeout | App never ready → 503 returned; wake lock released |
| Unknown hostname | 404 returned |
| Terminating environment | 410 Gone returned |
| Browser client waking page | Accept: text/html → returns auto-refresh HTML |

### Running tests

```bash
# Operator tests (requires envtest binaries)
go test ./pkg/controller/... -v

# Gateway tests (no cluster needed)
go test ./pkg/gateway/... -v

# All tests
go test ./...
```

Envtest binaries are downloaded via `setup-envtest` from `sigs.k8s.io/controller-runtime/tools/setup-envtest`. Add this to the CI pipeline Stage 2.

---

## 11. Scope, Phases & Milestones

### Phases

| Phase | What | Priority |
|---|---|---|
| 1 | Core control plane: API, CRD, operator, webhook, provisioning, state machine, TTL termination | Must |
| 2 | Resource-efficient lifecycle: warm hibernation, deep hibernation, image reuse, restoration, finalizer cleanup | Must |
| 3 | Traffic-aware gateway: routing, hibernation detection, wake-up, deduplication, timeouts | Must |
| 4 | Governance + observability: ResourceQuota, capacity limits, Prometheus metrics, Grafana dashboards | Must |
| SPE | CI/CD pipeline: Jenkins, Docker, Trivy, Helm, Ansible, Minikube deploy | Must |

### Milestones

| Date | Target | Deliverable |
|---|---|---|
| 2026-11-04 | Kickoff | Team registered, topic abstract submitted |
| 2026-11-11 | Proposal | 1-page proposal: app, architecture, tools, extension |
| 2026-11-25 | Pipeline working | Jenkins pipeline builds, scans, pushes all 4 images; deploys Forge to Minikube |
| 2026-12-02 | Core lifecycle | Provisioning + hibernation + gateway routing working end-to-end |
| 2026-12-09 | Full system | Restoration + termination + Prometheus + Grafana live |
| 2026-12-14 | Demo rehearsal | Full demo dry run; smoke test passes |
| 2026-12-15 | Final demo & viva | Live end-to-end demo + individual Q&A |

### Work Split (recommended)

| Team member A | Team member B |
|---|---|
| Operator (CRD + controller-runtime + reconciler) | Gateway (reverse proxy + hibernation detection + wake-up) |
| Lifecycle manager (inactivity detection + state transitions) | CI/CD pipeline (Jenkins + Docker + Trivy + Helm) |
| Environment API (REST endpoints) | Ansible playbooks (machine setup) |
| Finalizer-based cleanup + orphan detection | Prometheus metrics + Grafana dashboards |

Both members must have commits across both Go and DevOps work for full collaboration marks.

### Scope Cut Line

If time runs short, cut in this order (later = cut first):

1. Keep: full Phase 1 + Phase 2 + SPE pipeline
2. Keep if possible: Phase 3 gateway (basic routing + wake-up without deduplication)
3. Cut if needed: concurrent wake-up deduplication (replace with simple sequential approach)
4. Cut entirely: snapshot-based data restoration, predictive pre-warming, request buffering, CRIU

---

## 12. Open Questions & Future Extensions

### Open Questions

| Question | Status | Notes |
|---|---|---|
| Sample app for demo | Decide by Nov 8 | Need a simple containerized app (Go HTTP service recommended) |
| Jenkins vs GitHub Actions | Decide by Nov 11 | Jenkins = local, no dependency; Actions = easier setup |
| Stateful environment support | Decide by Nov 15 | PersistentVolumes add complexity; may skip for MVP |
| Minikube networking for `*.dev.local` | Investigate | Need to configure `/etc/hosts` or local DNS for wildcard subdomains |
| Two-member work split | Decide by Nov 4 | See recommended split above |

### Future Extensions (out of scope for Dec demo)

- **Snapshot-based restoration:** Database snapshots before deep hibernation; restore from snapshot on wake
- **Predictive pre-warming:** Analyse PR activity patterns and pre-warm environments before they're accessed
- **Request buffering:** Buffer incoming requests during restoration instead of holding connections open
- **gVisor/Kata isolation:** Stronger sandboxing for environments running untrusted branch code
- **Multi-cluster management:** Spread environments across multiple clusters for capacity
- **CRIU process checkpointing:** Save and restore running process memory (very advanced, not needed for stateless apps)
- **Cost attribution:** Track per-developer resource consumption and report it

---

*This is a living document. When a design decision changes, add a dated note under the relevant section explaining what changed and why. Do not delete old decisions — mark them as superseded.*
