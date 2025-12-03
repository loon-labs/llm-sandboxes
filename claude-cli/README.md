# Claude Code Sandbox

Secure, isolated development container for running Anthropic's Claude Code CLI with network firewall restrictions.

## Overview

This Dev Container provides a hardened environment for Claude Code with:

- **Network firewall** restricting outbound access to approved domains only
- **Claude Code CLI** pre-installed and configured
- **Modern development tools** (git, GitHub CLI, ripgrep, fzf, delta)
- **ZSH shell** with Oh My Zsh and plugins
- **Persistent volumes** for command history and Claude configuration
- **VS Code extensions** for an enhanced development experience

## Quick Start

### Using VS Code

1. Open this directory in VS Code:
   ```bash
   code claude-cli/
   ```

2. When prompted, click "Reopen in Container" or use Command Palette:
   - `Cmd+Shift+P` / `Ctrl+Shift+P`
   - Select "Remote-Containers: Reopen in Container"

3. Wait for the container to build and the firewall to initialize

4. Start using Claude Code:
   ```bash
   claude
   ```

### Using Docker Directly

```bash
docker build -t claude-code-sandbox .
docker run -it --cap-add=NET_ADMIN --cap-add=NET_RAW \
  -v $(pwd):/workspace \
  -v claude-code-config:/home/node/.claude \
  -v claude-code-history:/commandhistory \
  claude-code-sandbox
```

## Network Access Policy

The firewall allows outbound connections to:

### Anthropic Services
- `api.anthropic.com` - Claude API endpoints
- `sentry.io` - Error reporting
- `statsig.anthropic.com` - Feature flags and analytics
- `statsig.com` - Analytics service

### Development Services
- GitHub (web, api, git) - All GitHub IP ranges
- `registry.npmjs.org` - npm package registry
- `marketplace.visualstudio.com` - VS Code extensions
- `vscode.blob.core.windows.net` - VS Code resources
- `update.code.visualstudio.com` - VS Code updates

### System Services
- DNS (UDP port 53)
- SSH (TCP port 22)
- Localhost (127.0.0.1)
- Docker host network

**All other outbound traffic is blocked** and will receive an `icmp-admin-prohibited` response.

## Container Configuration

### Build Arguments

| Argument | Default | Description |
|----------|---------|-------------|
| `TZ` | `America/Chicago` | Container timezone |
| `CLAUDE_CODE_VERSION` | `latest` | Version of @anthropic-ai/claude-code to install |
| `GIT_DELTA_VERSION` | `0.18.2` | Version of git-delta for enhanced diffs |
| `ZSH_IN_DOCKER_VERSION` | `1.2.0` | Version of zsh-in-docker installer |

### Capabilities

The container requires these Linux capabilities to configure the firewall:
- `NET_ADMIN` - Configure network interfaces and iptables
- `NET_RAW` - Use raw sockets and ipset

### Persistent Volumes

| Volume | Mount Point | Purpose |
|--------|-------------|---------|
| `claude-code-bashhistory-${devcontainerId}` | `/commandhistory` | Bash/ZSH command history |
| `claude-code-config-${devcontainerId}` | `/home/node/.claude` | Claude Code configuration and cache |

### Environment Variables

| Variable | Value | Description |
|----------|-------|-------------|
| `DEVCONTAINER` | `true` | Indicates running in a dev container |
| `NODE_OPTIONS` | `--max-old-space-size=4096` | Increase Node.js memory limit |
| `CLAUDE_CONFIG_DIR` | `/home/node/.claude` | Claude Code config directory |
| `POWERLEVEL9K_DISABLE_GITSTATUS` | `true` | Disable slow git status in prompt |
| `SHELL` | `/bin/zsh` | Default shell |
| `EDITOR` | `nano` | Default text editor |
| `VISUAL` | `nano` | Visual editor |

## Installed Tools

### CLI Tools
- `claude` - Anthropic's Claude Code CLI
- `gh` - GitHub CLI
- `git` - Version control
- `delta` - Syntax-highlighted git diffs
- `ripgrep` - Fast text search
- `fzf` - Fuzzy finder
- `jq` - JSON processor
- `curl` - HTTP client

### Network Tools
- `dig` - DNS lookup
- `iptables` - Firewall configuration
- `ipset` - IP set management
- `aggregate` - CIDR aggregation

### Editors
- `nano` - Simple text editor (default)
- `vim` - Advanced text editor

