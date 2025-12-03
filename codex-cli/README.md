# Codex CLI Sandbox

Secure, isolated development container for running OpenAI's Codex CLI with network firewall restrictions.

## Overview

This Dev Container provides a hardened environment for Codex CLI with:

- **Network firewall** restricting outbound access to approved domains only
- **Codex CLI** pre-installed and configured
- **Sandbox bypass** (`CODEX_UNSAFE_ALLOW_NO_SANDBOX=1`) since container provides isolation
- **Modern development tools** (git, GitHub CLI, ripgrep, fzf, delta)
- **ZSH shell** with Oh My Zsh and plugins
- **Persistent volumes** for command history and Codex configuration
- **VS Code extensions** for an enhanced development experience

## Quick Start

### Using VS Code

1. Open this directory in VS Code:
   ```bash
   code codex-cli/
   ```

2. When prompted, click "Reopen in Container" or use Command Palette:
   - `Cmd+Shift+P` / `Ctrl+Shift+P`
   - Select "Remote-Containers: Reopen in Container"

3. Wait for the container to build and the firewall to initialize

4. Start using Codex:
   ```bash
   codex
   ```

### Using Docker Directly

```bash
docker build -t codex-cli-sandbox .
docker run -it --cap-add=NET_ADMIN --cap-add=NET_RAW \
  -e CODEX_UNSAFE_ALLOW_NO_SANDBOX=1 \
  -v $(pwd):/workspace \
  -v codex-config:/home/node/.codex \
  -v codex-history:/commandhistory \
  codex-cli-sandbox
```

## Network Access Policy

The firewall allows outbound connections to:

### OpenAI Services
- `api.openai.com` - OpenAI API endpoints for Codex

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
| `TZ` | Inherited from host | Container timezone |
| `CODEX_VERSION` | `latest` | Version of @openai/codex to install |
| `GIT_DELTA_VERSION` | `0.18.2` | Version of git-delta for enhanced diffs |
| `ZSH_IN_DOCKER_VERSION` | `1.2.0` | Version of zsh-in-docker installer |

### Capabilities

The container requires these Linux capabilities to configure the firewall:
- `NET_ADMIN` - Configure network interfaces and iptables
- `NET_RAW` - Use raw sockets and ipset

### Environment Variables

| Variable | Value | Description |
|----------|-------|-------------|
| `DEVCONTAINER` | `true` | Indicates running in a dev container |
| `CODEX_UNSAFE_ALLOW_NO_SANDBOX` | `1` | Bypass Codex sandboxing (container provides isolation) |
| `SHELL` | `/bin/zsh` | Default shell |
| `EDITOR` | `nano` | Default text editor |
| `VISUAL` | `nano` | Visual editor |

### Note on Sandboxing

The `CODEX_UNSAFE_ALLOW_NO_SANDBOX=1` environment variable is set because:
- The Docker container itself provides process isolation
- The network firewall restricts external access
- The container runs with limited capabilities
- This avoids nested sandboxing which can cause issues

## Installed Tools

### CLI Tools
- `codex` - OpenAI's Codex CLI
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

## Known Issues

### Filename Typo

There is a typo in the firewall script filename:
- Current: `init-filrewall.sh` (incorrect)
- Expected: `init-firewall.sh` (correct)

The Dockerfile correctly references `init-firewall.sh`, so ensure the file is named correctly.

### Missing devcontainer.json

The `devcontainer.json` file currently appears to be empty or incomplete. A complete configuration should include:

```json
{
  "name": "Codex CLI Sandbox",
  "build": {
    "dockerfile": "Dockerfile",
    "args": {
      "TZ": "${localEnv:TZ:America/Chicago}",
      "CODEX_VERSION": "latest",
      "GIT_DELTA_VERSION": "0.18.2",
      "ZSH_IN_DOCKER_VERSION": "1.2.0"
    }
  },
  "runArgs": ["--cap-add=NET_ADMIN", "--cap-add=NET_RAW"],
  "remoteUser": "node",
  "workspaceMount": "source=${localWorkspaceFolder},target=/workspace,type=bind",
  "workspaceFolder": "/workspace",
  "postStartCommand": "sudo /usr/local/bin/init-firewall.sh",
  "waitFor": "postStartCommand"
}
```

## Customization

### Adding Allowed Domains

Edit `init-firewall.sh` and add domains to the resolution loop (around line 67):

```bash
for domain in \
    "registry.npmjs.org" \
    "api.openai.com" \
    "marketplace.visualstudio.com" \
    "vscode.blob.core.windows.net" \
    "update.code.visualstudio.com" \
    "your-new-domain.com"; do  # Add your domain here
```

Rebuild the container after changes:
```bash
docker build --no-cache -t codex-cli-sandbox .
```

### Changing Codex Version

Rebuild with a specific version:
```bash
docker build --build-arg CODEX_VERSION="1.2.3" -t codex-cli-sandbox .
```

### Changing Timezone

Rebuild with your timezone:
```bash
docker build --build-arg TZ="America/New_York" -t codex-cli-sandbox .
```

## Troubleshooting

### Codex Cannot Connect to API

Check if api.openai.com is in the allowed set:
```bash
sudo ipset list allowed-domains | grep -A 5 "openai"
```

Verify DNS resolution:
```bash
dig +short api.openai.com
```

Test connectivity:
```bash
curl -v https://api.openai.com
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
docker build --no-cache -t codex-cli-sandbox .
```

Check Docker has internet access:
```bash
docker run --rm alpine wget -O- https://api.github.com/meta
```

### Codex Complains About Sandboxing

The `CODEX_UNSAFE_ALLOW_NO_SANDBOX=1` environment variable should suppress sandbox warnings. If you still see them:

1. Verify the environment variable is set:
   ```bash
   echo $CODEX_UNSAFE_ALLOW_NO_SANDBOX
   ```

2. Check if it's set in the container:
   ```bash
   docker inspect <container-id> | grep CODEX_UNSAFE
   ```

3. Ensure it's in the Dockerfile and devcontainer.json

## Security Notes

- The container runs as the `node` user (non-root)
- Firewall rules prevent exfiltration to unauthorized domains
- Codex sandbox is disabled (`CODEX_UNSAFE_ALLOW_NO_SANDBOX=1`) - container provides isolation instead
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
- **Codex CLI**: Visit OpenAI's documentation
- **This container**: Open an issue in the parent repository
- **Firewall rules**: Check `init-firewall.sh` and iptables documentation

## License

This configuration is provided as-is for development and testing purposes.
