---
name: rollout
description: Rollout latest image tag from feature branch to flux repo
triggers:
  - /rollout
  - rollout
  - deploy tag
  - update tag
  - push tag
---

# Rollout Image Tag

Update the image tag in the flux repo for a service using the current branch's PR number and commit SHA.

## Tag Format

**CRITICAL**: The image tag format is exactly:
```
ps-<pr_number>-<commit_sha_first_7_chars>
```

Example: `ps-123-a1b2c3d`

**Rules:**
- PR number: Get from `gh pr view --json number`
- Commit SHA: **Exactly 7 characters** - use `git rev-parse --short=7 HEAD`
- **DO NOT** check if tag exists in registry or git
- **DO NOT** verify if build pipeline passed (unless explicitly asked)
- **DO NOT** use 8 characters or full SHA

## Instructions

### Step 1: Verify prerequisites

1. Check rwenv is set for current directory (use `/rwenv-cur`)
2. Get the flux repo URL from rwenv config

### Step 2: Construct the image tag

```bash
# Get PR number for current branch
PR_NUM=$(gh pr view --json number --jq '.number')

# Get first 7 chars of HEAD commit (EXACTLY 7, not 8)
COMMIT_SHA=$(git rev-parse --short=7 HEAD)

# Construct tag
TAG="ps-${PR_NUM}-${COMMIT_SHA}"
```

**Example:** PR #42, commit `a1b2c3def` → tag `ps-42-a1b2c3d`

### Step 3: Clone/update flux repo

Flux repo location: `~/.claude/rwenv/flux-repos/<rwenv-name>/`

```bash
FLUX_REPO_URL="<from rwenv config: fluxGitRepo>"
FLUX_REPO_PATH="$HOME/.claude/rwenv/flux-repos/<rwenv-name>"

if [[ -d "$FLUX_REPO_PATH" ]]; then
    cd "$FLUX_REPO_PATH" && git pull
else
    git clone "$FLUX_REPO_URL" "$FLUX_REPO_PATH"
fi
```

### Step 4: Discover flux path for service (on-demand)

Search the flux repo for the service's HelmRelease or values file:

```bash
cd "$FLUX_REPO_PATH"

# Find HelmRelease with matching name
HELM_RELEASE=$(grep -rl "name: <service>" --include="*.yaml" . | xargs grep -l "kind: HelmRelease" | head -1)

# Or find values.yaml in service directory
VALUES_FILE=$(find . -path "*/<service>/*" -name "values.yaml" | head -1)
```

Common patterns to check:
- `apps/<service>/helmrelease.yaml` - HelmRelease manifest
- `apps/<service>/values.yaml` - Helm values file
- `releases/<service>.yaml` - Release definition
- `clusters/*/apps/<service>/` - Cluster-specific paths

### Step 5: Update the image tag

Once the file is found, update the tag using `yq`:

**For values.yaml:**
```bash
yq -i '.image.tag = "<new_tag>"' <values_file>
```

**For HelmRelease with inline values:**
```bash
yq -i '.spec.values.image.tag = "<new_tag>"' <helmrelease_file>
```

**For nested image config:**
```bash
# Check structure first
yq '.image' <file>
# or
yq '.spec.values.image' <file>
```

### Step 6: Commit and push

```bash
cd "$FLUX_REPO_PATH"
git add <updated_file>
git commit -m "chore(<service>): update image tag to <new_tag>"
git push
```

### Step 7: Report success

```
Updated <service> to <tag>

Flux repo: <flux_repo_url>
File: <updated_file>
Tag: <new_tag>

Flux will reconcile automatically. To force immediate sync:
  flux reconcile kustomization <name> --with-source
```

## Example Session

**User:** rollout papi

**Claude:**
1. Checks rwenv is set → `rdebug`
2. Gets PR number → `42`
3. Gets commit SHA → `a1b2c3d` (7 chars)
4. Constructs tag → `ps-42-a1b2c3d`
5. Updates flux repo at `~/.claude/rwenv/flux-repos/rdebug/`
6. Finds `apps/papi/values.yaml`
7. Updates `.image.tag` to `ps-42-a1b2c3d`
8. Commits and pushes
9. Reports: "Updated papi to ps-42-a1b2c3d"

## Error Handling

| Error | Response |
|-------|----------|
| No rwenv set | "No rwenv set. Use `/rwenv-set` first." |
| No PR for branch | "No PR found for current branch. Create a PR first with `gh pr create`." |
| No fluxGitRepo in rwenv | "No flux repo configured for this rwenv. Add `fluxGitRepo` to envs.json." |
| Service not found in flux repo | "Could not find flux config for '<service>'. Searched for HelmRelease and values.yaml." Then ask user to provide the path. |
| Git push fails | "Failed to push to flux repo. Check permissions and try again." |

## What NOT To Do

- **DO NOT** run `git tag` or `git ls-remote --tags`
- **DO NOT** run `gcloud container images list-tags`
- **DO NOT** check GitHub Actions or build pipeline status
- **DO NOT** use 8 characters for SHA (must be exactly 7)
- **DO NOT** verify the image exists before updating
