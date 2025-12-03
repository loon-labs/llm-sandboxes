# Threads Post

Running AI coding assistants in production? You might want network isolation.

I built secure Docker sandboxes for Claude Code and Codex CLI with iptables-based firewalls that whitelist only approved domains (API endpoints, GitHub, npm). Everything else gets blocked.

Key features:
- IP-based filtering with automatic GitHub range aggregation
- VS Code Dev Container integration
- Runs as non-root with minimal capabilities
- Full dev environment (zsh, ripgrep, fzf, gh CLI)

Perfect for environments where you need AI assistance but can't risk unrestricted network access.

https://github.com/stitcombe/llm-sandboxes
