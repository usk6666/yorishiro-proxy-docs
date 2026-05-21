# Quick setup

The `yorishiro-proxy install` command performs MCP configuration and CA certificate generation in a single step. This is the fastest way to get started.

## One-command setup

Run the install command to configure everything at once:

```bash
yorishiro-proxy install
```

This performs the following:

1. Registers yorishiro-proxy as an MCP server in your project-level `.mcp.json`
2. Generates the CA certificate for HTTPS interception

To also register the CA certificate in your OS trust store (requires `sudo` on macOS/Linux, or Administrator on Windows):

```bash
yorishiro-proxy install --trust
```

!!! note "Vulnerability-verification playbooks ship over MCP"
    yorishiro-proxy used to copy a `.claude/skills/yorishiro/` skill tree into your project as part of `install`. That tree is gone — the same playbooks are now delivered to your MCP host as [MCP Prompts](../guides/mcp-prompts.md) and become visible automatically the first time the host connects. No extra install step is required. The `install skills` subcommand and the `--skills-dir` flag have been removed.

## Individual setup targets

You can run specific setup targets independently:

| Command | Description |
|---------|-------------|
| `yorishiro-proxy install mcp` | Configure MCP server integration only |
| `yorishiro-proxy install ca` | Generate the CA certificate only |
| `yorishiro-proxy install playwright` | Set up playwright-cli integration (auto-detects browser, installs if needed) |

For example, to generate the CA certificate and register it in your OS trust store without touching MCP configuration:

```bash
yorishiro-proxy install ca --trust
```

## Additional flags

| Flag | Description |
|------|-------------|
| `--trust` | Register the CA certificate in the OS trust store |
| `--interactive` | Launch a wizard that walks you through each step |
| `--user-scope` | Register MCP configuration in the user-level settings file (`~/.claude/settings.json`) instead of the project-level `.mcp.json` |

## Upgrading

Update yorishiro-proxy to the latest release from GitHub:

```bash
yorishiro-proxy upgrade
```

To check for updates without installing:

```bash
yorishiro-proxy upgrade --check
```

The bundled MCP prompts ship inside the binary, so a fresh `yorishiro-proxy upgrade` automatically delivers the latest playbook content the next time your MCP host reconnects — no separate install step is needed.

## Next steps

- [MCP configuration](mcp-configuration.md) -- manual configuration and CLI flags
- [CA certificate](ca-certificate.md) -- manual CA certificate setup for HTTPS interception
- [MCP prompts](../guides/mcp-prompts.md) -- vulnerability-verification playbooks delivered over MCP
