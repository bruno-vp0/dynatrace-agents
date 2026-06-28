# Security Compliance Report — CIS Benchmark Findings

**Environment:** `https://xjr77121.apps.dynatrace.com`  
**Cluster:** `aks-demo-dt` | Resource Group: `rg-demo-dt`  
**Date:** 2026-06-28  
**Standard:** CIS Kubernetes Benchmark  
**Total findings:** 39 misconfigurations (6 critical, 2 high documented below)  
**Source:** Dynatrace Security Posture Management (`security.events` / `COMPLIANCE_FINDING`)

---

## How These Were Found

These findings were **not surfaced by the initial investigation**. They were discovered only after the user pointed to the Dynatrace Kubernetes Security tab in the UI, which showed "6 critical findings" on the cluster panel.

The initial investigation queried only:
- `dt.davis.problems` — operational Davis problems
- `dt.davis.events` — Kubernetes availability events

It did **not** query `security.events` with `COMPLIANCE_FINDING` type, which is the correct data source for security posture findings. This was a gap in the investigation methodology. See the lessons section at the bottom.

---

## Triggering Prompt

> "are there any critical findings on dynatrace related to the cluster on dynatrace?"

Followed by the user sharing a screenshot of the Dynatrace Kubernetes Security tab showing 6 critical misconfigurations.

---

## Findings Summary

| Rule ID | Title | Severity | Real Violation | Action |
|---|---|---|---|---|
| CIS-76105 | 4.1.2 Minimize access to secrets | CRITICAL | `cluster-admin` only | No action — bootstrap role |
| CIS-83081 | 4.1.7 Limit Bind/Impersonate/Escalate permissions | CRITICAL | `cluster-admin` only | No action — bootstrap role |
| CIS-83083 | 4.1.9 Minimize access to proxy sub-resource of nodes | CRITICAL | `cluster-admin` + `system:prometheus` | Raise with Azure Support |
| CIS-83084 | 4.1.10 Minimize access to CSR approval | CRITICAL | `cluster-admin` only | No action — bootstrap role |
| CIS-83085 | 4.1.11 Minimize access to webhook configuration objects | CRITICAL | `cluster-admin` only | No action — bootstrap role |
| CIS-76107 | 4.1.4 Minimize access to create pods | HIGH | `cluster-admin` only | No action — bootstrap role |
| CIS-76128 | 4.5.1 Prefer secrets as files over env vars | HIGH | `dynatrace/aks-demo-dt-otel-collector-0` only | Fix via DynaKube CR or Dynatrace Support |

---

## Detailed Analysis

### Flagged Roles vs. Actual Violations

The Dynatrace KSPM scanner flagged 5 roles across all RBAC findings. After the DevOps agent inspected the actual ClusterRoles and ClusterRoleBindings:

| Role | Managed by | Real permissions | Verdict |
|---|---|---|---|
| `cluster-admin` | Kubernetes (bootstrap) | `apiGroups: ['*'], resources: ['*'], verbs: ['*']` | Real finding — untouchable |
| `appmonitoringconfig-user` | AKS addon (`addonmanager.kubernetes.io/mode: Reconcile`) | `monitor.azure.com/instrumentations` only | **False positive** |
| `system:metrics-server` | AKS addon | `get, list, watch` on pods/nodes/namespaces only | **False positive** |
| `system:prometheus` | AKS managed (`kubernetes.azure.com/managedby: aks`) | Includes `nodes/proxy` — legitimate for 4.1.9 | Real finding for 4.1.9 only |
| `container-health-log-reader` | AKS addon | Read-only on pods/log, events, deployments | **False positive** |

### 4.1.9 — system:prometheus nodes/proxy

`system:prometheus` explicitly lists `nodes/proxy`, bound to AKS internal users `hcpService` and `konnectivity-prometheus-client`. This is a legitimate CIS violation but AKS-managed — any `kubectl patch` will be reverted by AKS reconciliation.

The role itself contains this annotation comment:
> *"TODO: why do we need this? It seems to be a duplicate or near-duplicate of aks-operator-metrics."*

**Action:** Raise with Azure Support as a managed-cluster compliance gap. Reference CIS 4.1.9.

### 4.5.1 — Secrets as environment variables

App namespaces (`backend`, `database`, `frontend`) are **clean** — zero pods consume secrets via env vars.

Only offender: `dynatrace/aks-demo-dt-otel-collector-0` uses `DT_DATA_INGEST_TOKEN` as an env var from the `aks-demo-dt` secret. This StatefulSet is operator-managed — kubectl patches are reverted immediately.

**Action:** Check `DynaKube` CR for volume-mount option or raise with Dynatrace Support.

---

## Auto-Remediation Assessment

**The DevOps agent cannot auto-remediate any of these findings.** All real violations are Kubernetes/AKS bootstrap infrastructure or operator-managed resources that revert on change.

| Finding | Safe to auto-remediate? | Reason |
|---|---|---|
| 4.1.2 / 4.1.7 / 4.1.10 / 4.1.11 / 4.1.4 | No | Kubernetes bootstrap role (`cluster-admin`) |
| 4.1.9 | No | AKS-managed ClusterRole, will be reverted |
| 4.5.1 | No | Operator-managed StatefulSet, will be reverted |

---

## Required Actions

| Action | Owner | How |
|---|---|---|
| Raise `system:prometheus` nodes/proxy gap | You | Azure Support ticket, reference CIS 4.1.9 |
| Fix `otel-collector` secret env var | Dynatrace Operator | Update `DynaKube` CR or contact Dynatrace Support |
| Accept `cluster-admin` findings as known risk | You | Document as accepted exception in compliance tooling |
| Mark false positives in Dynatrace | You | Dynatrace UI → Security → Compliance → mark as exception |

---

## Lessons Learned — Investigation Gap

### What went wrong

When asked *"are there any critical findings on dynatrace related to the cluster?"*, the initial response only checked operational data sources (`dt.davis.problems`, `dt.davis.events`). Security posture findings in `security.events` (`COMPLIANCE_FINDING`) were not queried.

The user had to point to the Dynatrace UI screenshot before the compliance layer was checked. This means **real critical findings went unreported** in the first pass.

### Root cause

The SRE agent SKILL.md only covered operational tools (problems, events, traces). It had no instruction to check compliance/security posture findings as part of a health or security investigation.

### Fix applied

The SRE agent SKILL.md has been updated to include `mcp__dynatrace__get-dynatrace-compliance-findings` and `mcp__dynatrace__get-security-events-summary` as required checks whenever the user asks about security, critical findings, or overall cluster health.

---

## Agents Involved

| Agent | Role | Gap |
|---|---|---|
| SRE Agent | Initial problem/event check | Missed compliance layer entirely |
| DevOps Agent | RBAC audit via kubectl, secret env var scan, per-finding remediation assessment | None — performed correctly once directed |
