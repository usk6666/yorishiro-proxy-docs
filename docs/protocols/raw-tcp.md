# Raw TCP

yorishiro-proxy includes a raw TCP handler that acts as a fallback for connections that do not match any specific protocol handler. It relays data bidirectionally between the client and a configured upstream target, recording all traffic chunks to the flow store.

## Fallback protocol

The raw TCP handler's `Detect` method always returns `true`, which means it matches any connection. Because of this, it must be registered **last** in the protocol detector's priority order:

```
1. SOCKS5  (first byte = 0x05)
2. HTTP/2  (preface: "PRI * HTTP/2.0\r\n")
3. HTTP    (method prefix: GET, POST, PUT, etc.)
4. TCP     (always matches -- fallback)
```

If none of the specific protocol handlers match the peeked bytes, the connection falls through to the raw TCP handler.

## TCP forwarding mappings

The raw TCP handler requires explicit forwarding configuration. Each mapping associates a local listen port with an upstream target address. Connections arriving on a port without a configured mapping are closed immediately.

Configure TCP forwarding via the `proxy_start` tool:

```json
// proxy_start
{
  "listen_addr": "127.0.0.1:3306",
  "tcp_forwards": {
    "3306": "db.example.com:3306"
  }
}
```

This forwards all traffic arriving on port 3306 to `db.example.com:3306`, recording everything along the way.

### Multiple forwards

You can configure multiple forwarding targets:

```json
// proxy_start
{
  "listen_addr": "127.0.0.1:3306",
  "tcp_forwards": {
    "3306": "db.example.com:3306",
    "6379": "redis.example.com:6379",
    "5432": "postgres.example.com:5432"
  }
}
```

Forward mappings can be updated dynamically -- new mappings are merged with existing ones, so previously configured forwards remain active.

## Data recording

All data flowing through the relay is recorded as flow messages:

- **Protocol**: `TCP`
- **Flow type**: `bidirectional`
- **Direction**: `send` (client to upstream) or `receive` (upstream to client)

Each data chunk read from either side is recorded as a separate message with:

| Field | Description |
|-------|-------------|
| `raw_bytes` | The raw data bytes (copied to avoid aliasing) |
| `sequence` | Monotonically increasing sequence number |
| `direction` | `send` or `receive` |
| `metadata.chunk_size` | Size of the data chunk in bytes |

The relay uses a 32 KB buffer for each direction, so messages may be up to 32 KB in size. The sequence counter is shared across both directions, preserving interleaved ordering.

## Connection lifecycle

```
1. Client connects to proxy on a configured port
2. Proxy resolves the forwarding target from the local port
3. Proxy dials the upstream target (30-second timeout)
4. Flow record is created (State: "active")
5. Bidirectional relay starts with recording
6. Connection closes (either side EOF, error, or context cancellation)
7. Flow record is updated (State: "complete" or "error")
```

If the upstream dial fails, the flow is not created and the client connection is closed.

## Plugin hooks

The TCP handler dispatches plugin hooks per data chunk:

| Direction | Hooks dispatched |
|-----------|-----------------|
| Client to upstream | `on_receive_from_client` then `on_before_send_to_server` |
| Upstream to client | `on_receive_from_server` then `on_before_send_to_client` |

Plugin capabilities:

- **Drop chunks**: returning `ActionDrop` silently skips the chunk (not forwarded, not recorded)
- **Modify data**: returning modified `data` in the result updates the chunk before forwarding
- **Size limits**: if a plugin-modified chunk exceeds the maximum TCP plugin chunk size, the modification is discarded and the original data is preserved

Plugin hook data includes:

| Key | Description |
|-----|-------------|
| `protocol` | `"tcp"` |
| `data` | The raw chunk bytes |
| `direction` | `"client_to_server"` or `"server_to_client"` |
| `forward_target` | The upstream target address |
| `conn_info` | Client and server address information |

A per-chunk transaction context is created so plugins can pass state between the receive and send hooks for the same chunk.

Plugin errors are logged but do not interrupt the relay (fail-open behavior).

## Limitations

- **No protocol awareness** -- the raw TCP handler treats all traffic as opaque bytes with no protocol-specific parsing
- **No TLS termination** -- encrypted TCP connections pass through as raw bytes without decryption
- **Explicit mapping required** -- connections to unmapped ports are closed immediately
- **No target scope enforcement** -- the TCP handler does not apply target scope rules (the mapping itself acts as the scope)
- **No safety filter** -- since there is no HTTP-layer understanding, safety filter rules do not apply

## Related pages

- [SOCKS5](socks5.md) -- SOCKS5 tunnels that may fall through to raw TCP relay
- [Plugin hook reference](../plugins/hook-reference.md) -- detailed hook documentation
- [proxy_start](../tools/proxy-start.md) -- configuring TCP forwards
