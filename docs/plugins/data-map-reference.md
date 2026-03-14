# Protocol data map reference

This page documents the data dictionary keys passed to plugin hook functions for each protocol. Each hook receives a `data` dictionary with protocol-specific keys that you can read or modify.

## HTTP / HTTPS

Hooks receive the following data keys for HTTP/1.x and HTTPS (MITM) traffic.

### Request hooks (`on_receive_from_client`, `on_before_send_to_server`)

| Key | Type | Description |
|-----|------|-------------|
| `protocol` | string | `"HTTP/1.x"` or `"HTTPS"` |
| `method` | string | HTTP method (GET, POST, etc.) |
| `url` | string | Full request URL |
| `scheme` | string | URL scheme (`"http"` or `"https"`) |
| `host` | string | Request host |
| `path` | string | URL path |
| `query` | string | Raw query string |
| `headers` | dict | HTTP headers as key to list-of-values mapping |
| `body` | bytes | Request body |
| `conn_info` | dict | Connection metadata (see [conn_info](#connection-info-conn_info)) |
| `ctx` | dict | Transaction context for cross-hook data sharing |

### Response hooks (`on_receive_from_server`, `on_before_send_to_client`)

| Key | Type | Description |
|-----|------|-------------|
| `protocol` | string | `"HTTP/1.x"` or `"HTTPS"` |
| `status_code` | int | HTTP response status code |
| `headers` | dict | HTTP headers as key to list-of-values mapping |
| `body` | bytes | Response body |
| `conn_info` | dict | Connection metadata |
| `request` | dict | Read-only request summary with `method`, `url`, `host` |
| `ctx` | dict | Transaction context |

### Header format

HTTP headers are represented as a dictionary where each key maps to a list of string values:

```python
# Reading headers
auth = data.get("headers", {}).get("Authorization", [])
if auth:
    print("Auth header: %s" % auth[0])

# Setting headers (single value)
headers = data.get("headers", {})
headers["X-Custom"] = "value"
data["headers"] = headers
```

## HTTP/2

HTTP/2 data maps are identical to HTTP/HTTPS, with the `protocol` value set to `"h2"`:

| Key | Value |
|-----|-------|
| `protocol` | `"h2"` |

All other fields are the same as [HTTP / HTTPS](#http-https).

## gRPC

gRPC plugins operate in **observe-only** mode. Only `action.CONTINUE` is allowed.

### Request hooks

| Key | Type | Description |
|-----|------|-------------|
| `protocol` | string | `"grpc"` |
| `method` | string | gRPC method path (e.g., `/package.Service/Method`) |
| `url` | string | Same as `method` for gRPC |
| `headers` | dict | gRPC metadata/headers |
| `body` | string | Serialized protobuf message |
| `conn_info` | dict | Connection metadata |
| `ctx` | dict | Transaction context |

### Response hooks

| Key | Type | Description |
|-----|------|-------------|
| `protocol` | string | `"grpc"` |
| `headers` | dict | gRPC response metadata/trailers |
| `body` | string | Serialized protobuf message |
| `conn_info` | dict | Connection metadata |
| `request` | dict | Read-only request summary |
| `ctx` | dict | Transaction context |

## WebSocket

| Key | Type | Description |
|-----|------|-------------|
| `protocol` | string | `"websocket"` |
| `opcode` | int | WebSocket frame opcode (1=text, 2=binary) |
| `payload` | string | Message payload |
| `is_text` | bool | `True` if the message is a text frame |
| `direction` | string | `"client_to_server"` or `"server_to_client"` |
| `conn_info` | dict | Connection metadata |

## TCP (raw)

| Key | Type | Description |
|-----|------|-------------|
| `protocol` | string | `"tcp"` |
| `data` | string | Raw TCP data |
| `direction` | string | `"client_to_server"` or `"server_to_client"` |
| `conn_info` | dict | Connection metadata |
| `forward_target` | string | Target address for TCP forwarding |

## SOCKS5

The `on_socks5_connect` hook is called when a SOCKS5 CONNECT tunnel is successfully established. It receives the following data:

| Key | Type | Description |
|-----|------|-------------|
| `event` | string | Always `"socks5_connect"` |
| `target_host` | string | Destination hostname (e.g., `"example.com"`) |
| `target_port` | int | Destination port (e.g., `443`) |
| `target` | string | Full destination address (e.g., `"example.com:443"`) |
| `auth_method` | string | Authentication method used: `"none"` or `"username_password"` |
| `auth_user` | string | Authenticated username (empty if `auth_method` is `"none"`) |
| `client_addr` | string | Remote address of the client |

Only `action.CONTINUE` is allowed; `DROP` and `RESPOND` are not supported.

Flows that pass through a SOCKS5 tunnel are recorded with protocol identifiers like `SOCKS5+HTTPS` or `SOCKS5+HTTP`, and include `socks5_target` and `socks5_auth_method` in their tags.

## Connection info (`conn_info`)

The `conn_info` dictionary is available in all protocols and contains network connection metadata:

| Key | Type | Description |
|-----|------|-------------|
| `client_addr` | string | Remote address of the client (e.g., `"192.168.1.100:54321"`) |
| `server_addr` | string | Resolved address of the upstream server |
| `tls_version` | string | Negotiated TLS version (e.g., `"TLS 1.3"`) |
| `tls_cipher` | string | Negotiated TLS cipher suite name |
| `tls_alpn` | string | Negotiated ALPN protocol (e.g., `"h2"`, `"http/1.1"`) |

For non-TLS connections, the TLS-related fields are empty strings.

## Related pages

- [Hook reference](hook-reference.md) -- When each hook is called and which actions are allowed
- [Writing plugins](writing-plugins.md) -- How to write plugins
- [Examples](examples.md) -- Ready-to-use plugin samples
