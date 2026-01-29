# Conditional Transformation Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Support both dev container and local tool execution modes with a single plugin.

**Architecture:** Add `useDevContainer` global setting (default: true). Hook checks this flag and either routes through docker exec or runs locally with just `--context` flag. New skill to toggle mode.

**Tech Stack:** Bash, jq, Claude Code hooks

---

## Task 1: Add Helper Function to rwenv-utils.sh

**Files:**
- Modify: `lib/rwenv-utils.sh`

**Step 1: Add get_use_dev_container function**

Add after line 23 (after `get_dev_container` function):

```bash
# Get useDevContainer setting (defaults to true)
get_use_dev_container() {
    local config_dir
    config_dir="$(get_config_dir)"
    local envs_file="$config_dir/envs.json"

    if [[ ! -f "$envs_file" ]]; then
        echo "true"  # Default to dev container
        return
    fi

    jq -r '.useDevContainer // true' "$envs_file"
}
```

**Step 2: Test the function**

Run:
```bash
source lib/rwenv-utils.sh && get_use_dev_container
```
Expected: `true`

**Step 3: Commit**

```bash
git add lib/rwenv-utils.sh
git commit -m "feat: add get_use_dev_container helper function"
```

---

## Task 2: Update transform-commands.sh for Conditional Execution

**Files:**
- Modify: `hooks/transform-commands.sh`

**Step 1: Add useDevContainer check after loading rwenv config**

Replace lines 129-136 (the dev container running check) with:

```bash
# Get execution mode
USE_DEV_CONTAINER=$(get_use_dev_container)
DEV_CONTAINER=$(get_dev_container)

# Handle dev container mode
if [[ "$USE_DEV_CONTAINER" == "true" ]]; then
    # Check if dev container is running
    if ! docker ps --format '{{.Names}}' 2>/dev/null | grep -q "^${DEV_CONTAINER}$"; then
        cat >&2 <<EOF
Dev container '$DEV_CONTAINER' is not running.

Options:
1. Start the container and retry
2. Switch to local mode (requires kubectl/helm/flux installed locally)

To switch to local mode, run: /rwenv-local-mode
EOF
        exit 2
    fi
fi
```

**Step 2: Update build_transformed_command function**

Replace the `build_transformed_command` function (lines 210-255) with:

```bash
# Build the transformed command based on the base command
build_transformed_command() {
    local cmd_prefix=""

    if [[ "$USE_DEV_CONTAINER" == "true" ]]; then
        # Dev container mode: use docker exec with full kubeconfig path
        cmd_prefix="docker exec -i $DEV_CONTAINER"
        local kubeconfig_flag="--kubeconfig=$KUBECONFIG_PATH"
    else
        # Local mode: no docker, no kubeconfig (use kubectl defaults)
        cmd_prefix=""
        local kubeconfig_flag=""
    fi

    case "$BASE_CMD" in
        kubectl)
            check_write_operation "$BASE_CMD" "$CMD_ARGS" "kubectl"
            if [[ -n "$cmd_prefix" ]]; then
                echo "$cmd_prefix kubectl $kubeconfig_flag --context=$K8S_CONTEXT $CMD_ARGS"
            else
                echo "kubectl --context=$K8S_CONTEXT $CMD_ARGS"
            fi
            ;;
        helm)
            check_write_operation "$BASE_CMD" "$CMD_ARGS" "helm"
            if [[ -n "$cmd_prefix" ]]; then
                echo "$cmd_prefix helm $kubeconfig_flag --kube-context=$K8S_CONTEXT $CMD_ARGS"
            else
                echo "helm --kube-context=$K8S_CONTEXT $CMD_ARGS"
            fi
            ;;
        flux)
            check_write_operation "$BASE_CMD" "$CMD_ARGS" "flux"
            if [[ -n "$cmd_prefix" ]]; then
                echo "$cmd_prefix flux $kubeconfig_flag --context=$K8S_CONTEXT $CMD_ARGS"
            else
                echo "flux --context=$K8S_CONTEXT $CMD_ARGS"
            fi
            ;;
        gcloud)
            check_gcloud_for_k3s
            # gcloud is ALWAYS read-only
            if is_gcloud_write_operation "$CMD_ARGS"; then
                cat >&2 <<EOF
ERROR: gcloud write operations are blocked for safety.

Blocked command: gcloud $CMD_ARGS

gcloud is always read-only regardless of rwenv settings.
Blocked operations include: create, delete, start, stop, reset, resize, patch, update, rm, cp, mv

Use the GCP Console or a dedicated deployment pipeline for write operations.
EOF
                exit 2
            fi
            if [[ -n "$cmd_prefix" ]]; then
                echo "$cmd_prefix gcloud --project=$GCP_PROJECT $CMD_ARGS"
            else
                echo "gcloud --project=$GCP_PROJECT $CMD_ARGS"
            fi
            ;;
        vault)
            if [[ -n "$cmd_prefix" ]]; then
                echo "$cmd_prefix vault $CMD_ARGS"
            else
                echo "vault $CMD_ARGS"
            fi
            ;;
        *)
            echo "$ORIGINAL_CMD"
            ;;
    esac
}
```

