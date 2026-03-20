# Config file

yorishiro-proxy can load proxy defaults from a JSON config file. The file format is identical to the `proxy_start` tool's input, so you can reuse the same JSON structure for both file-based configuration and runtime MCP tool calls.

## Loading a config file

Pass the config file path using the `-config` flag or `YP_CONFIG` environment variable:

```bash
yorishiro-proxy -config proxy.json
```

```bash
YP_CONFIG=proxy.json yorishiro-proxy
```

## File structure overview

The config file is a single JSON object with the following top-level sections:

```json
{
  "listen_addr": "127.0.0.1:8080",
  "upstream_proxy": "",
  "tls_fingerprint": "chrome",
  "tls_passthrough": [],
  "capture_scope": {},
  "intercept_rules": {},
  "auto_transform": [],
  "tcp_forwards": {},
  "client_cert": "",
  "client_key": "",
  "host_tls": {},
  "socks5_auth": "none",
  "socks5_username": "",
  "socks5_password": "",
  "target_scope_policy": {},
  "safety_filter": {},
  "plugins": [],
  "codec_plugins": []
}
```

All fields are optional. Only include the sections you need.

## Proxy core settings

### `listen_addr`

The TCP address the proxy listens on.

| Type | Default |
|------|---------|
| `string` | `"127.0.0.1:8080"` |

```json
{
  "listen_addr": "127.0.0.1:9090"
}
```

### `upstream_proxy`

Upstream proxy URL for proxy chaining. Supports HTTP and SOCKS5 proxy URLs.

| Type | Default |
|------|---------|
| `string` | `""` (direct connection) |

```json
{
  "upstream_proxy": "http://proxy.corp.example:3128"
}
```

### `tls_fingerprint`

TLS ClientHello fingerprint profile for upstream HTTPS connections. Uses uTLS to mimic a browser's TLS fingerprint, evading JA3/JA4-based bot detection.

| Type | Default | Valid values |
|------|---------|-------------|
| `string` | `""` (Go default TLS) | `chrome`, `firefox`, `safari`, `edge`, `random`, `none` |

```json
{
  "tls_fingerprint": "chrome"
}
```

### `tls_passthrough`

List of domain patterns that bypass TLS interception (MITM). Connections to matching hosts are forwarded without decryption.

| Type | Default |
|------|---------|
| `string[]` | `[]` |

Supported patterns:

- Exact match: `"example.com"`
- Wildcard: `"*.example.com"` (matches any subdomain)

```json
{
  "tls_passthrough": ["*.google.com", "*.googleapis.com"]
}
```

### `capture_scope`

Controls which requests are recorded to the flow store. This is passed as a raw JSON object to the proxy engine.

```json
{
  "capture_scope": {
    "include": ["*.target.com"],
    "exclude": ["*.static.target.com"]
  }
}
```

### `intercept_rules`

Configures request/response intercept rules. This is passed as a raw JSON object to the intercept engine.

```json
{
  "intercept_rules": {
    "request": true,
    "response": false
  }
}
```

### `auto_transform`

Configures auto-transform rules for automatic request/response modification. This is passed as a raw JSON array to the transform pipeline.

```json
{
  "auto_transform": [
    {
      "match": {"header": "Authorization"},
      "set_header": {"Authorization": "Bearer new-token"}
    }
  ]
}
```

### `tcp_forwards`

Maps local listen ports to upstream forwarding configurations. Each value can be a string (legacy format) or a ForwardConfig object with protocol detection and TLS options.

| Type | Default |
|------|---------|
| `object` | `{}` |

**String format (legacy)** -- forwards raw bytes without L7 parsing:

```json
{
  "tcp_forwards": {
    "5432": "db.internal:5432",
    "6379": "redis.internal:6379"
  }
}
```

**ForwardConfig object format** -- enables protocol detection and TLS MITM:

```json
{
  "tcp_forwards": {
    "50051": {
      "target": "api.internal:50051",
      "protocol": "grpc"
    },
    "8443": {
      "target": "backend.internal:443",
      "protocol": "auto",
      "tls": true
    },
    "5432": "db.internal:5432"
  }
}
```

