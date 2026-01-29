# Conditional Command Transformation Design

**Date:** 2026-01-29
**Status:** Approved

## Problem

Team members who don't use a dev container have kubectl/helm/flux installed locally. The current plugin always routes commands through the dev container, which doesn't work for these users.

## Solution

Add conditional transformation that supports both execution modes with a single plugin.

## Configuration

Add a global `useDevContainer` setting to `~/.claude/rwenv/envs.json`:

```json
{
  "version": "1.0",
  "useDevContainer": true,
  "devContainer": "alpine-dev-container-zsh-rdebug",
  "rwenvs": { ... }
}
```

**Behavior:**
- `true` (default): Commands run through dev container
- `false`: Commands run locally with just `--context` flag

## Transformation Examples

| Mode | Input | Output |
|------|-------|--------|
| Dev container | `kubectl get pods` | `docker exec -i <container> kubectl --kubeconfig=... --context=rdebug-61 get pods` |
| Local | `kubectl get pods` | `kubectl --context=rdebug-61 get pods` |

## Hook Logic

Update `transform-commands.sh`:

```bash
# Load setting (default: true)
USE_DEV_CONTAINER=$(load_envs | jq -r '.useDevContainer // true')

if [[ "$USE_DEV_CONTAINER" == "true" ]]; then
    # Check if dev container is running
    if ! docker ps --format '{{.Names}}' | grep -q "^${DEV_CONTAINER}$"; then
        # Container not found - offer to switch
        cat >&2 <<EOF
Dev container '$DEV_CONTAINER' is not running.

Options:
1. Start the container and retry
2. Switch to local mode (requires kubectl/helm/flux installed locally)

To switch to local mode, run: /rwenv-local-mode
EOF
        exit 2
    fi

    # Dev container mode
    echo "docker exec -i $DEV_CONTAINER kubectl --context=$K8S_CONTEXT $CMD_ARGS"
else
    # Local mode - just add context flag
    echo "kubectl --context=$K8S_CONTEXT $CMD_ARGS"
fi
```

**Key points:**
- Setting defaults to `true` if not present (backward compatible)
- Clear error message with options when container not found
- Local mode only adds `--context`, no `--kubeconfig`

## Mode Switching Skill

New skill `/rwenv-local-mode` to toggle the setting:

**Behavior:**
1. Read `~/.claude/rwenv/envs.json`
2. Toggle `useDevContainer`:
   - If true → set to false, confirm: "Switched to local mode"
   - If false → set to true, confirm: "Switched to dev container mode"
3. Write updated config

**Validation:**
- When switching TO dev container mode: warn if container not running
- When switching TO local mode: warn if kubectl not found locally

## Visibility

Update `/rwenv-cur` to show execution mode:

```
Current rwenv: rdebug

Type:        k3s
Description: VM based dev setup (k3s)
Context:     rdebug-61
Read-Only:   No
Exec Mode:   Dev Container (alpine-dev-container-zsh-rdebug)
```

Or for local mode:
```
Exec Mode:   Local (kubectl/helm/flux from PATH)
```

## Files to Change

| File | Change |
|------|--------|
| `hooks/transform-commands.sh` | Add conditional transformation based on `useDevContainer` |
| `lib/rwenv-utils.sh` | Add `get_use_dev_container()` helper |
| `skills/rwenv-local-mode/SKILL.md` | New skill to toggle mode |
| `skills/rwenv-cur/SKILL.md` | Show exec mode in output |
| `config/envs.example.json` | Add `useDevContainer: true` example |

**No changes needed to:**
- Subagents (they use the same transformed commands)
- Other skills (rwenv-list, rwenv-set, rollout)
- Git validation hook
