# MCP Server Playbook

## Overview

MCP (Model Context Protocol) servers extend LLM tooling with new tools. This playbook
covers running servers locally on a laptop and migrating to a dedicated always-on host.

For LLM-specific server registration steps (e.g. Claude Code `settings.json`),
see the companion addendum: `references/mcp-playbook-claude.md`.

---

## Part 1 — Laptop Setup

### Prerequisites

```bash
brew install node          # or: brew install nvm && nvm install --lts
brew install python@3.12   # or: brew install pyenv
brew install uv            # fast Python package manager — replaces pip for MCP servers
```

### Transport types

| Transport | How the LLM connects | Best for |
|---|---|---|
| **stdio** | LLM tool spawns the process on demand | Local servers, no daemon needed |
| **SSE** | LLM tool connects to a running URL | Remote servers, persistent state |

### Registering servers

Register servers in your LLM tool's config file.
See `references/mcp-playbook-claude.md` for Claude Code examples.

Typical patterns:

**stdio (spawned on demand — simplest):** provide a `command` and `args` pointing to the server binary.

**SSE (persistent server, survives LLM tool restarts):** provide the server URL.

### Keeping SSE servers alive with launchd

For stdio servers: no daemon needed — the LLM tool spawns and tears them down automatically.

For SSE servers, create `~/Library/LaunchAgents/com.user.mcp-<name>.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.user.mcp-skills</string>
  <key>ProgramArguments</key>
  <array>
    <string>/usr/local/bin/uv</string>
    <string>run</string>
    <string>${SKILLFORGE_DIR}/mcp/skills_server.py</string>
  </array>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
  <key>StandardOutPath</key><string>/tmp/mcp-skills.log</string>
  <key>StandardErrorPath</key><string>/tmp/mcp-skills.err</string>
</dict>
</plist>
```

```bash
launchctl load   ~/Library/LaunchAgents/com.user.mcp-skills.plist   # start
launchctl unload ~/Library/LaunchAgents/com.user.mcp-skills.plist   # stop
launchctl list | grep mcp                                            # status
```

### Recommended servers for current skills

| Server | Package | Transport | Skill it enhances |
|---|---|---|---|
| Filesystem | `@modelcontextprotocol/server-filesystem` | stdio | skills-wf: safer `references/` writes |
| Git | `mcp-server-git` (uvx) | stdio | git-commit-wf: richer diff/log data |
| Bandit wrapper | custom Python | stdio | security-sme: static analysis on staged files |
| Skills validator | custom Python | stdio | skills-wf: SKILL.md frontmatter check |

---

## Part 2 — Dedicated Host Migration Roadmap

### Why migrate

| Laptop | Dedicated host |
|---|---|
| Sleeps — interrupts long tasks | Always on |
| Shares CPU/RAM with dev work | Dedicated to serving |
| stdio only (local process) | Exposes SSE over LAN or VPN |

### Phase 1 — Same network, local SSE

1. Give the host a static LAN IP or stable mDNS name (e.g. `mac-mini.local`).
2. Install servers on the host; register them as launchd agents using the plist pattern above.
3. Update the LLM tool's server config on the laptop to point to the SSE URL:
   `"skills": { "url": "http://mac-mini.local:3100/sse" }`
4. Firewall rule: allow port 3100 inbound from LAN only.

### Phase 2 — Secure remote access via Tailscale

1. Install Tailscale on both machines:
   ```bash
   brew install --cask tailscale
   ```
2. Join the same Tailnet. The host gets a stable `100.x.x.x` address that works from anywhere.
3. Replace the mDNS name with the Tailscale IP in the server config.
4. No router port-forwarding needed — Tailscale handles the encrypted tunnel.

### Phase 3 — Centralised MCP proxy

Run one multiplexing proxy on the host so the laptop makes a single SSE connection:

```
Laptop (LLM tool)
    │  SSE :3100
    ▼
Host  ←  mcp-proxy
    ├── :3101  skills-validator  (Python/uv)
    ├── :3102  git server        (Node/npx)
    └── :3103  bandit wrapper    (Python/uv)
```

Use [`mcp-proxy`](https://github.com/sparfenyuk/mcp-proxy) or a minimal nginx SSE reverse proxy to multiplex.

### Migration checklist

- [ ] Static IP or mDNS name assigned on host
- [ ] launchd plists deployed and verified (`launchctl list | grep mcp`)
- [ ] LAN ports confirmed reachable from laptop (`curl http://mac-mini.local:3100/sse`)
- [ ] LLM tool server config updated to SSE URLs
- [ ] Logs tailed on first use (`tail -f /tmp/mcp-*.log`)
- [ ] Tailscale installed and joined (Phase 2)
- [ ] Proxy configured with routing rules (Phase 3)
