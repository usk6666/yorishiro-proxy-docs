# proxychains integration

yorishiro-proxy includes a built-in SOCKS5 listener that integrates with [proxychains](https://github.com/haadcode/proxychains) and [proxychains-ng](https://github.com/rofl0r/proxychains-ng). This allows you to route traffic from arbitrary TCP-based tools (nmap, curl, sqlmap, etc.) through the proxy for capture and analysis.

## How the SOCKS5 flow works

When a tool runs through proxychains, the traffic flows through these layers:

```
Tool (nmap, curl, etc.)
  -> proxychains (intercepts TCP connect calls)
    -> yorishiro-proxy SOCKS5 listener (127.0.0.1:1080)
      -> Protocol detection (HTTP, HTTPS, Raw TCP)
        -> Target server
```

HTTPS traffic intercepted through SOCKS5 is recorded with the `SOCKS5+HTTPS` protocol identifier. Plaintext HTTP traffic is recorded as `SOCKS5+HTTP`.

## Step 1: Start the SOCKS5 listener

Start a dedicated SOCKS5 listener on port 1080:

```json
// proxy_start
{
  "listen_addr": "127.0.0.1:1080",
  "protocols": ["SOCKS5"]
}
```

You can run the SOCKS5 listener alongside an HTTP proxy listener. yorishiro-proxy supports multiple listeners simultaneously:

```json
// proxy_start
{
  "listen_addr": "127.0.0.1:8080"
}
```

```json
// proxy_start
{
  "listen_addr": "127.0.0.1:1080",
  "protocols": ["SOCKS5"]
}
```

Now you have an HTTP proxy on port 8080 and a SOCKS5 proxy on port 1080, both feeding into the same flow store.

### With capture scope

Limit which traffic is recorded:

```json
// proxy_start
{
  "listen_addr": "127.0.0.1:1080",
  "protocols": ["SOCKS5"],
  "capture_scope": {
    "includes": [{"hostname": "target.example.com"}]
  }
}
```

## Step 2: Configure proxychains

Edit your proxychains configuration file. The file is typically located at `/etc/proxychains.conf` or `~/.proxychains/proxychains.conf`.

### Basic configuration

```ini
# proxychains.conf
strict_chain
proxy_dns
tcp_read_time_out 15000
tcp_connect_time_out 8000

[ProxyList]
socks5 127.0.0.1 1080
```

### Key settings

- **`strict_chain`** -- use proxies in the order listed (recommended for single-proxy setups)
- **`proxy_dns`** -- resolve DNS through the proxy (important for capturing the full request)
- Timeouts should be generous enough for MITM processing overhead

## Step 3: Route traffic through proxychains

Prefix any command with `proxychains` to route its traffic through the proxy:

```bash
# HTTP requests
proxychains curl https://target.example.com/api/v1/users

# Port scanning (TCP connect scan only)
proxychains nmap -sT -Pn -p 80,443,8080 target.example.com

# Web tools
proxychains wget https://target.example.com/robots.txt

# Custom scripts
proxychains python3 my-scanner.py
```

!!! warning
    proxychains only works with TCP connect operations. Tools that use raw sockets (e.g., `nmap -sS` SYN scan) bypass proxychains. Use `-sT` for TCP connect scans.

## Step 4: Query SOCKS5 flows

### List all SOCKS5 flows

```json
// query
{
  "resource": "flows",
  "filter": {"protocol": "SOCKS5+HTTPS"}
}
```

### Filter by host

```json
// query
{
  "resource": "flows",
  "filter": {"host": "target.example.com"}
}
```

### View flow details

```json
// query
{"resource": "flow", "id": "<flow-id>"}
```

The flow details include the full request/response as well as SOCKS5 metadata.

## SOCKS5 authentication

For environments where you want to restrict who can use the proxy, enable SOCKS5 username/password authentication (RFC 1929).

### Enable authentication

```json
// configure
{
  "socks5_auth": {
    "method": "password",
    "username": "proxyuser",
    "password": "proxypass"
  }
}
```

You can also set authentication when starting the listener:

```json
// proxy_start
{
  "listen_addr": "127.0.0.1:1080",
  "protocols": ["SOCKS5"],
  "socks5_auth": "password",
  "socks5_username": "proxyuser",
  "socks5_password": "proxypass"
}
```

### Update proxychains configuration

Add credentials to the proxy line:

```ini
[ProxyList]
socks5 127.0.0.1 1080 proxyuser proxypass
```

### Disable authentication

To remove authentication:

```json
// configure
{
  "socks5_auth": {
    "method": "none"
  }
}
```

## Verifying the SOCKS5+HTTPS flow

To confirm that traffic is being intercepted and recorded correctly, run a simple test:

### 1. Start the SOCKS5 listener

```json
// proxy_start
{
  "listen_addr": "127.0.0.1:1080",
  "protocols": ["SOCKS5"]
}
```

### 2. Send a request through proxychains

```bash
proxychains curl -s https://httpbin.org/get
```

### 3. Check the captured flow

```json
// query
{
  "resource": "flows",
  "filter": {"url_pattern": "httpbin.org"},
  "limit": 5
}
```

You should see a flow with protocol `SOCKS5+HTTPS` containing the full request and response.

### 4. Inspect the details

```json
// query
{"resource": "flow", "id": "<flow-id>"}
```

The response includes the HTTP request headers, body, status code, and response content, exactly as if it had been captured through the HTTP proxy.

## Tool-specific tips

### curl

```bash
# Simple GET
proxychains curl https://api.target.com/users

# POST with JSON body
proxychains curl -X POST https://api.target.com/users \
  -H "Content-Type: application/json" \
  -d '{"name": "test"}'
```

### nmap

```bash
# TCP connect scan (the only scan type that works with proxychains)
proxychains nmap -sT -Pn -p 80,443,8080,8443 target.example.com

# Service version detection
proxychains nmap -sT -Pn -sV -p 80,443 target.example.com
```

!!! note
    Always use `-Pn` (skip host discovery) with proxychains, because ICMP ping does not work through SOCKS5.

### sqlmap

```bash
proxychains sqlmap -u "https://target.example.com/api/search?q=test" --batch
```

### Python scripts

```bash
# Any Python script using standard socket operations
proxychains python3 exploit.py --target target.example.com
```

## Combining with upstream proxy

You can chain proxychains through yorishiro-proxy and then to a corporate proxy:

```json
// proxy_start
{
  "listen_addr": "127.0.0.1:1080",
  "protocols": ["SOCKS5"],
  "upstream_proxy": "http://corporate-proxy:3128"
}
```

This routes all SOCKS5 traffic through the corporate proxy after interception.

## Related pages

- [SOCKS5](../protocols/socks5.md) -- SOCKS5 protocol reference
- [Upstream proxy](../configuration/upstream-proxy.md) -- chaining through upstream proxies
- [proxy_start tool](../tools/proxy-start.md) -- proxy_start tool reference
- [configure tool](../tools/configure.md) -- configure tool reference
