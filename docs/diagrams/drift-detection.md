# Environment Drift Detection

A preview environment is built from a PR branch at a point in time. While it lives, the base branch keeps moving. Drift detection measures that gap and acts on it.

```mermaid
flowchart TD
    PUSH["Push to base branch\ne.g. main"] --> WH["Webhook Receiver"]
    WH --> LIST["List Environments\nwhere spec.baseBranch == pushed branch"]
    LIST --> SKIP1{"Policy off\nor terminating?"}
    SKIP1 -->|yes| DONE["Skip"]
    SKIP1 -->|no| CMP["GitHub compare API\nbase HEAD vs merge base"]

    CMP -->|API error| ERR["Record lastCheckError\nkeep previous numbers\nnever fail the environment"]
    CMP -->|success| EVAL{"commitsBehind >= threshold\nOR branchPoint older than maxAge?"}

    EVAL -->|no| CLEAR["stale = false"]
    EVAL -->|yes| MARK["stale = true\npatch status.drift"]

    MARK --> POLICY{"spec.drift.policy"}
    POLICY -->|warn| BANNER["Gateway injects banner\ninto HTML responses"]
    POLICY -->|block| BLOCK["Gateway returns 409\nrebase required page"]
    POLICY -->|auto-rebuild| COOL{"Last rebuild\nwithin 1h?"}

    COOL -->|yes| DONE
    COOL -->|no| ACT{"Environment ACTIVE?"}
    ACT -->|no| DEFER["Defer to next wake\ndo not burn a slot"]
    ACT -->|yes| BUILD["Trigger CI rebuild\nstamp lastRebuildAt"]
```

## Why each guard exists

| Guard | Without it |
|---|---|
| Filter by `baseBranch` + `baseRepo` | Every push to any branch recomputes drift for every environment |
| Skip terminating environments | Wasted GitHub API calls on environments about to disappear |
| Error never fails the environment | A GitHub outage would break previews that are otherwise fine |
| 1-hour rebuild cooldown | Rebuild → `synchronize` → recheck → still behind → rebuild, an unbounded loop on a busy base branch |
| Skip rebuild when hibernated | Rebuilding something nobody is looking at spends a restoration slot and a build |
| Recompute merge base on `synchronize` | A rebase moves the merge base; comparing against the old one reports drift the developer already fixed |

## Banner injection guards

Rewriting a proxied response body is the most dangerous thing the gateway does. Injection happens only when every condition holds:

```mermaid
flowchart LR
    RESP["Proxied response"] --> C1{"Status 200?"}
    C1 -->|no| PASS["Pass through\nuntouched"]
    C1 -->|yes| C2{"Content-Type\ntext/html?"}
    C2 -->|no| PASS
    C2 -->|yes| C3{"Content-Encoding\nempty?"}
    C3 -->|no| PASS
    C3 -->|yes| C4{"body tag in\nfirst 8KB?"}
    C4 -->|no| PASS
    C4 -->|yes| INJ["Insert banner div\nrecalculate Content-Length"]
```

A stale `Content-Length` truncates the page in every browser, so it is always recalculated. The banner is a single `div` with inline styles and no script, so a strict Content-Security-Policy on the application cannot break it, and it cannot break the application.

## Two triggers, two failure modes

| Repo type | Commit threshold alone | Age threshold alone | Both |
|---|---|---|---|
| Busy (100 commits/day) | Fires constantly, becomes noise | Never fires meaningfully | Threshold raised per environment |
| Quiet (3 commits/month) | Never fires | Catches a month-old branch point | Correct in both cases |

Either trigger firing marks the environment stale. `commitThreshold: 0` disables the count trigger; an empty `maxAge` disables the age trigger.
