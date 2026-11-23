# Gateway Request Flow

```mermaid
flowchart TD
    REQ(["Incoming request\npayments-pr-101.dev.local"])

    REQ --> PARSE["Parse hostname\nstrip port, validate domain suffix\nvalidate K8s name pattern"]
    PARSE --> EXISTS{"Environment in\ninformer cache?"}

    EXISTS -->|no| R404["404 Not Found"]
    EXISTS -->|yes| DRIFT{"Policy block\nand stale?"}

    DRIFT -->|yes| R409["409 Conflict\nrebase required page"]
    DRIFT -->|no| STATE{"Lifecycle state?"}

    STATE -->|ACTIVE| FORWARD["Reverse proxy to Service"]
    STATE -->|PROVISIONING| WAIT["202 or waking page\nauto-refresh every 3s"]
    STATE -->|RESTORING| JOIN["Join existing wake lock\nwait on channel"]
    STATE -->|TERMINATING or TERMINATED| R410["410 Gone"]
    STATE -->|PROVISIONING_FAILED| R503A["503 with failure reason"]
    STATE -->|WARM or DEEP HIBERNATED| LOCK
    STATE -->|RESTORATION_FAILED| LOCK

    LOCK{"Wake lock held\nfor this environment?"}
    LOCK -->|yes| JOIN
    LOCK -->|no — this caller owns the wake| SLOT{"Restore slot\navailable?"}

    SLOT -->|queue full| R429["429 Too Many Requests\nRetry-After header"]
    SLOT -->|acquired| TRIGGER["POST /restore with bearer token\nindependent context, not request context"]

    TRIGGER --> POLL["Poll CRD state every 2s\nmax 60s"]
    POLL --> READY{"State ACTIVE?"}

    READY -->|yes| RELEASE["Close channel while holding lock\nthen delete entry\nrelease restore slot"]
    RELEASE --> FORWARD
    READY -->|timeout or failed| ERR["Close channel with error\nrelease slot\n503"]

    JOIN --> JOINED{"Wake succeeded?"}
    JOINED -->|yes| FORWARD
    JOINED -->|no| ERR2["503 Restoration failed"]

    FORWARD --> BANNER{"Policy warn, stale,\nHTML, uncompressed, 200?"}
    BANNER -->|yes| INJECT["Insert banner div\nrecalculate Content-Length"]
    BANNER -->|no| PASS["Pass response through"]
    INJECT --> RESP(["Response to client"])
    PASS --> RESP

    FORWARD --> ACT["Debounced patch\nlastActivityAt\nmax once per 10s"]
```

## Critical ordering rules

**Deduplicate before acquiring a slot.** If 20 callers arrive at a hibernated environment, the wake lock must dedupe first so exactly one caller reaches the semaphore. Acquiring the slot first means 20 callers consume the entire restore budget and the queue reports itself full against a single environment.

**Close the channel while still holding the lock, then delete the entry.** Deleting first opens a window where a new caller finds no entry, creates its own, and starts a second concurrent restoration of the same environment.

**Use an independent context for the restore, not the request context.** If the restore is tied to the first caller's request context, that caller navigating away cancels a restoration the other 19 are waiting on.

**`RESTORATION_FAILED` enters the wake path, not an error page.** A real request is the signal that a human is waiting, which is exactly when spending a retry attempt is justified. Predictive signals deliberately do not retry failed restorations.

## Status codes

| Code | When |
|---|---|
| 200 | Proxied successfully |
| 202 | Environment provisioning, browser gets the auto-refresh waking page |
| 404 | Hostname does not match any environment |
| 409 | Drift policy is `block` and the environment is stale |
| 410 | Environment is terminating or terminated |
| 429 | Restoration queue full — includes `Retry-After` |
| 503 | Restoration timed out or failed after retries |
