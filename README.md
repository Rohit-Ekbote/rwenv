# Claude Plugins

A collection of Claude Code plugins for infrastructure and development workflows.

## Plugins

| Plugin | Description |
|--------|-------------|
| [rwenv](./rwenv) | Multi-cluster Kubernetes environment management with safety guardrails |

## Installation

### Option 1: Add as Marketplace (Recommended)

Add this repository as a plugin marketplace to browse and install plugins:

```bash
# Add the marketplace
claude plugins marketplace add Rohit-Ekbote/claude-plugins

# List available plugins
claude plugins marketplace list

# Install a plugin from the marketplace
claude plugins install rwenv
```

### Option 2: Direct Installation

Install a plugin directly from the repository:

```bash
claude plugins install gh:Rohit-Ekbote/claude-plugins/rwenv
```

### Option 3: Manual Installation

```bash
# Clone the repository
git clone https://github.com/Rohit-Ekbote/claude-plugins.git ~/.claude/plugins/claude-plugins

# Install a specific plugin
claude plugins install ~/.claude/plugins/claude-plugins/rwenv
```

## Managing Plugins

### List Installed Plugins

```bash
claude plugins list
```

### Upgrade Plugins

```bash
# Upgrade a specific plugin
claude plugins upgrade rwenv

# Upgrade all plugins
claude plugins upgrade --all
```

### Uninstall Plugins

```bash
claude plugins uninstall rwenv
```

### Remove Marketplace

```bash
claude plugins marketplace remove Rohit-Ekbote/claude-plugins
```

## Structure

Each plugin is a self-contained directory with:

```
<plugin-name>/
├── .claude-plugin/
│   └── plugin.json      # Plugin metadata
├── agents/              # Subagent definitions
├── hooks/               # Command hooks
├── skills/              # User-facing skills
└── README.md            # Plugin documentation
```

## License

MIT
