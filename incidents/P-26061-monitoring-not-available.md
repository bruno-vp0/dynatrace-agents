# Incident Report — P-26061: Monitoring Not Available

**Environment:** `https://xjr77121.apps.dynatrace.com`  
**Cluster:** `aks-demo-dt` | Resource Group: `rg-demo-dt`  
**Date:** 2026-06-28  
**Status:** CLOSED (auto-resolved at 20:35 UTC)  
**Duration:** ~54 minutes (19:41 – 20:35 UTC)

---

## Triggering Prompt

> "tell me whats going on on dynatrace and list the namespaces from the k8s cluster"

This prompt was routed by the orchestrator to both the **SRE Agent** (Dynatrace) and the **DevOps Agent** (AKS) in parallel.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 19:41 | ActiveGate pod killed during AKS node drain (`vmss000003` removed) |
| 19:41 | PVC `ag-tmp-gateway-aks-demo-dt-activegate-0` still attached to old node — `FailedAttachVolume` (Multi-Attach error) |
| 19:41 | Dynatrace detects monitoring gap → **raises P-26061** |
| ~19:46 | PVC detaches from old node; `SuccessfulAttachVolume` on `vmss000002` |
| ~19:46 | ActiveGate pod restarts and begins initialising |
| 20:15 | ActiveGate reaches `RUNNING` state; outbound comms to Dynatrace resume |
| 20:35 | Dynatrace confirms connectivity restored → **P-26061 auto-closes** |

---

## Findings

### SRE Agent (Dynatrace)

- **1 active problem found:** P-26061 — AVAILABILITY, severity 3, infrastructure impact
- **Affected entity:** Kubernetes cluster `aks-demo-dt` (`KUBERNETES_CLUSTER-29D6637CF17F4E12`)
- **Blast radius:** Zero — no services, SLOs, or end-users impacted
- **79 Dynatrace events** recorded, all periodic refreshes of the same correlation event
- No other critical or high-severity problems in the environment during this window

### DevOps Agent (AKS)

- **Cluster API server:** Healthy
- **ActiveGate pod:** `1/1 Running`, 0 restarts, 0 errors, 0 timeouts
- **Outbound messages at 20:25 UTC:** 3831 sent, 0 errors, 0 timeouts
- **DynaKube phase:** `Running`, all 25+ conditions `True`
- **Network policies:** None in `dynatrace` namespace — egress unrestricted

### Namespaces at Time of Investigation

| Namespace | Status | Age |
|---|---|---|
| `backend` | Active | 21h |
| `database` | Active | 21h |
| `default` | Active | 21h |
| `dynatrace` | Active | 67m |
| `frontend` | Active | 21h |
| `kube-node-lease` | Active | 21h |
| `kube-public` | Active | 21h |
| `kube-system` | Active | 21h |

---

## Root Cause

**AKS node drain without graceful PVC detach.**

The node `vmss000003` was scaled down while the ActiveGate StatefulSet had an Azure Disk PVC (`ReadWriteOnce`) attached to it. RWO volumes can only be attached to one node at a time. When the pod was rescheduled onto `vmss000002`, the volume was still locked to the old node, causing a ~5 minute `FailedAttachVolume` window — enough for Dynatrace to flag monitoring as unavailable.

---

## Resolution

No manual action was required. The PVC detached automatically once the old node was fully terminated, the ActiveGate restarted cleanly, and Dynatrace auto-closed P-26061 at 20:35 UTC.

---

## Recommendations

### Prevent Recurrence
1. **Cordon before drain** — always cordon the node first and wait for volume detachment before scaling down:
   ```bash
   kubectl cordon <node>
   kubectl get volumeattachment | grep <pvc-name>  # wait until gone
   az aks scale ...
   ```
2. **Set `terminationGracePeriodSeconds`** on the ActiveGate StatefulSet to allow clean volume unmounting before rescheduling.
3. **PodDisruptionBudget** — configure one for the ActiveGate to sequence drains safely.

### Longer Term
- **Define SLOs** — no SLOs currently configured. Add availability SLOs for `frontend`, `backend`, and `database`.
- **Multi-node topology** — cluster is single-node (`vmss000002`). A node failure would take down all workloads with no redundancy.

---

## Agents Involved

| Agent | Role | Tools Used |
|---|---|---|
| SRE Agent | Detected P-26061, assessed blast radius, confirmed closure | `mcp__dynatrace__query-problems`, `mcp__dynatrace__get-events-for-kubernetes-cluster`, `mcp__dynatrace__execute-dql` |
| DevOps Agent | Checked ActiveGate logs, network policies, DynaKube status, cluster health | `kubectl logs`, `kubectl get events`, `kubectl describe`, `kubectl cluster-info` |
