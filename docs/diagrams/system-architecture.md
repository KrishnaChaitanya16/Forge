# System Architecture

```mermaid
flowchart TD
    subgraph DEV["Developer Interface"]
        GH["GitHub PR Events\nopen · update · close\nreview · comment · CI"]
        PUSH["Push to base branch"]
        RAPI["REST API\nPOST /api/v1/environments"]
        BROWSER["Reviewer browser"]
    end

    subgraph CP["Forge Control Plane"]
        WH["Webhook Receiver\nsignature validation\npredictive wake\ndrift detection"]
        EAPI["Environment API\nREST + bearer auth"]
        OP["Forge Operator\ncontroller-runtime reconciler\nwatches Environment CRDs"]
        LM["Lifecycle Manager\ninactivity · TTL\npredictive wake TTL"]
        RM["Capacity Worker\nadmission · priority LRU eviction"]
    end

    subgraph K8S["Kubernetes — Minikube"]
        CRD["Environment CRD\nforge.dev/v1alpha1"]
        NS["Namespace per environment"]
        WL["Deployments · CronJobs\nService · Ingress"]
        RQ["ResourceQuota · LimitRange\nNetworkPolicy"]
    end

    subgraph DP["Data Plane"]
        GW["Forge Gateway\nhostname routing\nwake-lock dedup\nbounded restore queue\ndrift banner injection"]
        E1["payments-pr-101\nactive"]
        E2["payments-pr-102\nactive"]
        E3["payments-pr-103\nhibernated"]
    end

    subgraph SEC["Secrets"]
        VAULT["HashiCorp Vault\nwebhook secret · API token\nregistry credentials"]
    end

    subgraph OBS["Observability"]
        PROM["Prometheus\nlifecycle metrics"]
        GRAF["Grafana\ndashboards"]
        ELK["ELK Stack\nFilebeat · Logstash\nElasticsearch · Kibana"]
    end

    GH --> WH
    PUSH --> WH
    RAPI --> EAPI
    BROWSER --> GW

    WH --> EAPI
    WH -->|predictive wake\npatch status| CRD
    WH -->|drift status patch| CRD
    EAPI --> CRD
    CRD --> OP
    OP --> NS
    OP --> WL
    OP --> RQ
    LM --> CRD
    RM --> CRD

    GW -->|watch CRDs\nvia informer| CRD
    GW -->|trigger restore| EAPI
    GW --> E1
    GW --> E2
    WL --> E1
    WL --> E2

    VAULT -.->|agent sidecar\ninjects secrets| WH
    VAULT -.->|agent sidecar| EAPI
    VAULT -.->|agent sidecar| GW

    OP --> PROM
    GW --> PROM
    PROM --> GRAF
    OP --> ELK
    GW --> ELK
    EAPI --> ELK
    WH --> ELK
```

## Control plane vs data plane

| Plane | Components | What it does | Request path? |
|---|---|---|---|
| Control | Webhook, API, Operator, Lifecycle Manager, Capacity Worker | Decides what should exist and when | No — reacts to events and timers |
| Data | Gateway, application Pods | Carries actual preview traffic | Yes — every reviewer request |

The split matters because the control plane can be slow and eventually consistent without anyone noticing, while the data plane is on the critical path of every page load. This is why the gateway reads CRD state from an in-memory informer cache rather than calling the API server per request.

## Who writes what to the CRD

Three processes patch the same `status` subresource. Each owns a disjoint set of fields, so merge patches do not clobber each other.

| Writer | Fields it owns |
|---|---|
| Operator | `lifecycleState`, `namespaceName`, `previewURL`, `imageDigest`, `hibernatedWorkloads`, `restoredAt`, `restorationStartedAt`, `restorationAttempts`, `conditions` |
| Gateway | `lastActivityAt` only (debounced to at most once per 10s) |
| Lifecycle Manager | `lifecycleState`, `message`, `restorationTrigger` |
| Webhook Receiver | `drift`, and `lifecycleState` only for the hibernated-to-RESTORING predictive transition |

Conflicts on `lifecycleState` are resolved by losing gracefully: a predictive wake that conflicts with a concurrent hibernation is dropped, because the reactive path still covers the reviewer when they actually arrive.
