# Claude Plugins

A collection of Claude Code plugins for infrastructure and development workflows.

## Plugins

| Plugin | Description |
|--------|-------------|
| [rwenv](./rwenv) | Multi-cluster Kubernetes environment management with safety guardrails |

## Installation

Install a plugin from this repository:

```bash
# Clone the repository
git clone https://github.com/Rohit-Ekbote/claude-plugins.git ~/.claude/plugins/claude-plugins

# Install a specific plugin
claude plugins install ~/.claude/plugins/claude-plugins/rwenv
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
