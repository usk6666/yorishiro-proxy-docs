# CLI flags

yorishiro-proxy accepts command-line flags, environment variables, and a JSON config file to control its behavior. This page documents all available subcommands, flags, and the priority rules that determine which value takes effect.

## Priority order

When the same setting is specified in multiple places, the following priority applies:

1. **CLI flag** (highest priority)
2. **Environment variable** (`YP_` prefix)
3. **Config file** (`-config`)
4. **Default value** (lowest priority)

## Subcommands

yorishiro-proxy uses a subcommand-based CLI. When no subcommand is given, it defaults to `server` for backward compatibility.

```
yorishiro-proxy [subcommand] [flags]
```

| Subcommand | Description |
|------------|-------------|
| `server` | Start the proxy server (default when no subcommand given) |
| `client` | Call MCP tools via CLI against a running server |
| `docs` | Show tool documentation (works offline, no server needed) |
| `install` | Install and configure components (MCP, CA, Playwright) |
| `upgrade` | Check for and install updates from GitHub Releases |
| `version` | Print version information |

### `server`

Start the proxy server. This is the default behavior when no subcommand is given.

```bash
yorishiro-proxy server [flags]
yorishiro-proxy [flags]          # equivalent — "server" is implicit
```

By default, the server starts an HTTP MCP transport on a random loopback port (`127.0.0.1:0`) and writes connection details to `~/.yorishiro-proxy/server.json` for automatic client discovery.

### `client`

Call MCP tools on a running server from the command line. Useful for scripting, automation, and ad-hoc pentest workflows.

```bash
yorishiro-proxy client <tool> [key=value ...] [flags]
```

The client resolves the server address in this order:

1. `-server-addr` flag (or `YP_CLIENT_ADDR` env var)
2. `-token` flag (or `YP_CLIENT_TOKEN` env var)
3. Auto-detection from `~/.yorishiro-proxy/server.json`

#### Client flags

| Flag | Env variable | Description |
|------|-------------|-------------|
| `-server-addr` | `YP_CLIENT_ADDR` | Server address (host:port) |
| `-token` | `YP_CLIENT_TOKEN` | Bearer token for authentication |
| `--format` | `YP_CLIENT_FORMAT` | Output format: `json`, `table`, `raw`, `text` |
| `--raw` | -- | Compact JSON (overrides `--format`) |
| `-q` / `--quiet` | -- | Suppress output on success |

When stdout is a TTY, the default format is `json` (pretty-printed). When piped, it switches to `raw` (compact JSON). The `docs` tool defaults to `text`, which prints the document unescaped.

Values are coerced to the type declared in the tool's JSON Schema, including nullable types and nested dot-notation paths (for example `params.max_requests_per_second=100` is sent as a number, not a string).

#### Client examples

```bash
# Query proxy status (auto-detect server from server.json)
yorishiro-proxy client query resource=status

# Start a listener
yorishiro-proxy client proxy_start listen_addr=127.0.0.1:8080

# List flows with filters
yorishiro-proxy client query resource=flows limit=20 filter.method=POST

# Replay a request
yorishiro-proxy client resend_http flow_id=abc123

# Configure upstream proxy
yorishiro-proxy client configure upstream_proxy=http://proxy:8888

# Positional arguments (tool-specific)
yorishiro-proxy client query flows         # equivalent to resource=flows
yorishiro-proxy client query flow abc123   # equivalent to resource=flow id=abc123

# Explicit connection to a specific server
yorishiro-proxy client -server-addr 127.0.0.1:4000 -token my-token query resource=status

# Pipe output to jq
yorishiro-proxy client query resource=flows --format raw | jq '.[] | select(.method == "POST")'

# Per-tool help (no server connection needed)
yorishiro-proxy client query --help

# Read the docs tool through a running server
yorishiro-proxy client docs topic=fuzz_http section="Macro hook scopes"
```

### `docs`

Print yorishiro-proxy's own documentation without contacting a server. The documents are embedded in the binary, so this works offline — useful when you are still setting up the proxy, or when driving it from CLI-based workflows. It serves the same corpus as the [`docs` MCP tool](../tools/docs.md).

```bash
yorishiro-proxy docs [topic] [section] [flags]
```