ForwardConfig fields:

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `target` | `string` | (required) | Upstream `host:port` address |
| `protocol` | `string` | `"auto"` | L7 protocol: `auto`, `raw`, `http`, `http2`, `grpc`, `websocket` |
| `tls` | `bool` | `false` | Enable TLS MITM termination on this port |

Both formats can be mixed in the same object. Legacy string values are equivalent to `{"target": "host:port", "protocol": "raw"}`.

### SOCKS5 authentication

Configure SOCKS5 proxy authentication.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `socks5_auth` | `string` | `"none"` | Authentication method: `none` or `password` |
| `socks5_username` | `string` | `""` | Username for password authentication |
| `socks5_password` | `string` | `""` | Password for password authentication |

```json
{
  "socks5_auth": "password",
  "socks5_username": "user",
  "socks5_password": "secret"
}
```

## Client TLS (mTLS)

### Global client certificate

Configure a global mTLS client certificate for upstream connections. Both `client_cert` and `client_key` must be provided together.

| Field | Type | Description |
|-------|------|-------------|
| `client_cert` | `string` | Path to PEM-encoded client certificate |
| `client_key` | `string` | Path to PEM-encoded client private key |

```json
{
  "client_cert": "/path/to/client.crt",
  "client_key": "/path/to/client.key"
}
```

### Per-host TLS (`host_tls`)

Configure TLS settings for specific hosts. Supports per-host client certificates, custom CA bundles, and TLS verification control. Wildcard patterns (e.g., `*.example.com`) are supported.

Each entry supports:

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `client_cert` | `string` | -- | PEM-encoded client certificate path |
| `client_key` | `string` | -- | PEM-encoded client private key path |
| `tls_verify` | `bool\|null` | `null` (use global) | TLS verification: `true` = enforce, `false` = skip, `null` = global setting |
| `ca_bundle` | `string` | -- | PEM-encoded CA bundle file path |

```json
{
  "host_tls": {
    "api.internal.corp": {
      "client_cert": "/path/to/api-client.crt",
      "client_key": "/path/to/api-client.key"
    },
    "*.staging.example.com": {
      "tls_verify": false
    },
    "private-ca.example.com": {
      "ca_bundle": "/path/to/custom-ca.pem",
      "tls_verify": true
    }
  }
}
```

## SafetyFilter (`safety_filter`)

