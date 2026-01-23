# rwenv Plugin Design

**Date:** 2026-01-24
**Status:** Approved
**Author:** Claude + Rohit

## Overview

A Claude Code plugin for managing multi-cluster Kubernetes environments (rwenv = RunWhen environment). Enables safe interaction with GKE and k3s clusters through a dev container, with automatic context injection and safety enforcement.

## Goals

1. **Environment switching** - Easily switch between GKE/k3s environments per working directory
2. **Command safety** - Ensure commands run through dev container with explicit `--context`/`--project` flags
3. **Write protection** - Enforce read-only mode for sensitive environments, always read-only for gcloud/database
4. **Git safety** - Protect main branch in current project, allow main in rwenv repos

---

## Section 1: Core Data Model

### rwenv Definition (`~/.claude/rwenv/envs.json`)

```json
{
  "version": "1.0",
  "devContainer": "alpine-dev-container-zsh-rdebug",
  "databases": {
    "core": {
      "namespace": "backend-services",
      "secretName": "core-pguser-core",
      "pgbouncerHost": "core-pgbouncer.backend-services.svc.cluster.local",
      "database": "core",
      "username": "core"
    },
    "usearch": {
      "namespace": "backend-services",
      "secretName": "core-pguser-usearch",
      "pgbouncerHost": "core-pgbouncer.backend-services.svc.cluster.local",
      "database": "usearch",
      "username": "usearch"
    },
    "agentfarm": {
      "namespace": "databases",
      "secretName": "postgres-pguser-agentfarm",
      "pgbouncerHost": "core-pgbouncer.backend-services.svc.cluster.local",
      "database": "app_users",
      "username": "agentfarm"
    }
  },
  "rwenvs": {
    "rdebug": {
      "description": "VM based dev setup (k3s)",
      "type": "k3s",
      "kubeconfigPath": "/root/.kube/config",
      "kubernetesContext": "rdebug-61",
      "readOnly": false,
      "fluxGitRepo": "https://gitea.rdebug-61.local.runwhen.com/platform-setup/runwhen-platform-self-hosted-local-dev",
      "services": {
        "papi": "https://papi.rdebug-61.local.runwhen.com",
        "app": "https://app.rdebug-61.local.runwhen.com",
        "vault": "https://vault.rdebug-61.local.runwhen.com",
        "gitea": "https://gitea.rdebug-61.local.runwhen.com",
        "minio": "https://minio-console.rdebug-61.local.runwhen.com",
        "agentfarm": "https://agentfarm.rdebug-61.local.runwhen.com"
      }
    },
    "gke-prod": {
      "description": "GKE production cluster",
      "type": "gke",
      "kubeconfigPath": "/root/.kube/gke-prod.config",
      "kubernetesContext": "gke_project_region_cluster",
      "gcpProject": "my-gcp-project",
      "readOnly": true,
      "fluxGitRepo": "https://github.com/org/flux-repo",
      "services": {
        "papi": "https://papi.prod.example.com"
      }
    }
  }
}
```

### rwenv Selection (`~/.claude/rwenv/env-consumers.json`)

Maps working directory to active rwenv:

```json
{
  "/Users/rohitekbote/wd/project-a": "rdebug",
  "/Users/rohitekbote/wd/project-b/.worktrees/feature-x": "gke-prod"
}
```

### Key Design Decisions

- **Databases are shared** - Database config is at top level, not per-rwenv (same DBs across environments)
- **Config outside plugin** - `~/.claude/rwenv/` is separate from plugin for multi-team support
- **Configurable path** - Config location can be overridden via `RWENV_CONFIG_DIR` environment variable

---

## Section 2: Plugin Structure

**Location:** Local development in `/Users/rohitekbote/wd/rwenv-plugin/`, later published to git

```
rwenv-plugin/
├── manifest.json
├── README.md
├── docs/
│   ├── INSTALLATION.md
│   ├── USAGE.md
│   └── CONFIGURATION.md
├── config/
│   └── envs.example.json
├── skills/
│   ├── rwenv-list.md
│   ├── rwenv-cur.md
│   ├── rwenv-set.md
│   └── rwenv-add.md
├── hooks/
│   ├── pre-command.sh
│   └── validate-git.sh
├── subagents/
│   ├── k8s-ops.md
│   ├── db-ops.md
│   └── gcloud-ops.md
├── scripts/
│   ├── pg_query.sh
│   └── command-builder.sh
└── lib/
    └── rwenv-utils.sh
```

### manifest.json

