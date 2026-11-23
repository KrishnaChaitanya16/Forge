# CI/CD Pipeline

```mermaid
flowchart TD
    PUSH(["git push to GitHub"])
    PUSH --> TRIGGER["Jenkins GitHub Hook Trigger\nfor GITScm Polling"]
    TRIGGER --> S1

    subgraph S1["Stage 1 — Quality"]
        Q["go fmt check\ngo vet\ngolangci-lint"]
    end

    S1 --> S2

    subgraph S2["Stage 2 — Test"]
        T["go test -race ./...\nenvtest integration tests\nsetup-envtest binaries"]
    end

    S2 --> S3

    subgraph S3["Stage 3 — Build"]
        B["docker build all 4 images\noperator · gateway · api · webhook\ntagged with git SHA"]
    end

    S3 --> S4

    subgraph S4["Stage 4 — Trivy Scan"]
        TRIVY["trivy image on all 4 images\n--severity HIGH,CRITICAL\n--exit-code 1"]
    end

    S4 --> CVE{"HIGH or CRITICAL\nCVE found?"}
    CVE -->|yes| FAIL(["FAILED — deployment blocked"])
    CVE -->|no| S5

    subgraph S5["Stage 5 — Push to Docker Hub"]
        DH["docker login via Vault credentials\npush USER/forge-COMPONENT:SHA"]
    end

    S5 --> S6

    subgraph S6["Stage 6 — Push to local registry"]
        LR["push to localhost:5000\nMinikube pull source\navoids Docker Hub rate limits"]
    end

    S6 --> S7

    subgraph S7["Stage 7 — Apply CRDs"]
        CRDS["kubectl apply -f config/crd/\nschema must land before\nthe new operator starts"]
    end

    S7 --> S8

    subgraph S8["Stage 8 — Helm Deploy"]
        HELM["helm upgrade --install forge\n--set image.tag=SHA"]
    end

    S8 --> S9

    subgraph S9["Stage 9 — Verify"]
        RO["rollout status on all 4 components\n--timeout=120s"]
        SMOKE["hack/smoke-test.sh\ncreate env, wait ACTIVE, delete"]
        RO --> SMOKE
    end

    S9 --> SUCCESS(["PASSED — Forge deployed"])
```

## Why the stage order matters

| Order constraint | Reason |
|---|---|
| Trivy before any push | A vulnerable image must never reach a registry, not even a local one |
| Docker Hub before local registry | The rubric requires Docker Hub; local registry is the pull source for Minikube |
| CRDs before Helm | `helm upgrade` does **not** upgrade CRDs placed in `charts/forge/crds/`. A new operator starting against an old schema fails every reconcile immediately |
| Rollout status before smoke test | Testing against a half-rolled deployment gives a false failure |
| Smoke test last | It is the only stage that proves the deployed system actually works, not just that Pods started |

## Secrets in the pipeline

Docker Hub credentials come from Vault, not from Jenkins credentials store or environment variables in the Jenkinsfile:

```
Jenkins stage → vault read secret/forge/dockerhub → docker login → push
```

Nothing in the repository or the Jenkinsfile contains a credential.