The SafetyFilter engine blocks or masks destructive payloads. This is a Policy Layer setting -- once loaded from a config file, it cannot be modified at runtime via MCP tools.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enabled` | `bool` | `false` | Enable the SafetyFilter engine |
| `input` | `object` | -- | Input (request) filter rules |
| `output` | `object` | -- | Output (response) filter rules |

### Input filter rules

Input rules inspect outgoing requests and can block destructive payloads.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `action` | `string` | `"block"` | Default action: `block` or `log_only` |
| `rules` | `array` | `[]` | List of filter rules |

### Output filter rules

Output rules inspect incoming responses and can mask sensitive data.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `action` | `string` | `"mask"` | Default action: `mask` or `log_only` |
| `rules` | `array` | `[]` | List of filter rules |

### Rule definition

Each rule is either a **preset reference** or a **custom rule**. The two are mutually exclusive.

| Field | Type | Description |
|-------|------|-------------|
| `preset` | `string` | Built-in preset name (e.g., `destructive-sql`). Mutually exclusive with `pattern` |
| `id` | `string` | Unique identifier. Required for custom rules |
| `name` | `string` | Human-readable label. Optional |
| `pattern` | `string` | Regular expression pattern. Mutually exclusive with `preset` |
| `targets` | `string[]` | Parts to inspect. Input: `body`, `url`, `query`, `header`, `headers`. Output: `body`, `header:<name>`, `headers` |
| `replacement` | `string` | Override replacement string for output preset rules |

### SafetyFilter example

```json
{
  "safety_filter": {
    "enabled": true,
    "input": {
      "action": "block",
      "rules": [
        {"preset": "destructive-sql"},
        {"preset": "destructive-os-command"},
        {
          "id": "block-drop-api",
          "name": "Block dangerous API calls",
          "pattern": "DELETE\\s+/api/v[0-9]+/(users|accounts)",
          "targets": ["url"]
        }
      ]
    },
    "output": {
      "action": "mask",
      "rules": [
        {"preset": "credit-card"},
        {"preset": "email"},
        {
          "id": "mask-internal-ip",
          "name": "Mask internal IPs",
          "pattern": "10\\.\\d+\\.\\d+\\.\\d+",
          "targets": ["body"]
        }
      ]
    }
  }
}
```

## Target scope policy (`target_scope_policy`)

Target scope policy rules control which network targets the proxy is allowed to access. These rules are part of the Policy Layer and are immutable at runtime.

!!! note
    When a dedicated policy file is specified via `-target-policy-file` / `YP_TARGET_POLICY_FILE`, the `target_scope_policy` section in the config file is ignored.

### Structure

| Field | Type | Description |
|-------|------|-------------|
| `allows` | `array` | Rules that permit network access. When non-empty, only matching targets are allowed |
| `denies` | `array` | Rules that block network access. Deny rules take precedence over allow rules |
| `rate_limits` | `object` | Rate limiting configuration |
| `budget` | `object` | Diagnostic session budget |

### Target rule fields

All non-empty fields must match for a rule to apply (AND logic).

| Field | Type | Description |
|-------|------|-------------|
| `hostname` | `string` | Target hostname (case-insensitive). Supports `*.example.com` wildcard |
| `ports` | `int[]` | Restrict to specific port numbers. Empty = all ports |
| `path_prefix` | `string` | Match the beginning of the URL path (case-sensitive). Empty = all paths |
| `schemes` | `string[]` | Restrict to specific URL schemes (e.g., `http`, `https`). Empty = all schemes |

### Rate limits (`rate_limits`)

Rate limiting for AI agent request throttling. These are Policy Layer limits that the Agent Layer cannot exceed.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `max_requests_per_second` | `float` | `0` (unlimited) | Global rate limit (requests per second) |
| `max_requests_per_host_per_second` | `float` | `0` (unlimited) | Per-host rate limit (requests per second) |

### Budget (`budget`)

Diagnostic session budgets to prevent runaway AI agents.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `max_total_requests` | `int` | `0` (unlimited) | Maximum number of requests in the session |
| `max_duration` | `string` | `""` (unlimited) | Maximum session duration (e.g., `"30m"`, `"1h"`) |

### Target scope policy example

```json
{
  "target_scope_policy": {
    "allows": [
      {"hostname": "*.target.com"},
      {"hostname": "api.example.com", "ports": [443], "schemes": ["https"]}
    ],
    "denies": [
      {"hostname": "*.internal.corp"},
      {"hostname": "*.target.com", "path_prefix": "/admin"}
    ],
    "rate_limits": {
      "max_requests_per_second": 50,
      "max_requests_per_host_per_second": 10
    },
    "budget": {
      "max_total_requests": 10000,
      "max_duration": "1h"
    }
  }
}
```

You can also store target scope policy in a separate file and load it with `-target-policy-file`. The file format is the same as the `target_scope_policy` section:

```json
{
  "allows": [
    {"hostname": "*.target.com"}
  ],
  "denies": [
    {"hostname": "*.internal.corp"}
  ],
  "rate_limits": {
    "max_requests_per_second": 50
  }
}
```

## Plugins (`plugins`)

Configure Starlark-based plugins for the proxy pipeline. Each entry specifies a script path and configuration. Plugins are executed in order.

```json
{
  "plugins": [
    {
      "path": "/path/to/plugin.star",
      "target": "http",
      "hooks": ["on_request", "on_response"],
      "on_error": "log"
    }
  ]
}
```

For details on writing plugins, see the [plugin development guide](../plugins/overview.md).

## Codec plugins (`codec_plugins`)

Configure Starlark-based custom codec plugins. Each entry specifies a path to a Starlark codec file or a directory containing `*.star` codec files. Codec plugins are registered alongside built-in codecs.

```json
{
  "codec_plugins": [
    {"path": "/path/to/custom-codec.star"},
    {"path": "/path/to/codec-directory/"}
  ]
}
```

For details on codec plugins, see the [codec plugins guide](../plugins/codec-plugins.md).

## Configuration examples

### Minimal configuration

A minimal config file that sets just the listen address:

```json
{
  "listen_addr": "127.0.0.1:8080"
}
```

### Standard assessment configuration

A typical configuration for a vulnerability assessment engagement:

```json
{
  "listen_addr": "127.0.0.1:8080",
  "tls_fingerprint": "chrome",
  "tls_passthrough": ["*.google.com", "*.googleapis.com"],
  "target_scope_policy": {
    "allows": [
      {"hostname": "*.target.com"}
    ],
    "denies": [
      {"hostname": "*.internal.corp"}
    ],
    "rate_limits": {
      "max_requests_per_second": 50,
      "max_requests_per_host_per_second": 10
    },
    "budget": {
      "max_total_requests": 10000,
      "max_duration": "2h"
    }
  },
  "safety_filter": {
    "enabled": true,
    "input": {
      "action": "block",
      "rules": [
        {"preset": "destructive-sql"},
        {"preset": "destructive-os-command"}
      ]
    },
    "output": {
      "action": "mask",
      "rules": [
        {"preset": "credit-card"},
        {"preset": "email"}
      ]
    }
  }
}
```

### Full configuration

A comprehensive configuration demonstrating all available sections:

```json
{
  "listen_addr": "127.0.0.1:8080",
  "upstream_proxy": "http://proxy.corp.example:3128",
  "tls_fingerprint": "chrome",
  "tls_passthrough": ["*.google.com"],
  "capture_scope": {
    "include": ["*.target.com"],
    "exclude": ["*.static.target.com"]
  },
  "intercept_rules": {
    "request": true,
    "response": false
  },
  "auto_transform": [
    {
      "match": {"header": "Authorization"},
      "set_header": {"Authorization": "Bearer updated-token"}
    }
  ],
  "tcp_forwards": {
    "5432": "db.internal:5432",
    "50051": {
      "target": "api.internal:50051",
      "protocol": "grpc"
    }
  },
  "client_cert": "/path/to/client.crt",
  "client_key": "/path/to/client.key",
  "host_tls": {
    "api.internal.corp": {
      "client_cert": "/path/to/api-client.crt",
      "client_key": "/path/to/api-client.key"
    },
    "*.staging.example.com": {
      "tls_verify": false
    }
  },
  "socks5_auth": "password",
  "socks5_username": "user",
  "socks5_password": "secret",
  "target_scope_policy": {
    "allows": [
      {"hostname": "*.target.com"}
    ],
    "denies": [
      {"hostname": "*.internal.corp"}
    ],
    "rate_limits": {
      "max_requests_per_second": 50,
      "max_requests_per_host_per_second": 10
    },
    "budget": {
      "max_total_requests": 10000,
      "max_duration": "1h"
    }
  },
  "safety_filter": {
    "enabled": true,
    "input": {
      "action": "block",
      "rules": [
        {"preset": "destructive-sql"},
        {"preset": "destructive-os-command"}
      ]
    },
    "output": {
      "action": "mask",
      "rules": [
        {"preset": "credit-card"},
        {"preset": "email"}
      ]
    }
  },
  "plugins": [
    {
      "path": "/path/to/plugin.star",
      "target": "http",
      "hooks": ["on_request", "on_response"],
      "on_error": "log"
    }
  ],
  "codec_plugins": [
    {"path": "/path/to/custom-codec.star"}
  ]
}
```

## Related pages

- [CLI flags](cli-flags.md) -- Command-line flags reference
- [TLS](tls.md) -- TLS interception and certificate configuration
- [Upstream proxy](upstream-proxy.md) -- Proxy chaining configuration
- [Retention](retention.md) -- Flow retention and cleanup settings
- [SafetyFilter](../features/safety-filter.md) -- SafetyFilter feature details
- [Target scope](../features/target-scope.md) -- Target scope feature details
- [Rate limits & budgets](../features/rate-limits.md) -- Rate limiting feature details
