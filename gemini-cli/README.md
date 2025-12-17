# Gemini CLI Sandbox

Secure, isolated development container for running Google's Gemini CLI with network firewall restrictions.

## Overview

This Dev Container provides a hardened environment for Gemini CLI with:

- **Network firewall** restricting outbound access to approved domains only
- **Gemini CLI** pre-installed and configured
- **Modern development tools** (git, GitHub CLI, ripgrep, fzf, delta)
- **ZSH shell** with Oh My Zsh and plugins
- **Persistent volumes** for command history and Gemini configuration
- **VS Code extensions** for an enhanced development experience
- **Auto-start** - Gemini CLI launches automatically when container attaches

## Quick Start

### Using VS Code

1. Open this directory in VS Code:
   ```bash
   code gemini-cli/
   ```

2. When prompted, click "Reopen in Container" or use Command Palette:
   - `Cmd+Shift+P` / `Ctrl+Shift+P`
   - Select "Remote-Containers: Reopen in Container"

3. Wait for the container to build and the firewall to initialize

4. Gemini CLI will start automatically, or run manually:
   ```bash
   gemini
   ```

### Using Docker Directly

```bash
docker build -t gemini-cli-sandbox .
docker run -it --cap-add=NET_ADMIN --cap-add=NET_RAW \
  -v $(pwd):/workspace \
  -v gemini-config:/home/node/.gemini \
  -v gemini-history:/commandhistory \
  gemini-cli-sandbox
```

## Network Access Policy

The firewall allows outbound connections to:

### Google Services
- `apis.google.com` - Google APIs
- `www.googleapis.com` - Google API services
- `accounts.google.com` - Google authentication
- `codeassist.google.com` - Gemini code assistance services

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
| `GEMINI_VERSION` | `latest` | Version of @google/gemini-cli to install |
| `GIT_DELTA_VERSION` | `0.18.2` | Version of git-delta for enhanced diffs |
| `ZSH_IN_DOCKER_VERSION` | `1.2.0` | Version of zsh-in-docker installer |

### Capabilities

The container requires these Linux capabilities to configure the firewall:
- `NET_ADMIN` - Configure network interfaces and iptables
- `NET_RAW` - Use raw sockets and ipset

### Persistent Volumes

| Volume | Mount Point | Purpose |
|--------|-------------|---------|
| `gemini-bashhistory-${devcontainerId}` | `/commandhistory` | Bash/ZSH command history |
| `gemini-config-${devcontainerId}` | `/home/node/.gemini` | Gemini CLI configuration and cache |

### Environment Variables

| Variable | Value | Description |
|----------|-------|-------------|
| `DEVCONTAINER` | `true` | Indicates running in a dev container |
| `NODE_OPTIONS` | `--max-old-space-size=4096` | Increase Node.js memory limit |
| `GEMINI_CONFIG_DIR` | `/home/node/.gemini` | Gemini CLI config directory |
| `POWERLEVEL9K_DISABLE_GITSTATUS` | `true` | Disable slow git status in prompt |
| `SHELL` | `/bin/zsh` | Default shell |
| `EDITOR` | `nano` | Default text editor |
| `VISUAL` | `nano` | Visual editor |

## Installed Tools

### CLI Tools
- `gemini` - Google's Gemini CLI
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
- `Google.gemini-cli-vscode-ide-companion` - Gemini CLI VS Code companion
- `dbaeumer.vscode-eslint` - ESLint integration
- `esbenp.prettier-vscode` - Code formatter

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
    "apis.google.com" \
    "www.googleapis.com" \
    "accounts.google.com" \
    "codeassist.google.com" \
    "registry.npmjs.org" \
    "marketplace.visualstudio.com" \
    "vscode.blob.core.windows.net" \
    "update.code.visualstudio.com" \
    "your-new-domain.com"; do  # Add your domain here
```

Rebuild the container after changes:
```bash
docker build --no-cache -t gemini-cli-sandbox .
```

### Changing Gemini CLI Version

Rebuild with a specific version:
```bash
docker build --build-arg GEMINI_VERSION="1.2.3" -t gemini-cli-sandbox .
```

### Changing Timezone

Rebuild with your timezone:
```bash
docker build --build-arg TZ="America/New_York" -t gemini-cli-sandbox .
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

### Disabling Auto-Start

To prevent Gemini CLI from auto-starting, edit `devcontainer.json` and remove or comment out:
```json
"postAttachCommand": "gemini",
```

## Troubleshooting

### Gemini CLI Cannot Connect to API

Check if Google APIs are in the allowed set:
```bash
sudo ipset list allowed-domains | grep -A 5 "google"
```

Verify DNS resolution:
```bash
dig +short apis.google.com
dig +short www.googleapis.com
```

Test connectivity:
```bash
curl -v https://apis.google.com
curl -v https://www.googleapis.com
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
docker build --no-cache -t gemini-cli-sandbox .
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

### Authentication Issues

If you need to authenticate with Google services:

1. Ensure `accounts.google.com` is accessible:
   ```bash
   curl -I https://accounts.google.com
   ```

2. Check Gemini CLI configuration:
   ```bash
   ls -la ~/.gemini
   ```

3. Re-authenticate if needed:
   ```bash
   gemini auth login
   ```

## Security Notes

- The container runs as the `node` user (non-root)
- Firewall rules prevent exfiltration to unauthorized domains
- Persistent volumes may contain sensitive data (API keys, tokens)
- The `node` user has sudo access only to the firewall script
- Container requires elevated capabilities (NET_ADMIN, NET_RAW)
- Google service authentication tokens are stored in `/home/node/.gemini`

## Working Directory

Default workspace: `/workspace`

This is mounted from your local directory via:
- Dev Container: `workspaceMount` configuration
- Docker: `-v $(pwd):/workspace` flag

## Support

For issues with:
- **Gemini CLI**: Visit Google's Gemini CLI documentation
- **This container**: Open an issue in the parent repository
- **Firewall rules**: Check `init-firewall.sh` and iptables documentation

## License

This configuration is provided as-is for development and testing purposes.