```json
{
  "name": "rwenv",
  "version": "0.1.0",
  "description": "RunWhen environment management for multi-cluster k8s operations",
  "config": {
    "configDir": {
      "default": "~/.claude/rwenv",
      "env": "RWENV_CONFIG_DIR",
      "description": "Directory containing envs.json and env-consumers.json"
    }
  },
  "skills": ["skills/*.md"],
  "hooks": {
    "pre-command": "hooks/pre-command.sh"
  },
  "subagents": ["subagents/*.md"]
}
```

---

## Section 3: Skills

### `/rwenv-list` - List all rwenvs

- Reads `envs.json`
- Shows table: name, type (k3s/gke), description, readOnly status
- Marks currently active rwenv with indicator

### `/rwenv-set <name>` - Select rwenv

- If no name provided, show list and ask
- If rwenv already set and different from requested:
  - Show current vs requested comparison
  - **Ask for confirmation before changing**
- Update `env-consumers.json` with cwd → name mapping
- Display summary of activated rwenv

### `/rwenv-cur` - Show current rwenv

- Read current cwd's rwenv from `env-consumers.json`
- If not set: show "No rwenv set" and suggest `/rwenv-set`
- If set: display full details (context, project, services, readOnly)

### `/rwenv-add` - Create new rwenv interactively

Interactive prompts for:
1. Name
2. Type (gke/k3s)
3. GCP project (if gke)
4. Kubeconfig path
5. Kubernetes context
6. FluxCD git repo URL
7. Read-only mode
8. Service URLs (optional, repeating)
9. Option to set as active for current directory

### Natural Language Triggers

- "list environments" / "show rwenvs" → `/rwenv-list`
- "switch to rdebug" / "use gke-prod" → `/rwenv-set`
- "what environment am I using" / "current rwenv" → `/rwenv-cur`
- "add new environment" / "create rwenv" → `/rwenv-add`

---

## Section 4: Hooks

### pre-command.sh - Command Transformation & Safety

**1. Command Detection:**
Triggers on: `kubectl`, `helm`, `flux`, `gcloud`, `vault`

**2. rwenv Check:**
- No rwenv set → block command, list rwenvs, ask user to select
- rwenv set → proceed with transformation

**3. Command Transformation:**

```bash
# Original
kubectl get pods -n production

# Transformed
docker exec -it alpine-dev-container-zsh-rdebug \
  kubectl --kubeconfig=/root/.kube/config \
          --context=rdebug-61 \
          get pods -n production
```

For GKE with gcloud:
```bash
# Original
gcloud compute instances list

# Transformed
docker exec -it alpine-dev-container-zsh-rdebug \
  gcloud --project=my-gcp-project \
  compute instances list
```

**4. readOnly Enforcement:**

If `readOnly: true`, block:
- `kubectl apply|delete|patch|create|edit|replace|scale`
- `helm install|upgrade|uninstall|rollback`
- `flux reconcile|suspend|resume`

Allow: `get`, `describe`, `logs`, `top`, `exec`

### validate-git.sh - Git Branch Protection

**Current project (cwd):**
- Block: `git push origin main`, `git commit` while on main, `git merge ... main`

**rwenv repos (not cwd):**
- Allow: all git operations including main branch

---

## Section 5: Subagents

### k8s-ops - Kubernetes Operations

Handles common workflows with rwenv context automatically:

| Capability | Examples |
|------------|----------|
| Pod operations | restart deployment, get logs, describe pod |
| Flux operations | reconcile repo, check status, show pending |
| Debugging | why is X failing, exec into pod |

**Behaviors:**
- Uses active rwenv's context/project/kubeconfig
- Respects readOnly flag (warns and blocks mutations)
- Uses service FQDNs from rwenv config

### db-ops - PostgreSQL Queries

**Always read-only** regardless of rwenv.

| Capability | Examples |
|------------|----------|
| Query execution | "query core db", custom SQL |
| Schema inspection | list tables, describe table |
| Quick queries | count rows, sample data |

**Uses:**
- Context/kubeconfig from active rwenv
- Database config from shared `databases` section
- Credentials fetched from K8s secrets (never stored)

**Blocked operations:**
- INSERT, UPDATE, DELETE, DROP, TRUNCATE, ALTER, CREATE, GRANT, REVOKE

### gcloud-ops - GCP Operations

**Always read-only** regardless of rwenv.

**Blocked:**
```
gcloud compute instances create|delete|start|stop|reset
gcloud container clusters create|delete|resize
gcloud sql instances create|delete|patch
gcloud storage rm|cp|mv (write targets)
gcloud iam service-accounts create|delete
gcloud projects delete
gcloud deployment-manager deployments create|delete|update
```

