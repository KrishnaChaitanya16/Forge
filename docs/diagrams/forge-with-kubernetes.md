# Forge + Kubernetes Components

```mermaid
flowchart TD
    subgraph DEV["Developer"]
        GH["GitHub PR"]
        BROWSER["Browser"]
    end

    subgraph FORGE["Forge (forge-system namespace)"]
        WH["Webhook"]
        FAPI["Environment API"]
        OP["Operator"]
        GW["Gateway"]
        CAP["Capacity Worker"]
    end

    subgraph K8S["Kubernetes Control Plane"]
        KAPI["kube-apiserver"]
        ETCD["etcd"]
        KCM["kube-controller-manager"]
        KSCHED["kube-scheduler"]
    end

    subgraph NODE["Worker Node"]
        KUBELET["kubelet"]
        CRI["containerd"]
        DEP["Deployment"]
        POD["App Pod"]
        SVC["Service"]
    end

    subgraph OBS["Observability"]
        PROM["Prometheus"]
        GRAF["Grafana"]
    end

    GH -->|webhook POST| WH
    WH --> FAPI
    BROWSER -->|REST| FAPI
    BROWSER -->|preview traffic| GW

    FAPI -->|create Environment CRD| KAPI
    OP -->|create Namespace, Deployment, Service| KAPI
    CAP -->|list + patch Environments| KAPI
    GW -->|watch CRDs via informer| KAPI
    GW -->|trigger restore| FAPI

    KAPI <--> ETCD
    KAPI -->|watch events| OP
    KAPI -->|watch events| GW
    KCM -->|manages ReplicaSet + Pod| KAPI
    KSCHED -->|assigns Pod to node| KAPI

    KUBELET -->|reports status| KAPI
    KUBELET --> CRI
    CRI --> POD
    DEP --> POD
    SVC --> POD

    GW -->|forward after wake| SVC

    OP --> PROM
    GW --> PROM
    PROM --> GRAF
```

| Layer | Components | Owner |
|---|---|---|
| Forge | Operator, Gateway, API, Webhook, Capacity | Your code |
| K8s Control Plane | kube-apiserver, etcd, kube-controller-manager, kube-scheduler | Kubernetes |
| Worker Node | kubelet, containerd, Pods | Kubernetes node |
| Observability | Prometheus, Grafana | Infrastructure |
