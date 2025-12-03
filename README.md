# LLM Sandboxes

Secure, isolated development containers for LLM-powered CLI tools with network firewall restrictions.

## Overview

This repository provides hardened Docker containers for running LLM CLI tools in isolated environments with strict network access controls. Each sandbox includes:

- Network firewall rules that restrict outbound traffic to approved domains only
- Modern development tooling (git, GitHub CLI, ripgrep, fzf, delta)
- ZSH with Oh My Zsh configuration
- Persistent command history and configuration
- VS Code Dev Container integration

## Available Sandboxes

### [Claude Code Sandbox](claude-cli/)

Secure environment for running Anthropic's Claude Code CLI.

- **Location**: `claude-cli/`
- **API Access**: api.anthropic.com
- **Documentation**: [claude-cli/README.md](claude-cli/README.md)

### [Codex CLI Sandbox](codex-cli/)

Secure environment for running OpenAI's Codex CLI.

- **Location**: `codex-cli/`
- **API Access**: api.openai.com
- **Documentation**: [codex-cli/README.md](codex-cli/README.md)

Both sandboxes allow access to GitHub, npm registry, and VS Code services while blocking all other outbound traffic.

## Quick Start

### Using VS Code (Recommended)

1. Clone this repository:

   ```bash
   git clone https://github.com/stitcombe/llm-sandboxes.git
   cd llm-sandboxes
   ```

2. Open a sandbox directory in VS Code:

   ```bash
   code claude-cli/
   # or
   code codex-cli/
   ```

3. Click "Reopen in Container" when prompted, or use Command Palette (`Cmd+Shift+P` / `Ctrl+Shift+P`) and select "Remote-Containers: Reopen in Container"

4. Wait for the container to build and initialize

### Using Docker Directly

```bash
cd claude-cli  # or codex-cli
docker build -t sandbox .
docker run -it --cap-add=NET_ADMIN --cap-add=NET_RAW \
  -v $(pwd):/workspace \
  sandbox
```

## Key Features

### Network Firewall

Each sandbox runs `init-firewall.sh` on startup to:

- Configure iptables with strict outbound rules
- Allow only approved domains (API endpoints, GitHub, npm, VS Code)
- Block all other internet access
- Verify firewall effectiveness automatically

### Security

- Runs as non-root `node` user
- Requires `NET_ADMIN` and `NET_RAW` capabilities for firewall
- IP-based filtering of outbound traffic
- Persistent volumes for configuration and history

### Development Tools

- Node.js v22 LTS
- ZSH with Oh My Zsh (git, fzf plugins)
- GitHub CLI (`gh`)
- git-delta (syntax-highlighted diffs)
- ripgrep, fzf, jq, curl
- nano, vim

## Documentation

For detailed information about each sandbox, including:

- Complete network access policies
- Configuration options
- Troubleshooting guides
- Customization instructions
- Security considerations

Please refer to the individual README files:

- **Claude Code Sandbox**: [claude-cli/README.md](claude-cli/README.md)
- **Codex CLI Sandbox**: [codex-cli/README.md](codex-cli/README.md)

## Prerequisites

- Docker
- Visual Studio Code with Remote-Containers extension (recommended)
- Git

## Known Issues

- **codex-cli**: Filename typo `init-filrewall.sh` (should be `init-firewall.sh`)
- **codex-cli**: Missing or incomplete `devcontainer.json` configuration

See individual sandbox READMEs for details and workarounds.

## Security Considerations

These sandboxes are designed for development and testing environments. While they provide network isolation, they should not be considered production-grade security solutions.

- Firewall rules use IP-based filtering which can change over time
- Containers run with elevated capabilities (NET_ADMIN, NET_RAW)
- Persistent volumes may contain sensitive data (API keys, tokens)

Always review and understand the firewall rules before using in sensitive environments.

## Contributing

Contributions are welcome! Please:

1. Test your changes in both sandboxes
2. Ensure firewall rules remain restrictive
3. Update documentation for any new features or domains
4. Verify both Claude Code and Codex functionality

## License

This project is provided as-is for educational and development purposes.
