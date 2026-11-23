# Operator Reconciliation Loop

```mermaid
flowchart TD
    START(["Reconcile triggered\nCRD create · update · requeue"]) --> FETCH
    FETCH["Fetch Environment CRD"]
    FETCH --> DELETED{"deletionTimestamp set?"}

    DELETED -->|yes| TERMPATH["Set state TERMINATING"]
    DELETED -->|no| STATE{"status.lifecycleState"}

    STATE -->|empty| PROV
    STATE -->|PROVISIONING| PROV
    STATE -->|ACTIVE| ACT
    STATE -->|WARM_HIBERNATED| WARM
    STATE -->|DEEP_HIBERNATED| DEEP
    STATE -->|RESTORING| REST
    STATE -->|RESTORATION_FAILED| RFAIL
    STATE -->|TERMINATING| TERMPATH
    STATE -->|TERMINATED or PROVISIONING_FAILED| NOOP(["No-op — terminal"])

    subgraph PROV["PROVISIONING"]
        P1["Stamp provisioningStartedAt\ncompute expiresAt from TTL"]
        P2["Ensure Namespace\npersist namespaceName immediately"]
        P3["Ensure ResourceQuota\nLimitRange · NetworkPolicy"]
        P4["Ensure Deployment · Service · Ingress\nlabel forge.dev/owned=true"]
        P5{"All workloads ready?"}
        P1 --> P2 --> P3 --> P4 --> P5
    end
    P5 -->|no, under 10m| RQ1(["Requeue 5s"])
    P5 -->|no, over 10m| PFAIL["PROVISIONING_FAILED"]
    P5 -->|yes| CAPTURE["Capture image digest\nset ACTIVE · activeSince"]

    subgraph ACT["ACTIVE"]
        A1["Reconcile drift between\nspec and live workloads"]
        A2{"Spec generation changed?"}
        A1 --> A2
    end
    A2 -->|yes| PROV
    A2 -->|no| OK(["Done"])

    subgraph WARM["WARM_HIBERNATED"]
        W1{"Inventory already taken?"}
        W2["Inventory all workloads\npersist originalReplicas"]
        W3["Capture and delete HPAs\nbefore scaling"]
        W4["Scale all workloads to 0\nparallel, limit 8"]
        W1 -->|no| W2 --> W3
        W1 -->|yes| W3
        W3 --> W4
    end
    W4 --> OK

    subgraph DEEP["DEEP_HIBERNATED"]
        D1["Inventory if missing\ncapture and delete HPAs"]
        D2["Forge-owned: delete\nUser-owned: scale to 0"]
        D3["Delete Service\nRETAIN Ingress"]
        D1 --> D2 --> D3
    end
    D3 --> OK

    subgraph REST["RESTORING"]
        R1["Stamp restorationStartedAt\nif nil"]
        R2{"Elapsed over 60s?"}
        R3["Restore each workload to\noriginalReplicas, parallel"]
        R4["Restore HPAs after\nworkloads exist"]
        R5{"All workloads ready?"}
        R1 --> R2
        R2 -->|no| R3 --> R4 --> R5
    end
    R2 -->|yes| RFAILSET["Increment attempts\nset RESTORATION_FAILED"]
    R5 -->|no| RQ2(["Requeue 2s\nmessage names blocker"])
    R5 -->|yes| MEASURE["Measure duration BEFORE\nsetting restoredAt\nset ACTIVE"]

    subgraph RFAIL["RESTORATION_FAILED"]
        F1{"attempts >= 3?"}
        F2["Backoff 2^(n-1) × 30s\nreset to RESTORING"]
        F1 -->|no| F2
    end
    F1 -->|yes| HOLD(["Hold — a real request\nvia gateway resets it"])

    subgraph TERMPATH["TERMINATING"]
        T1["Delete Namespace\ncascades to all children"]
        T2{"Namespace still exists?"}
        T1 --> T2
    end
    T2 -->|yes| RQ3(["Requeue 5s — deletion\nis asynchronous"])
    T2 -->|no| FIN["Remove finalizer\nset TERMINATED"]
```

## Rules that prevent the known failure modes

| Rule | Failure it prevents |
|---|---|
| Persist `namespaceName` immediately after creating the Namespace | Later provisioner calls read an empty namespace from status and fail |
| Compute `expiresAt` at acceptance, not at readiness | An environment that spends 8h provisioning would then live another 72h |
| Inventory only when `len(hibernatedWorkloads) == 0` | Re-inventorying a half-scaled namespace records `originalReplicas: 0`, so restore brings it back dead |
| Delete HPAs **before** scaling to zero | The HPA controller observes `replicas: 0` and scales straight back to `minReplicas` |
| Restore HPAs **after** workloads exist | A dangling `scaleTargetRef` makes the HPA controller back off |
| Measure restore duration before assigning `restoredAt` | `time.Since(restoredAt)` right after setting it always reads ~0 |
| Use `restorationStartedAt` for the timeout, never `restoredAt` | `restoredAt` is nil during RESTORING, so the timeout check nil-panics |
| Requeue while the Namespace still exists | Removing the finalizer early orphans resources mid-deletion |
| Retain the Ingress through deep hibernation | The hostname stops resolving and the environment can never be woken |
| Terminal states are explicit no-ops | An unhandled state falls through the switch and reconciles forever |
