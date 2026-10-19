# Upstream proxy

yorishiro-proxy can chain through an upstream HTTP or SOCKS5 proxy, routing all outgoing traffic through it. This is useful when you need to access targets behind a corporate proxy, tunnel traffic through a VPN, or combine multiple proxy tools.

## Supported protocols

| Protocol | URL format | Description |
|----------|-----------|-------------|
| HTTP proxy | `http://host:port` | Standard HTTP CONNECT proxy |
| SOCKS5 proxy | `socks5://host:port` | SOCKS5 proxy with optional authentication |

## Config file

Set the `upstream_proxy` field in your config file to route all proxy traffic through an upstream proxy:

### HTTP proxy

```json
{
  "upstream_proxy": "http://proxy.corp.example:3128"
}
```

### HTTP proxy with authentication

```json
{
  "upstream_proxy": "http://user:password@proxy.corp.example:3128"
}
```

### SOCKS5 proxy

```json
{
  "upstream_proxy": "socks5://socks-proxy.example:1080"
}
```

### SOCKS5 proxy with authentication

```json
{
  "upstream_proxy": "socks5://user:password@socks-proxy.example:1080"
}
```

## IP rotation

Instead of a single static URL, you can give a listener a URL **template** that is re-expanded according to a rotation policy. This is designed for residential or rotating proxy gateways that select the exit IP from a session token in the username.

```json
{
  "upstream_proxy": {
    "url_template": "http://user-session-§__nonce§:password@gw.residential.example:8000",
    "rotation": {"policy": "per_connection"}
  }
}
```

The template uses the macro `§var§` syntax. On the live data path, the only available variable is `§__nonce§`, which expands to a fresh UUID each time the template is resolved.

| `rotation.policy` | A new URL is minted... |
|-------------------|------------------------|
| `per_request` | at every outbound TCP dial |
| `per_connection` | once per inbound client connection |
| `per_target_host` | once per upstream host for the listener |
| `sticky` | once for the listener's lifetime |

Rules for the object form:

- Exactly one of `url` or `url_template` must be set.
- `rotation` is required with `url_template` and not allowed with `url`.
- The template must expand to an `http://` or `socks5://` URL with an explicit port. It is probe-expanded when the config is applied, so mistakes fail at startup (or in the tool call), not at the first dial.

The same object is accepted by the `proxy_start` and `configure` tools. `query` with `resource: "config"` and the `configure` response report a rotating listener through `upstream_proxy_template` (credentials redacted) and `upstream_proxy_rotation_policy` instead of `upstream_proxy`.

### Per-iteration rotation in resend and fuzz

[`resend_http`](../tools/resend-http.md) and [`fuzz_http`](../tools/fuzz-http.md) accept their own `upstream_proxy.url_template` override, independent of the listener setting. In addition to `§__nonce§`, these templates can use `§__iteration§` (the zero-based variant index). `fuzz_http` re-expands the template for every variant, so each variant can exit from a distinct IP.

```json
// fuzz_http
{
  "flow_id": "abc123",
  "positions": [
    {"path": "raw_query", "payloads": ["id=1", "id=2", "id=3"]}
  ],
  "upstream_proxy": {
    "url_template": "http://user-session-§__nonce§:password@gw.residential.example:8000"
  }
}
```

## Runtime configuration via MCP

You can set or change the upstream proxy at runtime using the `configure` tool. This takes effect immediately for new connections.

### Enable proxy chaining

```json
// configure
{
  "upstream_proxy": "http://proxy.corp.example:3128"
}
```

### Switch to a SOCKS5 proxy

```json
// configure
{
  "upstream_proxy": "socks5://socks-proxy.example:1080"
}
```

### Enable rotation on a listener

```json
// configure
{
  "name": "default",
  "upstream_proxy": {
    "url_template": "socks5://user-§__nonce§:pass@gw.residential.example:1080",
    "rotation": {"policy": "per_target_host"}
  }
}
```

### Disable proxy chaining

Set the value to an empty string to return to direct connections:

```json
// configure
{
  "upstream_proxy": ""
}
```

## proxy_start with upstream proxy

You can also specify the upstream proxy when starting a listener:

```json
// proxy_start
{
  "listen_addr": "127.0.0.1:8080",
  "upstream_proxy": "http://proxy.corp.example:3128"
}
```

## Configuration examples

### Corporate proxy with TLS fingerprinting

Route traffic through a corporate proxy while using browser TLS fingerprinting:

```json
{
  "listen_addr": "127.0.0.1:8080",
  "upstream_proxy": "http://proxy.corp.example:3128",
  "tls_fingerprint": "firefox"
}
```

### SOCKS5 tunnel with target scope

Chain through a SOCKS5 proxy and restrict the proxy to specific targets:

```json
{
  "listen_addr": "127.0.0.1:8080",
  "upstream_proxy": "socks5://127.0.0.1:9050",
  "target_scope_policy": {
    "allows": [
      {"hostname": "*.target.com"}
    ]
  }
}
```

### Upstream proxy with TLS passthrough

Combine proxy chaining with TLS passthrough to skip MITM for specific domains:

```json
{
  "listen_addr": "127.0.0.1:8080",
  "upstream_proxy": "http://proxy.corp.example:3128",
  "tls_passthrough": ["*.google.com", "*.googleapis.com"]
}
```

## Behavior notes

- The upstream proxy setting is **per-listener** (USK-826). Each `proxy_start` call carries its own `upstream_proxy`; one listener can dial direct while another chains through a corporate proxy, and a third rotates through a residential gateway.
- The upstream proxy setting affects all outgoing connections from the listener, including both HTTP and HTTPS (CONNECT) traffic and dial-time SOCKS5 routing for raw/TCP forwards (USK-797).
- Changing the upstream proxy at runtime via `configure` applies to new connections only. Existing connections continue using their original route.
- Proxy credentials in the URL (e.g., `http://user:pass@proxy:3128`) are sent as `Proxy-Authorization` headers for HTTP proxies, or as SOCKS5 username/password authentication (RFC 1929). The SOCKS5 password authenticator is wired into the live data path so credentials work for both control and live dispatch (USK-797).
- The `configure` tool's response redacts proxy credentials for security.

## Related pages

- [CLI flags](cli-flags.md) -- All available CLI flags
- [Config file](config-file.md) -- Full config file reference
- [configure](../tools/configure.md) -- Runtime configuration tool reference
- [proxychains](../guides/proxychains.md) -- Using proxychains with yorishiro-proxy
