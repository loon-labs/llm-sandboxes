# Developer & Agent Reference

This file provides technical reference information for developers and AI agents working with this repository.

## Repository Overview

This repository provides secure, isolated Docker development containers for LLM-powered CLI tools with network firewall restrictions. The architecture consists of three sandboxes:

- **claude-cli/**: Secure environment for Anthropic's Claude Code CLI
- **codex-cli/**: Secure environment for OpenAI's Codex CLI
- **gemini-cli/**: Secure environment for Google's Gemini CLI

Each sandbox is a self-contained Docker development container with:

- Network firewall (iptables/ipset) restricting outbound traffic to approved domains only
- Modern development tooling (git, gh, ripgrep, fzf, delta)
- ZSH with Oh My Zsh configuration
- Persistent volumes for command history and CLI configuration
- VS Code Dev Container integration

## Architecture

### Sandbox Structure

Each sandbox directory (`claude-cli/`, `codex-cli/`, `gemini-cli/`) contains:

- `.devcontainer/` - Dev container configuration
  - `Dockerfile` - Container image definition
  - `devcontainer.json` - VS Code dev container configuration
  - `init-firewall.sh` - Network firewall initialization script
- `README.md` - Sandbox-specific documentation

### Firewall Implementation

The core security mechanism is the `init-firewall.sh` script that runs on container startup:

1. **Preserves Docker DNS** - Extracts and restores Docker's internal DNS rules before flushing
2. **Flushes Rules** - Clears existing iptables and ipset configurations
3. **Creates IP Set** - Creates `allowed-domains` hash:net ipset for CIDR support
4. **Adds GitHub IPs** - Fetches GitHub IP ranges from api.github.com/meta, aggregates with `aggregate` tool
5. **Resolves Domains** - Resolves approved domains (API endpoints, npm registry, VS Code services) to IPs using `dig`
6. **Configures iptables** - Sets up INPUT/OUTPUT rules with default DROP policy
7. **Verifies** - Tests that unauthorized domains (example.com) are blocked and authorized domains work

The firewall script is located at `/usr/local/bin/init-firewall.sh` and the `node` user has passwordless sudo access to run it.

### Container Configuration

All sandboxes share similar architecture:

- Base image: `node:22`
- Non-root user: `node`
- Working directory: `/workspace`
- Required capabilities: `NET_ADMIN`, `NET_RAW` (for iptables/ipset)
- Persistent volumes for command history and CLI configuration

## Common Development Tasks

### Building a Sandbox Container

From the root directory:

```bash
cd claude-cli/  # or codex-cli/ or gemini-cli/
docker build -t <sandbox-name> .devcontainer/
```

Build with custom versions:

```bash
# Claude Code example
docker build \
  --build-arg CLAUDE_CODE_VERSION="1.2.3" \
  --build-arg TZ="America/New_York" \
  -t claude-code-sandbox .devcontainer/

# Codex example
docker build \
  --build-arg CODEX_VERSION="1.0.0" \
  --build-arg TZ="America/New_York" \
  -t codex-cli-sandbox .devcontainer/

# Gemini example
docker build \
  --build-arg GEMINI_VERSION="1.0.0" \
  --build-arg TZ="America/New_York" \
  -t gemini-cli-sandbox .devcontainer/
```

### Opening in VS Code

The recommended workflow:

```bash
code claude-cli/  # or codex-cli/ or gemini-cli/
# Click "Reopen in Container" when prompted
# Or use Command Palette: "Remote-Containers: Reopen in Container"
```

### Running with Docker Directly

```bash
# Claude Code example
cd claude-cli/
docker run -it --cap-add=NET_ADMIN --cap-add=NET_RAW \
  -v $(pwd):/workspace \
  -v claude-code-config:/home/node/.claude \
  -v claude-code-history:/commandhistory \
  claude-code-sandbox

# Codex example
cd codex-cli/
docker run -it --cap-add=NET_ADMIN --cap-add=NET_RAW \
  -e CODEX_UNSAFE_ALLOW_NO_SANDBOX=1 \
  -v $(pwd):/workspace \
  -v codex-config:/home/node/.codex \
  -v codex-history:/commandhistory \
  codex-cli-sandbox

# Gemini example
cd gemini-cli/
docker run -it --cap-add=NET_ADMIN --cap-add=NET_RAW \
  -v $(pwd):/workspace \
  -v gemini-config:/home/node/.gemini \
  -v gemini-history:/commandhistory \
  gemini-cli-sandbox
```

### Modifying Firewall Rules

To add a new allowed domain:

1. Edit `.devcontainer/init-firewall.sh` in the relevant sandbox
2. Add the domain to the resolution loop (around line 67-75):
   ```bash
   for domain in \
       "registry.npmjs.org" \
       "api.anthropic.com" \
       "your-new-domain.com"; do
   ```
3. Rebuild the container: `docker build --no-cache -t <sandbox-name> .devcontainer/`

### Debugging Firewall Issues

Check if a domain is allowed:

```bash
sudo ipset list allowed-domains | grep -A 5 "domain-or-ip"
```

View iptables rules:

```bash
sudo iptables -L -n -v
```

Test connectivity:

```bash
dig +short example-domain.com
curl -v https://example-domain.com
```

View rejected packets:

```bash
sudo iptables -L OUTPUT -v -n | grep REJECT
```

## Sandbox-Specific Details

### Claude Code Sandbox (claude-cli/)

**Allowed domains:**

- api.anthropic.com (Claude API)
- sentry.io (error reporting)
- statsig.anthropic.com, statsig.com (analytics)
- GitHub, npm registry, VS Code services

**Configuration:**

- CLI command: `claude`
- Environment: `CLAUDE_CONFIG_DIR=/home/node/.claude`
- Config volume: `claude-code-config-${devcontainerId}`
- History volume: `claude-code-bashhistory-${devcontainerId}`

### Codex CLI Sandbox (codex-cli/)

**Allowed domains:**

- api.openai.com (OpenAI API)
- GitHub, npm registry, VS Code services

**Configuration:**

- CLI command: `codex`
- Environment: `CODEX_UNSAFE_ALLOW_NO_SANDBOX=1` (Docker provides isolation, bypasses Codex's internal sandboxing)
- Environment: `CODEX_CONFIG_DIR=/home/node/.codex`
- Config volume: `codex-config-${devcontainerId}`
- History volume: `codex-bashhistory-${devcontainerId}`

### Gemini CLI Sandbox (gemini-cli/)

**Allowed domains:**

- apis.google.com (Google APIs)
- www.googleapis.com (Google API services)
- accounts.google.com (Google authentication)
- codeassist.google.com (Gemini code assistance services)
- GitHub, npm registry, VS Code services

**Configuration:**

- CLI command: `gemini`
- Environment: `GEMINI_CONFIG_DIR=/home/node/.gemini`
- Config volume: `gemini-config-${devcontainerId}`
- History volume: `gemini-bashhistory-${devcontainerId}`
- Auto-start: Gemini CLI launches automatically via `postAttachCommand`

## Security Considerations

- Containers run as non-root `node` user
- Network firewall uses IP-based filtering (IPs can change over time - may need periodic updates)
- Containers require elevated capabilities (`NET_ADMIN`, `NET_RAW`) for firewall configuration
- Persistent volumes may contain sensitive data (API keys, tokens) - handle appropriately
- Firewall rules provide network isolation but should not be considered production-grade security
- The `node` user has sudo access only to `/usr/local/bin/init-firewall.sh`

## File Locations & Conventions

### Important Files

- `README.md` - User-facing documentation (root and per-sandbox)
- `AGENTS.md` - This file, technical reference for developers and AI agents
- `.devcontainer/Dockerfile` - Container image definition
- `.devcontainer/devcontainer.json` - VS Code dev container configuration
- `.devcontainer/init-firewall.sh` - Firewall initialization script

### Directory Structure

```
llm-sandboxes/
├── README.md                           # Main user documentation
├── AGENTS.md                           # Developer/agent reference
├── claude-cli/
│   ├── README.md                       # Claude sandbox documentation
│   └── .devcontainer/
│       ├── Dockerfile
│       ├── devcontainer.json
│       └── init-firewall.sh
├── codex-cli/
│   ├── README.md                       # Codex sandbox documentation
│   └── .devcontainer/
│       ├── Dockerfile
│       ├── devcontainer.json
│       └── init-firewall.sh
└── gemini-cli/
    ├── README.md                       # Gemini sandbox documentation
    └── .devcontainer/
        ├── Dockerfile
        ├── devcontainer.json
        └── init-firewall.sh
```

## Contributing Guidelines

When making changes to this repository:

1. **Test all sandboxes** - Changes should be tested in all three sandboxes when applicable
2. **Update documentation** - Keep README files and this AGENTS.md file in sync with code changes
3. **Maintain security** - Firewall rules should remain restrictive; justify any new allowed domains
4. **Preserve consistency** - Keep similar architecture and patterns across all sandboxes
5. **Version carefully** - Use build args for version pinning when stability is required

## Common Patterns

### Adding a New Sandbox

To add a new LLM CLI sandbox:

1. Create a new directory: `<name>-cli/`
2. Copy structure from an existing sandbox (e.g., `claude-cli/`)
3. Update `Dockerfile` with the new CLI tool installation
4. Update `devcontainer.json` with appropriate extensions and config
5. Update `init-firewall.sh` with required API domains
6. Create sandbox-specific `README.md`
7. Update root `README.md` and this `AGENTS.md` file

### Updating Firewall Rules

When adding domains to `init-firewall.sh`:

1. Add to the domain resolution loop (around line 67-75)
2. Document in sandbox's README.md under "Network Access Policy"
3. Document in this file under "Sandbox-Specific Details"
4. Test that the domain resolves and is accessible after rebuild
5. Verify that unauthorized domains remain blocked

### Version Management

All sandboxes support version pinning via build args:

- Claude: `CLAUDE_CODE_VERSION`
- Codex: `CODEX_VERSION`
- Gemini: `GEMINI_VERSION`
- Git Delta: `GIT_DELTA_VERSION` (all sandboxes)
- ZSH installer: `ZSH_IN_DOCKER_VERSION` (all sandboxes)
- Timezone: `TZ` (all sandboxes)
