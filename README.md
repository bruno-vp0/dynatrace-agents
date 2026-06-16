# Dynatrace Agents

Claude Code agent orchestrator for Dynatrace — triage incidents, analyse infrastructure, and build dashboards using AI agents and DQL.

## Agents

| Agent | Purpose |
|-------|---------|
| **SRE** | Incident triage, root cause analysis, SLO monitoring, distributed tracing |
| **DevOps** | Host/K8s/cloud resource health, deployments, capacity planning |
| **Dashboard** | Create and update Dynatrace dashboards and notebooks |

## Requirements

- [dtctl](https://github.com/Dynatrace/dynatrace-for-ai) — Dynatrace CLI for AI agents
- Claude Code with the Dynatrace skills plugin

## Skills

Built on top of the official Dynatrace skills for Claude Code:
[https://github.com/Dynatrace/dynatrace-for-ai](https://github.com/Dynatrace/dynatrace-for-ai)

## Example Prompt

```
Act as the SRE agent and analyse problem P-26061.
```

See each agent's `README.md` for agent-specific prompt examples.

## Setup

```bash
dtctl ctx add my-env \
  --url https://<your-env>.apps.dynatrace.com \
  --token <your-token>

dtctl ctx use my-env
```
