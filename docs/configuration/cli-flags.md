# CLI flags

yorishiro-proxy accepts command-line flags, environment variables, and a JSON config file to control its behavior. This page documents all available flags, subcommands, and the priority rules that determine which value takes effect.

## Priority order

When the same setting is specified in multiple places, the following priority applies:

1. **CLI flag** (highest priority)
2. **Environment variable** (`YP_` prefix)
3. **Config file** (`-config`)
4. **Default value** (lowest priority)

## Flags reference

| Flag | Env variable | Default | Description |
|------|-------------|---------|-------------|
| `-config` | `YP_CONFIG` | -- | JSON config file path for proxy defaults |
| `-db` | `YP_DB` | `~/.yorishiro-proxy/yorishiro.db` | SQLite database path or project name |
| `-ca-cert` | `YP_CA_CERT` | -- | CA certificate file path |
| `-ca-key` | `YP_CA_KEY` | -- | CA private key file path |
| `-ca-ephemeral` | `YP_CA_EPHEMERAL` | `false` | Use ephemeral in-memory CA |
| `-insecure` | `YP_INSECURE` | `false` | Skip upstream TLS certificate verification |
| `-tls-fingerprint` | `YP_TLS_FINGERPRINT` | -- | TLS fingerprint profile: `chrome`, `firefox`, `safari`, `edge`, `random`, `none` |
| `-log-level` | `YP_LOG_LEVEL` | `info` | Log level: `debug`, `info`, `warn`, `error` |
| `-log-format` | `YP_LOG_FORMAT` | `text` | Log format: `text`, `json` |
| `-log-file` | `YP_LOG_FILE` | stderr | Log output file path |
| `-mcp-http-addr` | `YP_MCP_HTTP_ADDR` | -- | Streamable HTTP listen address (also serves the Web UI) |
| `-mcp-http-token` | `YP_MCP_HTTP_TOKEN` | auto-generated | HTTP Bearer auth token |
| `-ui-dir` | `YP_UI_DIR` | -- | Directory for WebUI static files, overrides embedded assets |
| `-no-open-browser` | `YP_NO_OPEN_BROWSER` | `false` | Disable auto-opening WebUI in browser |
| `-target-policy-file` | `YP_TARGET_POLICY_FILE` | -- | Target scope policy JSON file path |
| `-safety-filter` | `YP_SAFETY_FILTER_ENABLED` | `false` | Enable SafetyFilter engine |

### Environment variable naming

All flags accept a `YP_` prefixed environment variable as a fallback. To derive the variable name from a flag:

1. Remove the leading `-`
2. Replace hyphens with underscores
3. Uppercase everything
4. Add the `YP_` prefix

For example, `-log-level` becomes `YP_LOG_LEVEL`.

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
yorishiro-proxy -db client-audit
yorishiro-proxy -db pentest-2026
yorishiro-proxy -db webapp-review
```

Valid project name characters are: alphanumeric (`a-z`, `A-Z`, `0-9`), hyphen (`-`), underscore (`_`), and dot (`.`). Names must not start with a dot or contain `..`.

## CA certificate modes

yorishiro-proxy supports three CA modes for TLS interception:

### Auto-persist (default)

When neither `-ca-cert`/`-ca-key` nor `-ca-ephemeral` is specified, the CA is stored in `~/.yorishiro-proxy/ca/`. If the files exist, they are loaded; otherwise a new CA is generated and saved automatically.

### Explicit

Specify both `-ca-cert` and `-ca-key` to load a CA from custom file paths. Both flags must be provided together.

```bash
yorishiro-proxy -ca-cert /path/to/ca.crt -ca-key /path/to/ca.key
```

### Ephemeral

Use `-ca-ephemeral` to generate an in-memory CA that is not persisted. A new CA is created on every startup. This cannot be combined with `-ca-cert`/`-ca-key`.

```bash
yorishiro-proxy -ca-ephemeral
```

## Subcommands

yorishiro-proxy provides three subcommands: `install`, `upgrade`, and `version`.

### `install`

Install and configure yorishiro-proxy components.

```bash
yorishiro-proxy install [target] [flags]
```

#### Targets

| Target | Description |
|--------|-------------|
| *(none)* | Install all components (MCP + CA + Skills) |
| `mcp` | Register MCP configuration only |
| `ca` | Generate CA certificate only |
| `skills` | Install Claude Code skills only |
| `playwright` | Configure Playwright integration (auto-detects browser, installs if needed, applies `--no-sandbox` in containers) |

#### Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--interactive` | `false` | Enable interactive prompts (wizard mode) |
| `--user-scope` | `false` | Register in user scope (`~/.claude/settings.json`) instead of project (`.mcp.json`) |
| `--listen-addr` | `127.0.0.1:8080` | Proxy listen address |
| `--trust` | `false` | Register CA in OS trust store (requires sudo). Only valid with `ca` target or no target |
| `--ca-dir` | -- | CA certificate output directory |
| `--skills-dir` | -- | Skills installation directory |

#### Examples

```bash
# Install all components
yorishiro-proxy install

# Register MCP configuration only
yorishiro-proxy install mcp

# Generate CA and register in OS trust store
yorishiro-proxy install ca --trust

# Install skills only
yorishiro-proxy install skills

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

After upgrading, you may want to update skills:

```bash
yorishiro-proxy install skills
```

### `version`

Print the current version and exit.

```bash
yorishiro-proxy version
```

## Usage examples

```bash
# Default: MCP stdio mode with auto-persist CA
yorishiro-proxy

# Load proxy configuration from a JSON file
yorishiro-proxy -config proxy.json

# Enable Streamable HTTP transport + Web UI
yorishiro-proxy -mcp-http-addr 127.0.0.1:3000

# Use a project-specific database
yorishiro-proxy -db pentest-2026

# Use an absolute database path
yorishiro-proxy -db /data/project.db

# Set project name via environment variable
YP_DB=client-audit yorishiro-proxy

# Skip upstream TLS verification
yorishiro-proxy -insecure

# Skip upstream TLS verification via environment variable
YP_INSECURE=true yorishiro-proxy

# Use Firefox TLS fingerprint
yorishiro-proxy -tls-fingerprint firefox

# Enable JSON-formatted logging to a file
yorishiro-proxy -log-format json -log-file /var/log/yorishiro.log

# Enable SafetyFilter engine
yorishiro-proxy -safety-filter

# Load target scope policy from a dedicated file
yorishiro-proxy -target-policy-file policy.json

# Combine multiple options
yorishiro-proxy -db pentest-2026 -mcp-http-addr 127.0.0.1:3000 -insecure -tls-fingerprint chrome
```

## Related pages

- [Config file](config-file.md) -- JSON config file reference
- [TLS](tls.md) -- TLS interception and certificate configuration
- [Upstream proxy](upstream-proxy.md) -- Proxy chaining configuration
- [Retention](retention.md) -- Flow retention and cleanup settings