**Step 3: Test dev container mode**

Run:
```bash
echo '{"tool_name":"Bash","tool_input":{"command":"kubectl get pods"}}' | hooks/transform-commands.sh | jq -r '.tool_input.command'
```
Expected: `docker exec -i alpine-dev-container-zsh-rdebug kubectl --kubeconfig=... --context=rdebug-61 get pods`

**Step 4: Commit**

```bash
git add hooks/transform-commands.sh
git commit -m "feat: add conditional transformation for dev container vs local mode"
```

---

## Task 3: Create rwenv-local-mode Skill

**Files:**
- Create: `skills/rwenv-local-mode/SKILL.md`

**Step 1: Create skill directory**

```bash
mkdir -p skills/rwenv-local-mode
```

**Step 2: Write skill file**

```markdown
---
name: rwenv-local-mode
description: Toggle between dev container and local execution mode
triggers:
  - /rwenv-local-mode
  - switch to local mode
  - switch to dev container
  - toggle execution mode
---

# Toggle Execution Mode

Switch between dev container and local tool execution modes.

## Instructions

1. **Read current setting** from `~/.claude/rwenv/envs.json`:
   - Check `useDevContainer` field (defaults to `true` if not present)

2. **Toggle the setting**:
   - If `true` → set to `false`
   - If `false` → set to `true`

3. **Validate before switching**:

   **When switching TO local mode (`useDevContainer: false`):**
   ```bash
   which kubectl || echo "WARNING: kubectl not found in PATH"
   which helm || echo "WARNING: helm not found in PATH"
   which flux || echo "WARNING: flux not found in PATH"
   ```
   Show warnings but proceed anyway.

   **When switching TO dev container mode (`useDevContainer: true`):**
   ```bash
   docker ps --format '{{.Names}}' | grep -q "<devContainer>"
   ```
   Warn if container not running but proceed anyway.

4. **Update the config file**:
   ```bash
   # Read, modify, write back
   jq '.useDevContainer = false' ~/.claude/rwenv/envs.json > /tmp/envs.json
   mv /tmp/envs.json ~/.claude/rwenv/envs.json
   ```

5. **Confirm the change**:

   If switched to local mode:
   ```
   Switched to LOCAL mode.

   Commands will run directly on your machine using:
     kubectl --context=<context> ...
     helm --kube-context=<context> ...
     flux --context=<context> ...

   Requirements:
   - kubectl, helm, flux must be installed locally
   - Kubeconfig must be configured with the correct context

   To switch back: /rwenv-local-mode
   ```

   If switched to dev container mode:
   ```
   Switched to DEV CONTAINER mode.

   Commands will run through: <devContainer>
     docker exec -i <container> kubectl --kubeconfig=... --context=... ...

   To switch back: /rwenv-local-mode
   ```

## Error Handling

| Error | Response |
|-------|----------|
| envs.json not found | Create it with `{"version":"1.0","useDevContainer":false}` |
| JSON parse error | Report error, don't modify file |
| Write permission denied | Report error with suggestion to check permissions |
```

