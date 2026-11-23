# Environment Lifecycle State Machine

```mermaid
stateDiagram-v2
    [*] --> PROVISIONING : PR opened / API request

    PROVISIONING --> ACTIVE : all workloads ready
    PROVISIONING --> PROVISIONING_FAILED : timeout 10m / K8s error

    ACTIVE --> WARM_HIBERNATED : idle past warm threshold
    ACTIVE --> DEEP_HIBERNATED : capacity eviction
    ACTIVE --> TERMINATING : PR closed / TTL expired

    WARM_HIBERNATED --> DEEP_HIBERNATED : idle past deep threshold
    WARM_HIBERNATED --> RESTORING : HTTP request
    WARM_HIBERNATED --> RESTORING : predictive signal
    WARM_HIBERNATED --> TERMINATING : PR closed / TTL expired

    DEEP_HIBERNATED --> RESTORING : HTTP request
    DEEP_HIBERNATED --> RESTORING : predictive signal
    DEEP_HIBERNATED --> TERMINATING : PR closed / TTL expired

    RESTORING --> ACTIVE : all workloads ready
    RESTORING --> RESTORATION_FAILED : timeout 60s

    RESTORATION_FAILED --> RESTORING : backoff retry (max 3)
    RESTORATION_FAILED --> TERMINATING : PR closed / TTL expired
    PROVISIONING_FAILED --> PROVISIONING : new commit / manual retry
    PROVISIONING_FAILED --> TERMINATING : PR closed / TTL expired

    TERMINATING --> TERMINATED : namespace confirmed gone
    TERMINATED --> [*]
```

## Predictive wake path

A predictive signal moves a hibernated environment to RESTORING before any request arrives. It is skipped entirely if the environment is not hibernated, is past TTL, has no free capacity slot, or the sender is a bot.

```mermaid
flowchart LR
    GH["GitHub event\nreview_requested\ncheck_suite success\nPR comment"] --> WH["Webhook Receiver"]
    WH --> C1{"Sender is bot?"}
    C1 -->|yes| SKIP["Skip\nrecord metric"]
    C1 -->|no| C2{"State hibernated?"}
    C2 -->|no| SKIP
    C2 -->|yes| C3{"TTL expired?"}
    C3 -->|yes| SKIP
    C3 -->|no| C4{"Capacity free?"}
    C4 -->|no| SKIP
    C4 -->|yes| PATCH["Patch state to RESTORING\ntrigger = predictive"]
    PATCH --> OP["Operator restores\nall workloads"]
```

## State reference

| State | What exists | How it exits |
|---|---|---|
| `PROVISIONING` | Namespace, quota, NetworkPolicy being created | → ACTIVE on ready, → PROVISIONING_FAILED on 10m timeout |
| `ACTIVE` | All workloads running, gateway forwarding, inactivity timer ticking | → WARM on idle, → DEEP on eviction, → TERMINATING on close/TTL |
| `WARM_HIBERNATED` | All workloads at replicas 0, Service + Ingress retained, HPAs captured and deleted | → DEEP on deeper idle, → RESTORING on request or signal, → TERMINATING |
| `DEEP_HIBERNATED` | Forge-owned workloads deleted, user workloads at 0, Ingress retained, inventory in CRD status | → RESTORING on request or signal, → TERMINATING |
| `RESTORING` | Workloads being recreated or scaled to `OriginalReplicas`, HPAs restored after | → ACTIVE when all ready, → RESTORATION_FAILED on 60s timeout |
| `RESTORATION_FAILED` | Nothing running; attempt counter held in status | → RESTORING via backoff (30s/60s/120s, max 3), → TERMINATING |
| `PROVISIONING_FAILED` | Namespace may exist; no workloads | → PROVISIONING on new commit, → TERMINATING |
| `TERMINATING` | Namespace deletion in flight; finalizer held | → TERMINATED once namespace confirmed gone |
| `TERMINATED` | Nothing | Terminal — CRD removed |

**Why the Ingress survives deep hibernation:** the gateway routes by hostname. Deleting the Ingress would stop the hostname resolving, so the wake-up request could never reach the gateway and the environment would be unreachable forever. The Ingress points at the gateway Service, not at the app, so it costs nothing while the environment sleeps.
