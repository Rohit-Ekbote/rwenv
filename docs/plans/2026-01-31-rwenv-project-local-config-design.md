# rwenv: Project-Local Configuration Design

## Problem

The current rwenv plugin uses a central registry (`~/.claude/rwenv/env-consumers.json`) to track which directory uses which environment. This causes:

1. **Worktree conflicts** - Each worktree needs explicit entry; doesn't naturally inherit or differ
2. **Multi-terminal races** - Multiple Claude sessions overwrite each other's state
3. **State fragmentation** - Multiple vestigial files created during iteration:
   - `current`, `current_env`, `current-env`, `current_env.json`
   - `currentEnvByDirectory` field in `envs.json`

## Solution

Store rwenv choice in each project/worktree locally, not centrally.

## Design

### File Structure

**Project-local (new):**
```
project/
└── .claude/
    └── rwenv              # Plain text: "rdebug"

project/.worktrees/feature-x/
└── .claude/
    └── rwenv              # Plain text: "dev-panda" (can differ)
```

**Central config (simplified):**
```
~/.claude/rwenv/
├── envs.json              # Environment definitions (read-only except /rwenv-add)
└── flux-repos/            # Cloned flux repos
```

### Behavior

| Scenario | Behavior |
|----------|----------|
| No `.claude/rwenv` in project | Commands fail: "No rwenv configured. Run /rwenv-set" |
| `/rwenv-set rdebug` | Creates `.claude/rwenv` with content `rdebug`, auto-adds to `.gitignore` |
| `/rwenv-cur` | Reads `.claude/rwenv` from current directory |
| Multiple terminals | Each reads project-local file, no conflicts |
| Worktrees | Each worktree has its own `.claude/rwenv` |

### Code Changes

**`get_current_rwenv()` - simplified:**
```bash
get_current_rwenv() {
    local dir="${1:-$PWD}"
    local rwenv_file="$dir/.claude/rwenv"

    if [[ -f "$rwenv_file" ]]; then
        cat "$rwenv_file" | tr -d '[:space:]'
        return 0
    fi

    return 1
}
```

**`set_rwenv_for_dir()` - simplified with auto-gitignore:**
```bash
set_rwenv_for_dir() {
    local dir="${1:-$PWD}"
    local rwenv_name="$2"
    local rwenv_file="$dir/.claude/rwenv"
    local gitignore="$dir/.gitignore"

    get_rwenv_by_name "$rwenv_name" >/dev/null || {
        echo "ERROR: Unknown rwenv '$rwenv_name'" >&2
        return 1
    }

    mkdir -p "$dir/.claude"
    echo "$rwenv_name" > "$rwenv_file"

    # Auto-gitignore
    if [[ -d "$dir/.git" ]] || git -C "$dir" rev-parse --git-dir >/dev/null 2>&1; then
        if ! grep -qxF '.claude/rwenv' "$gitignore" 2>/dev/null; then
            echo '.claude/rwenv' >> "$gitignore"
        fi
    fi
}
```

**Remove:**
- `load_consumers()` function

### Cleanup

**Delete from `~/.claude/rwenv/`:**
```bash
rm -f ~/.claude/rwenv/current
rm -f ~/.claude/rwenv/current_env
rm -f ~/.claude/rwenv/current-env
rm -f ~/.claude/rwenv/current_env.json
rm -f ~/.claude/rwenv/env-consumers.json
```

**Update `envs.json`:**
Remove `currentEnvByDirectory` field.

### Migration

Manual - users run `/rwenv-set` in each project as needed. No auto-migration.

## Benefits

1. **No races** - Each project has its own file, terminals can't conflict
2. **Worktree-friendly** - Each worktree naturally has its own setting
3. **Simpler code** - No parent directory walking, no central registry management
4. **Survives sessions** - File persists in project, always remembered
5. **Gitignore-safe** - Auto-ignored, won't pollute commits
