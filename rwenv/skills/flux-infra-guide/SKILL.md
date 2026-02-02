---
name: flux-infra-guide
description: Infrastructure reference for debugging and operations in the Flux-managed environment
triggers:
  - infrastructure guide
  - where does X run
  - what namespace
  - how to debug
  - check logs
  - flux resources
  - service unhealthy
  - pod failing
---

# Flux Infrastructure Guide

Quick reference for navigating the RunWhen infrastructure managed by Flux CD.

## When to Use This Skill

- Debugging service issues (unhealthy pods, failed migrations)
- Finding where a service/config is defined
- Understanding how secrets and config flow into services
- Querying logs and metrics for a specific service
- Tracing Flux reconciliation failures

## Prerequisites

- rwenv must be set (`/rwenv-cur` to verify)
- For Flux CLI commands: dev container running
- For log queries: access to observability stack

## Loading Data Files

This skill uses lazy loading. Read data files only when needed:

| Question Type | Data File |
|---------------|-----------|
| Service location ("where does X run") | `data/services.json` |
| Flux status ("why isn't X syncing") | `data/flux-resources.json` |
| Secrets ("where does X get credentials") | `data/secrets-map.json` |
| Config ("what value does X have") | `data/configmaps.json` |

Data files are located in this skill's `data/` directory.

---

## Service Map

Quick lookup for where services run.

### How to Use

For service location questions ("where does X run", "what namespace is Y"):
1. Read `data/services.json` from this skill's directory
2. Find the service entry
3. Use namespace/deployment info for kubectl commands

### Data Format (services.json)

```json
{
  "services": {
    "papi": {
      "namespace": "backend-services",
      "deployment": "papi",
      "fluxPath": "apps/backend-services/",
      "kustomization": "runwhen-backend-services",
      "description": "Platform API - core backend service"
    }
  },
  "metadata": {
    "generatedFrom": "infra-flux-nonprod-test",
    "generatedAt": "2026-02-02T10:00:00Z"
  }
}
```

### Common Namespaces

| Namespace | Purpose |
|-----------|---------|
| `backend-services` | Core platform services (PAPI, Celery, etc.) |
| `flux-system` | Flux controllers and sources |
| `linkerd` | Service mesh control plane |
| `loki` | Log aggregation |
| `mimir` | Metrics storage |
| `vault` | Secrets management |
| `cortex` | Application metrics |
| `llm-gateway` | LLM proxy services |
| `runner-system` | Runner infrastructure |
| `corestate` | State management |
| `gitservice` | Git integration |
