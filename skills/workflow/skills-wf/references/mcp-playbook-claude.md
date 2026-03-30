# MCP Server Playbook — Claude Code Addendum

Claude Code registers MCP servers in `~/.claude/settings.json`.

## stdio server registration

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "~/projects"]
    },
    "git": {
      "command": "uvx",
      "args": ["mcp-server-git", "--repository", "."]
    },
    "skills-validator": {
      "command": "uv",
      "args": ["run", "--with", "python-frontmatter", "${SKILLFORGE_DIR}/mcp/skills_validator.py"]
    }
  }
}
```

## SSE server registration

```json
{
  "mcpServers": {
    "skills": {
      "url": "http://localhost:3100/sse"
    }
  }
}
```

Replace `localhost:3100` with the remote host's SSE URL when migrating to a dedicated host (see `mcp-playbook.md` Phase 1–3).

## Notes

- Changes to `~/.claude/settings.json` take effect on next Claude Code session start.
- Claude Code discovers servers at startup; changes mid-session require a restart.
- The `settings.json` file is user-scoped — it applies to all projects, not just Skill Forge.
