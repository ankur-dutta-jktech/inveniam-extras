---
name: qa-debugging
description: >-
  Debug and investigate issues in QA Kubernetes environments. Use when the user
  asks to check logs, investigate errors, inspect pods, or troubleshoot services
  in any QA environment (qa1 through qa16).
---

# QA Environment Debugging

## Environment Layout

QA environments run on Kubernetes. Each QA environment has three namespace tiers:

| Tier | Namespace pattern | Contains |
|------|-------------------|----------|
| Application | `qa{N}-cluster` | Core backend services, frontend |
| Hub | `hub-qa{N}-cluster` | Hub/blockchain gateway services |
| ImmuDB | `inveniam-immudb-qa{N}-cluster` | Immutable ledger databases |

Where `{N}` is `1` through `16`.

## Service ↔ Pod Name Mapping

Pod names follow the pattern `inv-<short>-<replicaset>-<hash>`. Map between repo app names and pod prefixes:

| Pod prefix | App / repo directory |
|---|---|
| `inv-deals` | `deals-service` |
| `inv-hub` | `hub-service` |
| `inv-organisations` | `organisations-service` |
| `inv-iam` | `iam-service` |
| `inv-dataio` | `dataio-service` |
| `inv-metachain` | `metachain-service` |
| `inv-tasks` | `tasks-service` |
| `inv-workflows` | `workflows-service` |
| `inv-relay` | `relay-service` |
| `inv-ns` | `notifications-service` |
| `inv-ss` | `scheduler-service` |
| `inv-ds` | `data-room-service` |
| `inv-bs` | `batch-processing-service` |
| `inv-awf` | `automated-workflow-service` |
| `inv-ws-main` | `websockets-service` |
| `inv-front` | `frontend` |
| `inv-cron` | Cron jobs |
| `inv-ai` | AI service |
| `inv-mcp` | MCP service |
| `inv-chat-v2` | Chat service v2 |
| `inv-ui-v2` | UI v2 |
| `inv-provenance-os` | Provenance OS |

## Debugging Workflow

### Step 1 — Identify the target

Determine:
1. **Which QA environment** — e.g. `qa10`. If not specified, ask the user.
2. **Which service** — map user's description to the pod prefix above.
3. **Which namespace tier** — most services live in `qa{N}-cluster`. Hub services live in `hub-qa{N}-cluster`.

### Step 2 — Find the pod

```sh
kubectl get pods -n qa{N}-cluster | grep inv-{service}
```

If multiple pods exist for the same service (rolling deploy or multiple replicas), check all of them. Prefer the one with `Running` status and `1/1` ready.

### Step 3 — Retrieve logs

```sh
# Recent logs (last 200 lines)
kubectl logs {pod-name} -n {namespace} --tail=200

# Logs since a specific time
kubectl logs {pod-name} -n {namespace} --since=1h

# Follow live logs
kubectl logs {pod-name} -n {namespace} -f --tail=50
```

### Step 4 — Analyze the logs

Logs are **structured JSON**, one object per line. Key fields:

| Field | Description |
|-------|-------------|
| `level` | `INFO`, `WARN`, `ERROR` |
| `msg` | Log message / handler name |
| `component` | `app`, `NestAwsPubSub`, `SnsProducer`, `incoming-http-request`, `outgoing-http-response` |
| `context.errorStack` | Full stack trace (present on errors) |
| `context.correlationId` | Request correlation ID for tracing across services |
| `context.userId` | User who triggered the action |
| `context.organizationId` | Organisation context |
| `args.payload` | Event/command payload |
| `time` | ISO 8601 timestamp |

**Triage approach:**

1. First, filter for errors and warnings:
   ```sh
   kubectl logs {pod} -n {ns} --tail=500 2>&1 | grep -i '"level":"ERROR\|"level":"WARN"'
   ```
2. If errors found, pipe through `python3 -m json.tool` for readability.
3. Check `context.errorStack` for the root cause.
4. Use `context.correlationId` to trace the same request across multiple services.
5. Look at `component` and `msg` to identify which handler failed.

### Step 5 — Cross-service tracing

If an error spans services (e.g. deals-service publishes an event that hub-service handles):

1. Find the `correlationId` from the originating service's logs.
2. Search for that ID in the downstream service:
   ```sh
   kubectl logs {downstream-pod} -n {ns} --tail=500 2>&1 | grep {correlationId}
   ```

### Step 6 — Check pod health

```sh
# Pod status and restarts
kubectl get pods -n {namespace} | grep inv-{service}

# Detailed pod info (events, conditions, restart reasons)
kubectl describe pod {pod-name} -n {namespace}

# Resource usage
kubectl top pod {pod-name} -n {namespace}
```

Look for: high restart counts, `CrashLoopBackOff`, `OOMKilled` in events, or pods stuck in `0/1 Running`.

## Reporting Results

When presenting findings to the user:

1. **State clearly** whether errors were found or not.
2. **For each error**, report: timestamp, handler/component, error message, and a brief interpretation.
3. **Distinguish** between errors related to the current investigation vs. unrelated background errors.
4. **Highlight** any successful flows (e.g. events published and handled) to confirm what IS working.
5. If no errors found, confirm the service is healthy and mention key successful operations observed.
