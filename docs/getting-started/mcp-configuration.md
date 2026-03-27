# MCP configuration

yorishiro-proxy runs as an MCP server. By default, it starts an HTTP MCP transport on a random loopback port. To use it with Claude Code via stdio transport, you need a `.mcp.json` file in your project root or home directory that launches the proxy with `-stdio-mcp`.

!!! tip
    You can automate this setup by running `yorishiro-proxy install` (see [Quick setup](quick-setup.md)).

## Transport modes

yorishiro-proxy supports two MCP transport modes:

| Mode | Default | Description |
|------|---------|-------------|
| **HTTP MCP** | Enabled (`127.0.0.1:0`) | Streamable HTTP transport + Web UI. The server writes connection details to `~/.yorishiro-proxy/server.json` for automatic discovery |
| **Stdio MCP** | Disabled | Stdin/stdout transport for direct integration with MCP clients like Claude Code |

You can enable both transports simultaneously, or disable HTTP MCP for stdio-only mode.

## Manual configuration

Create a `.mcp.json` file in your project root with the following content:

```json
{
  "mcpServers": {
    "yorishiro-proxy": {
      "command": "/path/to/bin/yorishiro-proxy",
      "args": ["server", "-stdio-mcp", "-insecure", "-log-file", "/tmp/yorishiro-proxy.log"]
    }
  }
}
```

Replace `/path/to/bin/yorishiro-proxy` with the actual path to the binary. For example:

- If built from source: `"/home/user/yorishiro-proxy/bin/yorishiro-proxy"`
- If downloaded from GitHub Releases: `"/usr/local/bin/yorishiro-proxy"`

!!! note
    The `server` subcommand and `-stdio-mcp` flag are required when launching from `.mcp.json`. Claude Code communicates over stdin/stdout, so stdio MCP must be explicitly enabled. The `server` subcommand is optional (it is the default), but including it makes the intent clear.

## CLI flags

Common flags to include in the `args` array:

| Flag | Default | Description |
|------|---------|-------------|
| `-stdio-mcp` | `false` | Enable stdio MCP transport (required for Claude Code integration via `.mcp.json`) |
| `-no-http-mcp` | `false` | Disable HTTP MCP transport (use with `-stdio-mcp` for stdio-only mode) |
| `-insecure` | `false` | Skip upstream TLS certificate verification (useful for testing) |
| `-log-file <path>` | stderr | Write logs to a file instead of stderr (keeps MCP stdio clean) |
| `-log-level <level>` | `info` | Log verbosity: `debug`, `info`, `warn`, `error` |
| `-log-format <format>` | `text` | Log format: `text`, `json` |
| `-db <name-or-path>` | `~/.yorishiro-proxy/yorishiro.db` | SQLite database path or project name |
| `-ca-ephemeral` | `false` | Use an ephemeral in-memory CA (no persistent certificate files) |
| `-tls-fingerprint <profile>` | -- | TLS fingerprint profile: `chrome`, `firefox`, `safari`, `edge`, `random`, `none` |
| `-config <path>` | -- | JSON config file path for proxy defaults |
| `-mcp-http-addr <host:port>` | `127.0.0.1:0` | Streamable HTTP listen address (also serves the Web UI) |
| `-mcp-http-token <token>` | auto-generated | HTTP Bearer auth token for the Web UI |
| `-open-browser` | `false` | Open WebUI in browser when HTTP MCP starts |
| `-target-policy-file <path>` | -- | Target scope policy JSON file path |
| `-safety-filter` | `false` | Enable SafetyFilter engine |

The `-db` flag accepts an absolute path, a relative path with extension, or a plain project name. A project name (e.g., `pentest-2026`) resolves to `~/.yorishiro-proxy/pentest-2026.db`, making it easy to maintain separate databases per engagement.

### Example with stdio + HTTP MCP

When both transports are active, Claude Code communicates via stdio while you can also access the Web UI and use the `client` subcommand:

```json
{
  "mcpServers": {
    "yorishiro-proxy": {
      "command": "/path/to/bin/yorishiro-proxy",
      "args": [
        "server",
        "-stdio-mcp",
        "-insecure",
        "-log-file", "/tmp/yorishiro-proxy.log",
        "-mcp-http-addr", "127.0.0.1:3000"
      ]
    }
  }
}
```

### Example with stdio only (no HTTP MCP)

To disable the HTTP MCP transport and use stdio only (similar to pre-v0.14.0 behavior):

```json
{
  "mcpServers": {
    "yorishiro-proxy": {
      "command": "/path/to/bin/yorishiro-proxy",
      "args": [
        "server",
        "-stdio-mcp",
        "-no-http-mcp",
        "-insecure",
        "-log-file", "/tmp/yorishiro-proxy.log"
      ]
    }
  }
}
```

## Environment variables

All flags accept environment variables with the `YP_` prefix as a fallback. Replace hyphens with underscores and uppercase the name.

| Flag | Environment variable |
|------|---------------------|
| `-db` | `YP_DB` |
| `-insecure` | `YP_INSECURE` |
| `-log-level` | `YP_LOG_LEVEL` |
| `-log-format` | `YP_LOG_FORMAT` |
| `-log-file` | `YP_LOG_FILE` |
| `-ca-cert` | `YP_CA_CERT` |
| `-ca-key` | `YP_CA_KEY` |
| `-ca-ephemeral` | `YP_CA_EPHEMERAL` |
| `-config` | `YP_CONFIG` |
| `-tls-fingerprint` | `YP_TLS_FINGERPRINT` |
| `-mcp-http-addr` | `YP_MCP_HTTP_ADDR` |
| `-mcp-http-token` | `YP_MCP_HTTP_TOKEN` |
| `-no-http-mcp` | `YP_NO_HTTP_MCP` |
| `-stdio-mcp` | `YP_STDIO_MCP` |
| `-open-browser` | `YP_OPEN_BROWSER` |
| `-target-policy-file` | `YP_TARGET_POLICY_FILE` |
| `-safety-filter` | `YP_SAFETY_FILTER_ENABLED` |

Priority: CLI flag > environment variable > config file > default value.

For example, to skip TLS verification via environment variable:

```bash
YP_INSECURE=true yorishiro-proxy server
```

## Verify the connection

After creating `.mcp.json`, restart Claude Code. You should see eleven MCP tools become available:

| Tool | Description |
|------|-------------|
| `proxy_start` | Start the proxy listener |
| `proxy_stop` | Stop the proxy listener |
| `configure` | Change runtime settings (capture scope, intercept rules, TLS passthrough, etc.) |
| `query` | Retrieve flows, status, configuration, and fuzz results |
| `resend` | Resend captured requests with optional mutations |
| `fuzz` | Start, pause, resume, and cancel fuzz campaigns |
| `macro` | Define and run multi-step macro workflows |
| `intercept` | Release, modify, or drop intercepted requests |
| `manage` | Delete, export, and import flows |
| `security` | Configure target scope rules, rate limits, and budgets |
| `plugin` | List, reload, enable, and disable Starlark plugins |

If the tools do not appear, check the log file for errors. Common issues include an incorrect binary path or permission problems.

## Next steps

- [CA certificate](ca-certificate.md) -- set up HTTPS interception
- [First capture](first-capture.md) -- capture your first traffic
- [CLI flags](../configuration/cli-flags.md) -- full CLI flag reference