**Allowed:**
```
gcloud compute instances list|describe
gcloud container clusters list|describe|get-credentials
gcloud sql instances list|describe
gcloud storage ls|cat
gcloud iam service-accounts list|describe
gcloud projects list|describe
gcloud logging read
gcloud monitoring metrics list
```

**Note:** For k3s-type rwenvs (no `gcpProject`), gcloud commands blocked with: "gcloud not available for k3s rwenv"

---

## Section 6: Command Flow & Error Handling

### Complete Flow Example

```
User: "get pods in production namespace"

1. Hook detects: kubectl command
2. Check rwenv for cwd:
   - Not set? → Show rwenv list, ask user to select, stop
   - Set? → Continue
3. Load rwenv config (rdebug):
   - kubernetesContext: rdebug-61
   - kubeconfigPath: /root/.kube/config
   - readOnly: false
4. Build command:
   docker exec -it alpine-dev-container-zsh-rdebug \
     kubectl --kubeconfig=/root/.kube/config \
             --context=rdebug-61 \
             get pods -n production
5. Execute and return output
```

### Error Scenarios

| Scenario | Behavior |
|----------|----------|
| No rwenv set | List rwenvs, ask user to select |
| Dev container not running | Error: "Dev container 'X' not running. Start it first." |
| readOnly + mutation attempt | Block: "rwenv 'gke-prod' is read-only. Cannot execute: kubectl delete..." |
| Invalid rwenv name | Error: "rwenv 'foo' not found. Available: rdebug, gke-prod" |
| Git push to main (current project) | Block: "Cannot push to main branch in current project." |
| Missing kubeconfig | Error: "Kubeconfig not found at X in dev container" |
| gcloud on k3s rwenv | Block: "gcloud not available for k3s rwenv 'rdebug'" |

### Confirmation Prompts

| Action | Prompt |
|--------|--------|
| Change rwenv | "Switch from 'rdebug' to 'gke-prod'? (y/n)" |
| Destructive command (non-readOnly) | "About to delete deployment 'papi'. Confirm? (y/n)" |

---

## Section 7: Safety Matrix

| Component | readOnly rwenv | non-readOnly rwenv |
|-----------|---------------|-------------------|
| kubectl read | Allowed | Allowed |
| kubectl write | Blocked | Allowed (confirm) |
| gcloud read | Allowed | Allowed |
| gcloud write | Blocked | Blocked |
| db queries | Read-only | Read-only |
| git (current project) | No main | No main |
| git (rwenv repos) | Main OK | Main OK |

---

## Section 8: Multi-Team Support

**Config outside plugin** enables:

| Scenario | Config location |
|----------|----------------|
| Individual developer | `~/.claude/rwenv/` (default) |
| Team shared config | `/shared/team-a/rwenv/` via `RWENV_CONFIG_DIR` |
| Per-project config | `./project/.rwenv/` via env var |

---

## Section 9: Implementation Plan

### Phase 1: Core Foundation
1. Create plugin directory structure
2. Create `manifest.json` with plugin metadata and config
3. Create `lib/rwenv-utils.sh` - shared functions
4. Create `config/envs.example.json` - example config template

### Phase 2: Skills
1. `skills/rwenv-list.md`
2. `skills/rwenv-cur.md`
3. `skills/rwenv-set.md`
4. `skills/rwenv-add.md`

### Phase 3: Hooks
1. `hooks/pre-command.sh` - command transformation & safety
2. `hooks/validate-git.sh` - main branch protection

### Phase 4: Subagents
1. `subagents/k8s-ops.md`
2. `subagents/db-ops.md`
3. `subagents/gcloud-ops.md`

### Phase 5: Scripts
1. `scripts/pg_query.sh` - database query execution
2. `scripts/command-builder.sh` - docker exec wrapper

### Phase 6: Documentation
1. `README.md` - overview and features
2. `docs/INSTALLATION.md` - step-by-step installation guide
3. `docs/USAGE.md` - how to use each skill/command
4. `docs/CONFIGURATION.md` - config file reference

---

## Future Enhancements

- `/rwenv-edit <name>` - Modify existing rwenv
- `/rwenv-remove <name>` - Delete rwenv (with confirmation)
- `/rwenv-export` - Export config for sharing with team
- `/rwenv-import <file>` - Import rwenv definitions
- `vault-ops` subagent - Secrets management
- `monitoring-ops` subagent - Logs aggregation, metrics queries