| Argument / flag | Description |
|-----------------|-------------|
| `topic` | Document to print. Omit to list every topic. Every MCP tool name is a topic (`query`, `macro`, `fuzz_http`, ...), plus `getting-started`, `examples`, `template-syntax`, and `docs` |
| `section` | Print only this section. Accepts the heading text or its GitHub-style slug, case-insensitively; sub-sections are included |
| `--section <heading>` | Same as the second positional argument |

An unknown topic prints the list of valid topics; an unknown section prints that topic's heading outline. Both exit non-zero. Output is plain Markdown with no ANSI styling.

```bash
# List every topic with its estimated token size
yorishiro-proxy docs

# Print the whole macro reference
yorishiro-proxy docs macro

# Print a single section (heading text or slug)
yorishiro-proxy docs macro "Variable substitution syntax"
yorishiro-proxy docs macro --section variable-substitution-syntax
```

### `install`

Install and configure yorishiro-proxy components.

```bash
yorishiro-proxy install [target] [flags]
```

#### Targets

| Target | Description |
|--------|-------------|
| *(none)* | Install all components (MCP + CA) |
| `mcp` | Register MCP configuration only |
| `ca` | Generate CA certificate only |
| `playwright` | Configure Playwright integration (auto-detects browser, installs if needed, applies `--no-sandbox` in containers) |

!!! note "`skills` target removed"
    The previous `skills` target and `--skills-dir` flag were removed when vulnerability-verification playbooks migrated to [MCP Prompts](../guides/mcp-prompts.md). Playbooks ship inside the binary and are delivered to the MCP host over `prompts/list` — no filesystem install needed.

#### Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--interactive` | `false` | Enable interactive prompts (wizard mode) |
| `--user-scope` | `false` | Register in user scope (`~/.claude/settings.json`) instead of project (`.mcp.json`) |
| `--listen-addr` | `127.0.0.1:8080` | Proxy listen address |
| `--trust` | `false` | Register CA in OS trust store (requires sudo). Only valid with `ca` target or no target |
| `--ca-dir` | -- | CA certificate output directory |

#### Examples

```bash
# Install all components
yorishiro-proxy install

# Register MCP configuration only
yorishiro-proxy install mcp

# Generate CA and register in OS trust store
yorishiro-proxy install ca --trust

# Interactive wizard mode
yorishiro-proxy install --interactive

# Register MCP in user scope
yorishiro-proxy install mcp --user-scope
```

### `upgrade`

Check for and install updates from GitHub Releases.

```bash
yorishiro-proxy upgrade [flags]
```

| Flag | Default | Description |
|------|---------|-------------|
| `--check` | `false` | Check for updates without downloading |

#### Examples

```bash
# Check and install the latest version
yorishiro-proxy upgrade

# Check for updates without installing
yorishiro-proxy upgrade --check
```

After upgrading, the bundled MCP prompt playbooks ship inside the new binary — no separate update step is needed. The new prompt set becomes visible to MCP hosts on the next reconnect.

### `version`

Print the current version and exit.

```bash
yorishiro-proxy version
```

## Server flags reference

These flags apply to the `server` subcommand (or bare invocation without a subcommand).

