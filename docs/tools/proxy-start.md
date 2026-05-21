# proxy_start

Start a proxy listener with optional configuration. The proxy listens on the specified address and begins intercepting HTTP/HTTPS/SOCKS5 traffic. You can run multiple listeners simultaneously by assigning each a unique name.

!!! warning "Reset semantics -- not a delta-apply tool"
    Each `proxy_start` call **fully resets** all prior session settings to the values supplied in the request. Fields omitted from the request revert to their defaults -- including `capture_scope`, `tls_passthrough`, `intercept_rules`, `auto_transform`, `tcp_forwards`, SOCKS5 auth, TLS fingerprint, and connection/timeout limits. Use `proxy_start` for first-time startup or session initialization. For in-session partial updates to a running listener (adding a single rule, changing one timeout, etc.), use [`configure`](configure.md) instead.

    See [Common misuse: proxy_start as a partial-update tool](#common-misuse-proxy_start-as-a-partial-update-tool) below for the anti-pattern and the recommended initialize-then-configure workflow.

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
| `tcp_forwards` | object | No | `{}` | TCP forwarding map (`port` -> target string or ForwardConfig object) |
| `socks5_auth` | string | No | `"none"` | SOCKS5 authentication method (`none` or `password`) |
| `socks5_username` | string | No | | Username for SOCKS5 password auth |
| `socks5_password` | string | No | | Password for SOCKS5 password auth |
| `tls_fingerprint` | string | No | `"chrome"` | TLS ClientHello fingerprint profile |
| `client_cert` | string | No | | PEM client certificate path for mTLS |
| `client_key` | string | No | | PEM client private key path for mTLS |
| `max_connections` | integer | No | `128` | Maximum concurrent connections (1--100000) |
| `max_concurrent_streams` | integer | No | `500` | HTTP/2 `SETTINGS_MAX_CONCURRENT_STREAMS` advertised to clients (1--65535) |
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

### tcp_forwards

Maps local listen ports to upstream forwarding configurations. Each entry maps a port number (string key) to either a string or a ForwardConfig object.

**String format (legacy)** -- the value is an upstream `host:port` address. Traffic is relayed as raw bytes with no L7 parsing:

```json
"3306": "db.example.com:3306"
```

**ForwardConfig object format** -- provides control over protocol detection and TLS termination:

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `target` | string | Yes | -- | Upstream address in `host:port` format |
| `protocol` | string | No | `"auto"` | Expected protocol for L7 parsing |
| `tls` | boolean | No | `false` | Enable client-side TLS MITM termination on the forwarded listen port |
| `upstream_tls` | boolean | No | `false` | Wrap the upstream-dial connection in TLS (USK-911 / USK-916). Independent of `tls` -- target's scheme/port are advisory only |
| `upstream_insecure_skip_verify` | boolean | No | inherit `-insecure` | Per-entry override for upstream TLS certificate verification when `upstream_tls=true`. Set `true` to skip verification on this entry, `false` to enforce, or omit to inherit the global `Config.InsecureSkipVerify` (USK-918) |

**Protocol values:**

| Value | Description |
|-------|-------------|
| `"auto"` | Peek-based detection -- the proxy inspects initial bytes to determine the L7 protocol |
| `"raw"` | No L7 parsing -- relay as opaque bytes (same behavior as the legacy string format) |
| `"http"` | Parse traffic as HTTP/1.x |
| `"http2"` | Parse traffic as HTTP/2 |
| `"grpc"` | Parse traffic as gRPC (over HTTP/2) |
| `"websocket"` | Operator-declared expectation that the request initiates an RFC 6455 Upgrade |
| `"sse"` | Operator-declared expectation that the upstream returns a `text/event-stream` response (USK-913) |

**TLS modes (`tls` × `upstream_tls` matrix):**

`tls` and `upstream_tls` are independent: `tls` controls the client-direction handshake on the listen port; `upstream_tls` controls the upstream-direction handshake to `target`. The four combinations cover the realistic deployment shapes:

| `tls` | `upstream_tls` | Client wire | Upstream wire | Typical use case |
|-------|----------------|-------------|---------------|------------------|
| `false` | `false` | plaintext | plaintext | Raw L4/L7 forwarding (default) |
| `true` | `false` | TLS | plaintext | TLS termination only (downstream offload) |
| `false` | `true` | plaintext | TLS | Plaintext client -> TLS-only upstream |
| `true` | `true` | TLS | TLS | Full MITM: terminate then re-encrypt upstream |

When `tls=true`, the proxy issues a certificate using the target hostname and applies L7 parsing to the decrypted traffic. When `upstream_tls=true`, the proxy uses the global TLS fingerprint, SNI derived from `target`, and the configured verification / client-cert settings -- or the per-entry `upstream_insecure_skip_verify` override.

!!! note
    Setting `tls: true` with `protocol: "raw"` is valid but unusual -- TLS is terminated, yet the decrypted payload is relayed as raw bytes without L7 parsing.

Both formats can be mixed in the same `tcp_forwards` object.

!!! note "Port collision with listen_addr"
    Any `tcp_forwards` port that matches the `listen_addr` port is rejected fail-fast before binding (port `0` on either side is exempt because the kernel assigns distinct ephemeral ports). The check protects against silently-broken listeners that would otherwise be registered but fail to bind.

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

## Common misuse: proxy_start as a partial-update tool

`proxy_start` is **not** a delta-apply tool. Calling it repeatedly to "add one more intercept rule" wipes every other setting that was previously established for the listener.

### Anti-pattern (do NOT do this)

```json
// proxy_start -- step 1: set up the session
{
  "listen_addr": "127.0.0.1:8080",
  "capture_scope": {"includes": [{"hostname": "api.target.com"}]},
  "tls_passthrough": ["*.googleapis.com"]
}
```

```json
// proxy_start -- step 2 (WRONG): trying to add an intercept rule.
// This RESETS capture_scope back to "capture all" and clears tls_passthrough,
// because both fields are omitted from this second request.
{
  "listen_addr": "127.0.0.1:8080",
  "intercept_rules": [
    {
      "id": "r1",
      "enabled": true,
      "direction": "request",
      "conditions": {"host_pattern": "api\\.target\\.com"}
    }
  ]
}
```

### Recommended pattern: initialize with `proxy_start`, update with `configure`

```json
// proxy_start -- one-shot initialization
{
  "listen_addr": "127.0.0.1:8080",
  "capture_scope": {"includes": [{"hostname": "api.target.com"}]},
  "tls_passthrough": ["*.googleapis.com"]
}
```

```json
// configure -- in-session update: add an intercept rule without touching the rest
{
  "operation": "merge",
  "intercept_rules": {
    "add": [
      {
        "id": "r1",
        "enabled": true,
        "direction": "request",
        "conditions": {"host_pattern": "api\\.target\\.com"}
      }
    ]
  }
}
```

See [`configure`](configure.md) for the full merge / replace surface (add, remove, enable, disable per rule ID, or full-section replacement).

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

### Start with TCP forwarding (string format)

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

### Start with TCP forwarding (ForwardConfig with protocol detection)

```json
// proxy_start
{
  "listen_addr": "127.0.0.1:8080",
  "tcp_forwards": {
    "50051": {
      "target": "api.example.com:50051",
      "protocol": "grpc"
    },
    "8443": {
      "target": "secure.example.com:443",
      "protocol": "http2",
      "tls": true
    },
    "3306": "db.example.com:3306"
  }
}
```

### TCP forward with upstream TLS and per-entry verification override

```json
// proxy_start
{
  "listen_addr": "127.0.0.1:8080",
  "tcp_forwards": {
    "50051": {
      "target": "grpcbin.local:50051",
      "protocol": "grpc",
      "upstream_tls": true,
      "upstream_insecure_skip_verify": true
    }
  }
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
  "listen_addr": "127.0.0.1:1080"
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
