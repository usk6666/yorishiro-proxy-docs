# proxy_start

Start a proxy listener with optional configuration. The proxy listens on the specified address and begins intercepting HTTP/HTTPS/SOCKS5 traffic. You can run multiple listeners simultaneously by assigning each a unique name.

## Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `name` | string | No | `"default"` | Listener name for multi-listener support |
| `listen_addr` | string | No | `"127.0.0.1:8080"` | TCP address to listen on (loopback only) |
| `upstream_proxy` | string | No | direct | Upstream proxy URL (`http://host:port` or `socks5://host:port`) |
| `capture_scope` | object | No | capture all | Controls which requests are recorded |
| `tls_passthrough` | string[] | No | `[]` | Domain patterns that bypass TLS interception |
| `intercept_rules` | array | No | `[]` | Rules for intercepting requests/responses |
| `auto_transform` | array | No | `[]` | Rules for automatic request/response modification |
| `tcp_forwards` | object | No | `{}` | Raw TCP forwarding map (`port` -> `upstream host:port`) |
| `protocols` | string[] | No | all enabled | Enabled protocols for detection |
| `socks5_auth` | string | No | `"none"` | SOCKS5 authentication method (`none` or `password`) |
| `socks5_username` | string | No | | Username for SOCKS5 password auth |
| `socks5_password` | string | No | | Password for SOCKS5 password auth |
| `tls_fingerprint` | string | No | `"chrome"` | TLS ClientHello fingerprint profile |
| `client_cert` | string | No | | PEM client certificate path for mTLS |
| `client_key` | string | No | | PEM client private key path for mTLS |
| `max_connections` | integer | No | `128` | Maximum concurrent connections (1--100000) |
| `peek_timeout_ms` | integer | No | `30000` | Protocol detection timeout in ms (100--600000) |
| `request_timeout_ms` | integer | No | `60000` | HTTP request header read timeout in ms (100--600000) |

### capture_scope

Controls which requests are recorded to the flow store.

- **includes** (array of scope rules) -- only matching requests are captured. If empty, all requests match.
- **excludes** (array of scope rules) -- matching requests are excluded. Takes precedence over includes.

Each scope rule has:

| Field | Type | Description |
|-------|------|-------------|
| `hostname` | string | Hostname pattern (exact or `*.example.com` wildcard) |
| `url_prefix` | string | URL path prefix (e.g. `"/api/"`) |
| `method` | string | HTTP method (e.g. `"GET"`, `"POST"`) |

At least one field must be set per rule.

### intercept_rules

Each intercept rule has:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | Unique rule identifier |
| `enabled` | boolean | Yes | Whether the rule is active |
| `direction` | string | Yes | `"request"`, `"response"`, or `"both"` |
| `conditions` | object | Yes | Matching criteria (AND logic) |

Conditions:

| Field | Type | Description |
|-------|------|-------------|
| `host_pattern` | string | Regex for hostname matching (port excluded) |
| `path_pattern` | string | Regex for URL path matching |
| `methods` | string[] | HTTP method whitelist (case-insensitive) |
| `header_match` | object | Maps header names to regex patterns (AND logic) |

Multiple rules use OR logic -- a request/response is intercepted if any enabled rule matches.

### protocols

Valid values: `"HTTP/1.x"`, `"HTTPS"`, `"WebSocket"`, `"HTTP/2"`, `"gRPC"`, `"SOCKS5"`, `"TCP"`.

### tls_fingerprint

| Value | Description |
|-------|-------------|
| `"chrome"` | Mimic Chrome browser TLS fingerprint (default) |
| `"firefox"` | Mimic Firefox browser TLS fingerprint |
| `"safari"` | Mimic Safari browser TLS fingerprint |
| `"edge"` | Mimic Edge browser TLS fingerprint |
| `"random"` | Random browser fingerprint per connection |
| `"none"` | Standard Go crypto/tls (no fingerprint mimicry) |

## Response

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Listener name |
| `listen_addr` | string | Actual address the proxy is listening on |
| `status` | string | Proxy state (`"running"`) |
| `tcp_forwards` | object | Configured TCP forwarding map (if any) |
| `protocols` | string[] | Enabled protocols (if explicitly configured) |

## Examples

### Start with defaults

```json
// proxy_start
{}
```

### Start on a custom port

```json
// proxy_start
{
  "listen_addr": "127.0.0.1:9090"
}
```

### Start with capture scope and TLS passthrough

```json
// proxy_start
{
  "listen_addr": "127.0.0.1:8080",
  "capture_scope": {
    "includes": [
      {"hostname": "api.target.com"},
      {"hostname": "*.target.com", "url_prefix": "/api/"}
    ],
    "excludes": [
      {"hostname": "static.target.com"}
    ]
  },
  "tls_passthrough": ["*.googleapis.com", "accounts.google.com"]
}
```

### Start with intercept rules

```json
// proxy_start
{
  "intercept_rules": [
    {
      "id": "target-host",
      "enabled": true,
      "direction": "request",
      "conditions": {
        "host_pattern": "httpbin\\.org"
      }
    },
    {
      "id": "admin-api",
      "enabled": true,
      "direction": "request",
      "conditions": {
        "host_pattern": "api\\.target\\.com",
        "path_pattern": "/api/admin.*",
        "methods": ["POST", "PUT", "DELETE"],
        "header_match": {"Content-Type": "application/json"}
      }
    }
  ]
}
```

### Start with TCP forwarding

```json
// proxy_start
{
  "listen_addr": "127.0.0.1:8080",
  "tcp_forwards": {
    "3306": "db.example.com:3306",
    "6379": "redis.example.com:6379"
  }
}
```

### Start with specific protocols

```json
// proxy_start
{
  "listen_addr": "127.0.0.1:8080",
  "protocols": ["HTTP/1.x", "HTTPS", "gRPC"]
}
```

### Start with SOCKS5 password authentication

```json
// proxy_start
{
  "listen_addr": "127.0.0.1:8080",
  "socks5_auth": "password",
  "socks5_username": "proxyuser",
  "socks5_password": "proxypass"
}
```

### Start multiple listeners

```json
// proxy_start
{
  "name": "http-listener",
  "listen_addr": "127.0.0.1:8080"
}
```

```json
// proxy_start
{
  "name": "socks-listener",
  "listen_addr": "127.0.0.1:1080",
  "protocols": ["SOCKS5"]
}
```

### Start with TLS fingerprint

```json
// proxy_start
{
  "listen_addr": "127.0.0.1:8080",
  "tls_fingerprint": "firefox"
}
```

## Related pages

- [proxy_stop](proxy-stop.md) -- Stop the proxy
- [configure](configure.md) -- Modify settings at runtime
- [Quick setup](../getting-started/quick-setup.md) -- Getting started guide
- [Architecture](../concepts/architecture.md) -- System design