| Flag | Env variable | Default | Description |
|------|-------------|---------|-------------|
| `-config` | `YP_CONFIG` | -- | JSON config file path for proxy defaults |
| `-db` | `YP_DB` | `~/.yorishiro-proxy/yorishiro.db` | SQLite database path or project name |
| `-ca-cert` | `YP_CA_CERT` | -- | CA certificate file path |
| `-ca-key` | `YP_CA_KEY` | -- | CA private key file path |
| `-ca-ephemeral` | `YP_CA_EPHEMERAL` | `false` | Use ephemeral in-memory CA |
| `-insecure` | `YP_INSECURE` | `false` | Skip upstream TLS certificate verification |
| `-tls-fingerprint` | `YP_TLS_FINGERPRINT` | `firefox` | TLS fingerprint profile: `firefox`, `chrome`, `safari`, `edge`, `random`, `none`. Case-insensitive; unknown values are rejected at startup |
| `-log-level` | `YP_LOG_LEVEL` | `info` | Log level: `debug`, `info`, `warn`, `error` |
| `-log-format` | `YP_LOG_FORMAT` | `text` | Log format: `text`, `json` |
| `-log-file` | `YP_LOG_FILE` | stderr | Log output file path |
| `-mcp-http-addr` | `YP_MCP_HTTP_ADDR` | `127.0.0.1:0` | Streamable HTTP listen address (also serves the Web UI); use `0` for OS-assigned port |
| `-mcp-http-token` | `YP_MCP_HTTP_TOKEN` | auto-generated | HTTP Bearer auth token |
| `-no-http-mcp` | `YP_NO_HTTP_MCP` | `false` | Disable HTTP MCP transport |
| `-stdio-mcp` | `YP_STDIO_MCP` | `false` | Enable stdio MCP transport in addition to HTTP MCP |
| `-open-browser` | `YP_OPEN_BROWSER` | `false` | Open WebUI in browser when HTTP MCP starts |
| `-ui-dir` | `YP_UI_DIR` | -- | Directory for WebUI static files, overrides embedded assets |
| `-export-dir` | `YP_EXPORT_DIR` | `~/.yorishiro-proxy/exports` | Directory that confines `manage` `export_flows` / `import_flows` file paths. Must already exist when set explicitly. See [Export directory](#export-directory-export-dir) |
| `-target-policy-file` | `YP_TARGET_POLICY_FILE` | -- | Target scope policy JSON file path. Reloadable with `SIGHUP` |
| `-safety-filter` | `YP_SAFETY_FILTER_ENABLED` | `false` | Enable SafetyFilter engine |
| `-max-concurrent-streams` | `YP_MAX_CONCURRENT_STREAMS` | `0` | HTTP/2 `SETTINGS_MAX_CONCURRENT_STREAMS` advertised to clients. `0` uses the H2 Layer default (`500`). Range `1..4294967295` (USK-862) |

### Environment variable naming

All flags accept a `YP_` prefixed environment variable as a fallback. To derive the variable name from a flag:

1. Remove the leading `-`
2. Replace hyphens with underscores
3. Uppercase everything
4. Add the `YP_` prefix

For example, `-log-level` becomes `YP_LOG_LEVEL`.

## Signals

On Unix, sending `SIGHUP` to the server re-reads the target scope policy and replaces the running Policy Layer without a restart:

```bash
kill -HUP <pid>
```

- The source is `-target-policy-file` if set, otherwise the `target_scope_policy` section of `-config`.
- The reload is validated exactly like a boot. An invalid file is rejected (logged) and the current policy is kept.
- Live connections that the new policy denies are closed. See [Target scope](../features/target-scope.md#reloading-the-policy-layer).
- The handler is installed only when one of those flags is set (log line `target scope policy reload enabled`). Without either flag, `SIGHUP` terminates the process as before.

!!! warning "Closing the terminal"
    When the reload handler is active, closing the terminal that launched the server sends `SIGHUP` and triggers a reload instead of stopping the server. Stop it with `SIGINT` / `SIGTERM` instead.

## Export directory (`-export-dir`)

All server-side file I/O performed by the [`manage`](../tools/manage.md) tool's `export_flows` and `import_flows` actions is confined to a single directory:

- Relative `output_path` / `input_path` values resolve against the export directory — not against your working directory or the server's.
- Absolute paths must lie inside the export directory; anything else is rejected.
- The default `~/.yorishiro-proxy/exports` is created on demand. An explicit `-export-dir` must already exist and be a directory.
- `query` with `resource: "status"` reports the resolved path as `export_dir`.

Use a dedicated directory: MCP callers can overwrite any file inside it. The server refuses to start when an operator-owned path — the config file, policy file, CA, database, log file, UI directory, plugin scripts, or TLS material — resolves inside the export directory.

```bash
yorishiro-proxy server -export-dir /srv/yorishiro/exports
```

## MCP transport modes

By default, the server starts **HTTP MCP** on a random loopback port. The address and authentication token are written to `~/.yorishiro-proxy/server.json` for automatic discovery by the `client` subcommand and other tools.

| Mode | Flag | Default | Description |
|------|------|---------|-------------|
| HTTP MCP | `-mcp-http-addr` | `127.0.0.1:0` (enabled) | Streamable HTTP transport + Web UI |
| Stdio MCP | `-stdio-mcp` | disabled | Stdin/stdout transport for direct MCP client integration |
| No HTTP MCP | `-no-http-mcp` | disabled | Disable HTTP MCP (useful with `-stdio-mcp` for stdio-only mode) |

To use stdio mode for direct integration with MCP clients like Claude Code:

```bash
# Stdio-only (like pre-v0.14.0 behavior)
yorishiro-proxy server -stdio-mcp -no-http-mcp
```

To use both transports simultaneously:

```bash
# HTTP + stdio (both active)
yorishiro-proxy server -stdio-mcp
```

## server.json

When HTTP MCP starts, the server writes its connection details to `~/.yorishiro-proxy/server.json`:

```json
[
  {
    "addr": "127.0.0.1:54321",
    "token": "auto-generated-token",
    "pid": 12345,
    "started_at": "2026-03-27T10:00:00Z"
  }
]
```

This file supports multiple concurrent server instances. Each server appends its entry on startup and removes it on shutdown. Stale entries (dead PIDs) are automatically cleaned up.

The `client` subcommand reads this file for automatic server discovery when no explicit connection flags are given.

## Database path resolution (`-db`)

The `-db` flag supports three resolution modes:

| Input | Resolution | Example |
|-------|-----------|---------|
| Project name (no extension, no path separator) | `~/.yorishiro-proxy/<name>.db` | `-db pentest-2026` |
| Absolute path | Used as-is | `-db /data/project.db` |
| Relative path with extension | CWD-relative | `-db ./data.db` |

When omitted, it defaults to `~/.yorishiro-proxy/yorishiro.db`.

Project names make it easy to maintain separate databases per engagement:

```bash
# Each project gets its own database in ~/.yorishiro-proxy/
yorishiro-proxy server -db client-audit
yorishiro-proxy server -db pentest-2026
yorishiro-proxy server -db webapp-review
```

Valid project name characters are: alphanumeric (`a-z`, `A-Z`, `0-9`), hyphen (`-`), underscore (`_`), and dot (`.`). Names must not start with a dot or contain `..`.

## CA certificate modes

yorishiro-proxy supports three CA modes for TLS interception:

### Auto-persist (default)

When neither `-ca-cert`/`-ca-key` nor `-ca-ephemeral` is specified, the CA is stored in `~/.yorishiro-proxy/ca/`. If the files exist, they are loaded; otherwise a new CA is generated and saved automatically.

### Explicit

Specify both `-ca-cert` and `-ca-key` to load a CA from custom file paths. Both flags must be provided together.

```bash
yorishiro-proxy server -ca-cert /path/to/ca.crt -ca-key /path/to/ca.key
```

### Ephemeral

Use `-ca-ephemeral` to generate an in-memory CA that is not persisted. A new CA is created on every startup. This cannot be combined with `-ca-cert`/`-ca-key`.

```bash
yorishiro-proxy server -ca-ephemeral
```

## Usage examples

```bash
# Default: HTTP MCP on random loopback port with auto-persist CA
yorishiro-proxy server

# Load proxy configuration from a JSON file
yorishiro-proxy server -config proxy.json

# Use a fixed HTTP MCP address
yorishiro-proxy server -mcp-http-addr 127.0.0.1:3000

# Open the Web UI in a browser on startup
yorishiro-proxy server -mcp-http-addr 127.0.0.1:3000 -open-browser

# Use a project-specific database
yorishiro-proxy server -db pentest-2026

# Use an absolute database path
yorishiro-proxy server -db /data/project.db

# Set project name via environment variable
YP_DB=client-audit yorishiro-proxy server

# Skip upstream TLS verification
yorishiro-proxy server -insecure

# Use the Chrome TLS fingerprint instead of the Firefox default
yorishiro-proxy server -tls-fingerprint chrome

# Disable fingerprint spoofing (standard Go crypto/tls)
yorishiro-proxy server -tls-fingerprint none

# Enable JSON-formatted logging to a file
yorishiro-proxy server -log-format json -log-file /var/log/yorishiro.log

# Enable SafetyFilter engine
yorishiro-proxy server -safety-filter

# Load target scope policy from a dedicated file
yorishiro-proxy server -target-policy-file policy.json

# Stdio MCP for direct MCP client integration (no HTTP)
yorishiro-proxy server -stdio-mcp -no-http-mcp

# Combine multiple options
yorishiro-proxy server -db pentest-2026 -mcp-http-addr 127.0.0.1:3000 -insecure -tls-fingerprint chrome
```

## Related pages

- [Config file](config-file.md) -- JSON config file reference
- [TLS](tls.md) -- TLS interception and certificate configuration
- [Upstream proxy](upstream-proxy.md) -- Proxy chaining configuration
- [Retention](retention.md) -- Flow retention and cleanup settings