### Shell
- `zsh` - Default shell
- Oh My Zsh with plugins: git, fzf

## VS Code Extensions

Pre-installed extensions:
- `anthropic.claude-code` - Claude Code extension
- `dbaeumer.vscode-eslint` - ESLint integration
- `esbenp.prettier-vscode` - Code formatter
- `eamodio.gitlens` - Git history and lens

## Firewall Implementation

The firewall is initialized via `init-firewall.sh` during container startup:

1. **Preserve Docker DNS** - Extracts and restores Docker's internal DNS rules
2. **Flush Rules** - Clears existing iptables and ipset configurations
3. **Create IP Set** - Creates `allowed-domains` hash:net ipset
4. **Add GitHub IPs** - Fetches and aggregates GitHub IP ranges from api.github.com/meta
5. **Resolve Domains** - Resolves approved domains to IP addresses
6. **Configure iptables** - Sets up INPUT/OUTPUT rules with default DROP policy
7. **Verify** - Tests that example.com is blocked and api.github.com is accessible

### Firewall Script Location
`/usr/local/bin/init-firewall.sh`

The `node` user has passwordless sudo access to run this script.

## Customization

### Adding Allowed Domains

Edit `init-firewall.sh` and add domains to the resolution loop (around line 67):

```bash
for domain in \
    "registry.npmjs.org" \
    "api.anthropic.com" \
    "sentry.io" \
    "statsig.anthropic.com" \
    "statsig.com" \
    "marketplace.visualstudio.com" \
    "vscode.blob.core.windows.net" \
    "update.code.visualstudio.com" \
    "your-new-domain.com"; do  # Add your domain here
```

Rebuild the container after changes:
```bash
docker build --no-cache -t claude-code-sandbox .
```

### Changing Claude Code Version

Rebuild with a specific version:
```bash
docker build --build-arg CLAUDE_CODE_VERSION="1.2.3" -t claude-code-sandbox .
```

### Changing Timezone

Rebuild with your timezone:
```bash
docker build --build-arg TZ="America/New_York" -t claude-code-sandbox .
```

Or set it in `devcontainer.json`:
```json
{
  "build": {
    "args": {
      "TZ": "${localEnv:TZ:America/New_York}"
    }
  }
}
```

## Troubleshooting

### Claude Code Cannot Connect to API

Check if api.anthropic.com is in the allowed set:
```bash
sudo ipset list allowed-domains | grep -A 5 "anthropic"
```

Verify DNS resolution:
```bash
dig +short api.anthropic.com
```

Test connectivity:
```bash
curl -v https://api.anthropic.com
```

### Firewall Blocking Legitimate Traffic

Check iptables rules:
```bash
sudo iptables -L -n -v
```

View rejected packets:
```bash
sudo iptables -L OUTPUT -v -n | grep REJECT
```

Check firewall logs from container start:
```bash
# From host
docker logs <container-id>
```

### Cannot Install npm Packages

Verify registry.npmjs.org access:
```bash
dig +short registry.npmjs.org
curl -I https://registry.npmjs.org
```

Check if npm registry IPs are allowed:
```bash
sudo ipset list allowed-domains | grep -A 10 "registry"
```

### Container Build Fails

Clear Docker build cache:
```bash
docker build --no-cache -t claude-code-sandbox .
```

Check Docker has internet access:
```bash
docker run --rm alpine wget -O- https://api.github.com/meta
```

### ZSH Slow or Hanging

The `POWERLEVEL9K_DISABLE_GITSTATUS` variable is set to improve performance. If you still experience slowness:

```bash
# Disable git plugin
vim ~/.zshrc
# Comment out or remove 'git' from plugins array
```

## Security Notes

- The container runs as the `node` user (non-root)
- Firewall rules prevent exfiltration to unauthorized domains
- Persistent volumes may contain sensitive data (API keys)
- The `node` user has sudo access only to the firewall script
- Container requires elevated capabilities (NET_ADMIN, NET_RAW)

## Working Directory

Default workspace: `/workspace`

This is mounted from your local directory via:
- Dev Container: `workspaceMount` configuration
- Docker: `-v $(pwd):/workspace` flag

## Support

For issues with:
- **Claude Code**: Visit https://github.com/anthropics/claude-code
- **This container**: Open an issue in the parent repository
- **Firewall rules**: Check `init-firewall.sh` and iptables documentation

## License

This configuration is provided as-is for development and testing purposes.
