# P-26062 — OOMKill: oom-pod (backend namespace)

**Environment:** `https://xjr77121.apps.dynatrace.com`  
**Cluster:** `aks-demo-dt` | Resource Group: `rg-demo-dt`  
**Problem ID:** P-26062  
**Severity:** ERROR  
**Status:** RESOLVED  
**Started:** 2026-06-28T21:02Z  
**Resolved:** 2026-06-28T~21:15Z  
**Duration:** ~13 minutes  

---

## Triggering Prompt

> "SRE, what's going on in the cluster?"

---

## Context

This was a deliberate simulation to test the SRE → DevOps agent handoff flow. A stress pod was deployed to the `backend` namespace with a memory limit of 64Mi but configured to allocate 128M — a 2x overcommit guaranteed to trigger OOMKill.

**Deploy command used:**
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: oom-pod
  namespace: backend
spec:
  restartPolicy: Always
  containers:
  - name: stress
    image: polinux/stress
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "128M"]
    resources:
      limits:
        memory: 64Mi
EOF
```

---

## Timeline

| Time (UTC) | Event |
|---|---|
| ~21:00Z | `oom-pod` deployed to `backend` namespace |
| 21:01Z | First OOMKill — kernel kills `stress` process, container exits 137 |
| 21:02Z | Davis raises P-26062 (ERROR) |
| 21:01–21:04Z | 12 kernel OOM kill events, 5 container OOMKills, 4 BackOff restarts |
| ~21:05Z | SRE agent triaged — found P-26062, escalated to DevOps with full context |
| ~21:10Z | DevOps agent confirmed CrashLoopBackOff (6 restarts), deleted pod |
| ~21:10Z | Node conditions verified healthy (MemoryPressure: False, Ready: True) |
| ~21:15Z | P-26062 auto-closed by Davis after OOMKill cycle stopped |

---

## SRE Findings

- **Active problem:** P-26062 — Out-of-memory kills (ERROR, ACTIVE)
- **Root cause entity:** `CLOUD_APPLICATION-E9311FD2A54538EC` (`oom-pod`, namespace `backend`)
- **Affected node:** `aks-nodepool1-12814225-vmss000002`
- **Blast radius:** 1 workload, no user-facing services affected
- **Event breakdown (30 min window):**
  - 12 kernel OOM kill events on node
  - 5 container OOMKilled events
  - 4 BackOff restart events
- **Escalation:** Handed off to DevOps agent with full entity context

---

## DevOps Remediation

**Confirmed root cause:**
- Container `stress` requested `--vm-bytes 128M` against a hard limit of `64Mi` (2x overcommit)
- Exit code `137` (SIGKILL from OOM killer) confirmed in pod description
- Pod in `CrashLoopBackOff` with 6 restarts at time of intervention
- Dynatrace OneAgent init container had already injected enrichment — problem tracked correctly

**Actions taken:**
1. `kubectl get pod oom-pod -n backend` — confirmed `CrashLoopBackOff`, 6 restarts
2. `kubectl describe pod oom-pod -n backend` — confirmed OOMKill cause
3. `kubectl delete pod oom-pod -n backend` — deleted successfully
4. `kubectl get pod oom-pod -n backend` — `NotFound`, confirmed gone
5. Node conditions checked — all healthy post-deletion

**Node state after remediation:**

| Condition | Status |
|---|---|
| MemoryPressure | False |
| DiskPressure | False |
| PIDPressure | False |
| Ready | True |

---

## Agent Handoff Flow

```
User prompt
    └─→ SRE Agent
            ├─ mcp__dynatrace__query-problems                    → P-26062 found (ACTIVE)
            ├─ mcp__dynatrace__get-events-for-kubernetes-cluster → OOMKill events confirmed
            └─ Escalation to DevOps Agent
                    ├─ kubectl get pod oom-pod -n backend
                    ├─ kubectl describe pod oom-pod -n backend
                    ├─ kubectl delete pod oom-pod -n backend
                    ├─ kubectl get pod oom-pod -n backend        (verify gone)
                    └─ kubectl describe node                     (verify healthy)
```

---

## Outcome

- Pod deleted, OOMKill cycle stopped
- Node healthy with no residual memory pressure
- P-26062 auto-closed by Davis within ~5 minutes of remediation
- SRE → DevOps handoff worked end-to-end with no manual intervention required