**Step 3: Commit**

```bash
git add skills/rwenv-local-mode/
git commit -m "feat: add rwenv-local-mode skill to toggle execution mode"
```

---

## Task 4: Update rwenv-cur Skill to Show Execution Mode

**Files:**
- Modify: `skills/rwenv-cur/SKILL.md`

**Step 1: Update the skill to include Exec Mode**

Add to the output format section (after Read-Only line):

In the example output block around line 35, add after `Read-Only:   No`:

```
Exec Mode:   Dev Container (alpine-dev-container-zsh-rdebug)
```

And add instruction to read `useDevContainer` setting:

After step 4, add:

```markdown
5. **Show execution mode**:
   - Read `useDevContainer` from `envs.json` (defaults to `true`)
   - If `true`: `Exec Mode:   Dev Container (<devContainer name>)`
   - If `false`: `Exec Mode:   Local (tools from PATH)`
```

**Step 2: Commit**

```bash
git add skills/rwenv-cur/SKILL.md
git commit -m "feat: show execution mode in rwenv-cur output"
```

---

## Task 5: Update Example Config

**Files:**
- Modify: `config/envs.example.json`

**Step 1: Add useDevContainer field**

Update the file to:

```json
{
  "version": "1.0",
  "useDevContainer": true,
  "devContainer": "alpine-dev-container-zsh-rdebug",
  "rwenvs": {
    "rdebug": {
      "description": "VM based dev setup (k3s)",
      "type": "k3s",
      "kubeconfigPath": "/root/.kube/config",
      "kubernetesContext": "rdebug-61",
      "readOnly": false,
      "fluxGitRepo": "https://gitea.rdebug-61.local.runwhen.com/platform-setup/runwhen-platform-self-hosted-local-dev"
    },
    "gke-prod": {
      "description": "GKE production cluster",
      "type": "gke",
      "kubeconfigPath": "/root/.kube/gke-prod.config",
      "kubernetesContext": "gke_project_region_cluster",
      "gcpProject": "my-gcp-project",
      "readOnly": true,
      "fluxGitRepo": "https://github.com/org/flux-repo"
    }
  }
}
```

**Step 2: Commit**

```bash
git add config/envs.example.json
git commit -m "docs: add useDevContainer to example config"
```

---

## Task 6: Test End-to-End

**Step 1: Test dev container mode (default)**

```bash
# Verify setting
source lib/rwenv-utils.sh && get_use_dev_container
# Expected: true

# Test transformation
echo '{"tool_name":"Bash","tool_input":{"command":"kubectl get pods"}}' | hooks/transform-commands.sh | jq -r '.tool_input.command'
# Expected: docker exec -i ... kubectl --kubeconfig=... --context=... get pods
```

**Step 2: Test local mode**

```bash
# Temporarily set local mode
jq '.useDevContainer = false' ~/.claude/rwenv/envs.json > /tmp/envs.json && mv /tmp/envs.json ~/.claude/rwenv/envs.json

# Test transformation
echo '{"tool_name":"Bash","tool_input":{"command":"kubectl get pods"}}' | hooks/transform-commands.sh | jq -r '.tool_input.command'
# Expected: kubectl --context=rdebug-61 get pods

# Restore dev container mode
jq '.useDevContainer = true' ~/.claude/rwenv/envs.json > /tmp/envs.json && mv /tmp/envs.json ~/.claude/rwenv/envs.json
```

**Step 3: Final commit**

```bash
git add -A
git commit -m "test: verify conditional transformation works"
```

---

## Task 7: Sync and Create PR

**Step 1: Sync to plugin cache**

```bash
rsync -av --delete --exclude='.git' --exclude='docs/plans' . ~/.claude/plugins/cache/rwenv/rwenv/0.1.0/
```

**Step 2: Push and create PR**

```bash
git push -u origin feat/conditional-transformation
gh pr create --title "feat: conditional transformation for dev container vs local mode" --body "..."
```
