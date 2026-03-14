# Hook function reference

This page documents all hook functions available to yorishiro-proxy Starlark plugins, including when each hook is called, what data it receives, and which actions are permitted.

## Data hooks

Data hooks are called during request/response processing. They participate in transaction context sharing via the `ctx` key.

### on_receive_from_client

Called after TargetScope evaluation, before Intercept. The plugin receives the client's request data.

- **Allowed actions**: `CONTINUE`, `DROP`, `RESPOND`
- **Data**: Protocol-specific request data (see [Data map reference](data-map-reference.md))

This is the only hook that supports `DROP` and `RESPOND` actions. Use it to filter, block, or mock responses.

```python
def on_receive_from_client(data):
    url = data.get("url", "")
    if "/blocked" in url:
        return {"action": action.DROP}
    return {"action": action.CONTINUE}
```

### on_before_send_to_server

Called after Transform, before Recording. The plugin receives the request about to be sent upstream. Modifications made here affect the actual request sent to the server.

- **Allowed actions**: `CONTINUE` only
- **Data**: Protocol-specific request data

```python
def on_before_send_to_server(data):
    headers = data.get("headers", {})
    headers["X-Forwarded-By"] = "yorishiro-proxy"
    data["headers"] = headers
    return {"action": action.CONTINUE, "data": data}
```

### on_receive_from_server

Called after receiving the server response, before Transform. The plugin receives the raw server response.

- **Allowed actions**: `CONTINUE` only
- **Data**: Protocol-specific response data

```python
def on_receive_from_server(data):
    status = data.get("status_code", 0)
    if status >= 500:
        print("Server error: %d" % status)
    return {"action": action.CONTINUE}
```

### on_before_send_to_client

Called after Transform, before Recording. The plugin receives the response about to be sent to the client.

- **Allowed actions**: `CONTINUE` only
- **Data**: Protocol-specific response data

```python
def on_before_send_to_client(data):
    headers = data.get("headers", {})
    headers["X-Plugin-Processed"] = "true"
    data["headers"] = headers
    return {"action": action.CONTINUE, "data": data}
```

## Lifecycle hooks

Lifecycle hooks are called during connection lifecycle events. They do **not** participate in transaction context sharing (no `ctx` key).

### on_connect

Called when a new TCP connection is accepted.

- **Allowed actions**: `CONTINUE` only
- **Data**: Connection metadata

```python
def on_connect(data):
    conn_info = data.get("conn_info", {})
    client = conn_info.get("client_addr", "unknown")
    print("New connection from: %s" % client)
    return {"action": action.CONTINUE}
```

### on_tls_handshake

Called after a TLS handshake completes.

- **Allowed actions**: `CONTINUE` only
- **Data**: Connection metadata with TLS information

```python
def on_tls_handshake(data):
    conn_info = data.get("conn_info", {})
    version = conn_info.get("tls_version", "unknown")
    cipher = conn_info.get("tls_cipher", "unknown")
    print("TLS handshake: version=%s cipher=%s" % (version, cipher))
    return {"action": action.CONTINUE}
```

### on_disconnect

Called when a connection is closed.

- **Allowed actions**: `CONTINUE` only
- **Data**: Connection metadata

```python
def on_disconnect(data):
    conn_info = data.get("conn_info", {})
    client = conn_info.get("client_addr", "unknown")
    print("Disconnected: %s" % client)
    return {"action": action.CONTINUE}
```

### on_socks5_connect

Called after a SOCKS5 CONNECT tunnel is successfully established. See [SOCKS5 data map](data-map-reference.md#socks5) for the full list of data keys.

- **Allowed actions**: `CONTINUE` only
- **Data**: SOCKS5 tunnel metadata

```python
def on_socks5_connect(data):
    target = data.get("target", "unknown")
    auth = data.get("auth_method", "none")
    print("SOCKS5 tunnel to %s (auth: %s)" % (target, auth))
    return {"action": action.CONTINUE}
```

## Hook call order in the proxy pipeline

The data hooks are called in a specific order within the proxy pipeline:

```
Client request arrives
  -> TargetScope evaluation
  -> on_receive_from_client    (can DROP / RESPOND)
  -> Intercept
  -> Transform
  -> on_before_send_to_server  (CONTINUE only)
  -> Recording
  -> Forward to server

Server response arrives
  -> on_receive_from_server    (CONTINUE only)
  -> Transform
  -> on_before_send_to_client  (CONTINUE only)
  -> Recording
  -> Forward to client
```

## Actions summary

| Action | Allowed hooks | Behavior |
|--------|--------------|----------|
| `action.CONTINUE` | All hooks | Continue processing with (optionally modified) data |
| `action.DROP` | `on_receive_from_client` only | Silently drop the connection |
| `action.RESPOND` | `on_receive_from_client` only (HTTP/HTTPS) | Send a custom response to the client |

## Plugin chain behavior

When multiple plugins are registered for the same hook:

1. Plugins are called in registration order
2. Each plugin's modifications are visible to subsequent plugins
3. If a plugin returns `DROP` or `RESPOND`, the chain stops immediately
4. If a plugin errors with `on_error: "skip"`, it is skipped and the chain continues
5. If a plugin errors with `on_error: "abort"`, the chain stops and the error is returned

## Related pages

- [Data map reference](data-map-reference.md) -- Protocol-specific data keys for each hook
- [Writing plugins](writing-plugins.md) -- How to write plugins
- [Examples](examples.md) -- Ready-to-use plugin samples
