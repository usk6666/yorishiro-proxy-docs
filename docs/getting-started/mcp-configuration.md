# MCP configuration

yorishiro-proxy runs as an MCP server communicating over stdin/stdout. To connect it to Claude Code, you need a `.mcp.json` file in your project root or home directory.

!!! tip
    You can automate this setup by running `yorishiro-proxy install` (see [Quick setup](quick-setup.md)).

## Manual configuration

Create a `.mcp.json` file in your project root with the following content:

```json
{
  "mcpServers": {
    "yorishiro-proxy": {
      "command": "/path/to/bin/yorishiro-proxy",
      "args": ["-insecure", "-log-file", "/tmp/yorishiro-proxy.log"]
    }
  }
}
```

Replace `/path/to/bin/yorishiro-proxy` with the actual path to the binary. For example:

- If built from source: `"/home/user/yorishiro-proxy/bin/yorishiro-proxy"`
- If downloaded from GitHub Releases: `"/usr/local/bin/yorishiro-proxy"`

## CLI flags

Common flags to include in the `args` array:

| Flag | Default | Description |
|------|---------|-------------|
| `-insecure` | `false` | Skip upstream TLS certificate verification (useful for testing) |
| `-log-file <path>` | stderr | Write logs to a file instead of stderr (keeps MCP stdio clean) |
| `-log-level <level>` | `info` | Log verbosity: `debug`, `info`, `warn`, `error` |
| `-log-format <format>` | `text` | Log format: `text`, `json` |
| `-db <name-or-path>` | `~/.yorishiro-proxy/yorishiro.db` | SQLite database path or project name |
| `-ca-ephemeral` | `false` | Use an ephemeral in-memory CA (no persistent certificate files) |
| `-tls-fingerprint <profile>` | `chrome` | TLS fingerprint profile: `chrome`, `firefox`, `safari`, `edge`, `random`, `none` |
| `-config <path>` | -- | JSON config file path for proxy defaults |
| `-mcp-http-addr <host:port>` | -- | Enable Streamable HTTP transport and serve the WebUI |
| `-mcp-http-token <token>` | auto-generated | HTTP Bearer auth token for the WebUI |
| `-target-policy-file <path>` | -- | Target scope policy JSON file path |
| `-safety-filter` | `false` | Enable SafetyFilter engine |
| `-no-open-browser` | `false` | Disable auto-opening WebUI in browser |

The `-db` flag accepts an absolute path, a relative path with extension, or a plain project name. A project name (e.g., `pentest-2026`) resolves to `~/.yorishiro-proxy/pentest-2026.db`, making it easy to maintain separate databases per engagement.

### Example with WebUI enabled

```json
{
  "mcpServers": {
    "yorishiro-proxy": {
      "command": "/path/to/bin/yorishiro-proxy",
      "args": [
        "-insecure",
        "-log-file", "/tmp/yorishiro-proxy.log",
        "-mcp-http-addr", "127.0.0.1:3000"
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
| `-target-policy-file` | `YP_TARGET_POLICY_FILE` |
| `-safety-filter` | `YP_SAFETY_FILTER_ENABLED` |
| `-no-open-browser` | `YP_NO_OPEN_BROWSER` |

Priority: CLI flag > environment variable > config file > default value.

For example, to skip TLS verification via environment variable:

```bash
YP_INSECURE=true yorishiro-proxy
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
