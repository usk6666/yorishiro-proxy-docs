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
  "tls_fingerprint": "chrome"
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

- The upstream proxy setting affects all outgoing connections from the proxy, including both HTTP and HTTPS (CONNECT) traffic.
- Changing the upstream proxy at runtime via `configure` applies to new connections only. Existing connections continue using their original route.
- Proxy credentials in the URL (e.g., `http://user:pass@proxy:3128`) are sent as `Proxy-Authorization` headers for HTTP proxies, or as SOCKS5 username/password authentication.
- The `configure` tool's response redacts proxy credentials for security.

## Related pages

- [CLI flags](cli-flags.md) -- All available CLI flags
- [Config file](config-file.md) -- Full config file reference
- [configure](../tools/configure.md) -- Runtime configuration tool reference
- [proxychains](../guides/proxychains.md) -- Using proxychains with yorishiro-proxy
