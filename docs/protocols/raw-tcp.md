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

### Structured ForwardConfig format

In addition to the legacy string format, you can use a ForwardConfig object to control protocol detection and TLS termination per port:

```json
// proxy_start
{
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

See [proxy_start](../tools/proxy-start.md#tcp_forwards) for the full ForwardConfig reference.

## Protocol detection on TCP forwards

When you configure a TCP forward with the ForwardConfig object format, the `protocol` field controls how the proxy handles the connection at L7.

### Auto detection (default)

When `protocol` is `"auto"` (or omitted), the proxy peeks at the initial bytes of the connection to determine the L7 protocol. The detection logic follows the same priority as the main listener:

1. If the bytes match the HTTP/2 connection preface, the connection is handled as HTTP/2
2. If the bytes match an HTTP method prefix, the connection is handled as HTTP/1.x
3. Otherwise, the connection falls through to raw TCP relay

This allows a single forwarded port to handle multiple protocols automatically.

### Explicit protocol

When `protocol` is set to a specific value (`"http"`, `"http2"`, `"grpc"`, `"websocket"`), the proxy delegates the connection directly to the corresponding protocol handler without peeking. This is useful when you know the upstream protocol in advance and want to skip detection overhead.

### Raw mode

When `protocol` is `"raw"`, the proxy performs no L7 parsing and relays the connection as opaque bytes. This is the same behavior as the legacy string format (`"3306": "db.example.com:3306"`).

## TLS MITM on TCP forwards

When `tls` is `true` in a ForwardConfig, the proxy terminates TLS on the forwarded port before applying protocol detection or relay. The proxy:

1. Accepts the inbound TLS connection, issuing a certificate using the target hostname
2. Decrypts the traffic
3. Applies L7 protocol handling (based on the `protocol` setting) to the decrypted stream
4. Forwards the plaintext to the upstream target

This enables inspection of TLS-wrapped services such as gRPC-over-TLS or HTTPS backends that are accessed via TCP forwarding rather than through the HTTP CONNECT proxy.

```json
// proxy_start
{
  "tcp_forwards": {
    "8443": {
      "target": "secure.example.com:443",
      "protocol": "auto",
      "tls": true
    }
  }
}
```

!!! note
    Setting `tls: true` with `protocol: "raw"` is valid -- TLS is terminated but the decrypted payload is relayed as raw bytes without L7 parsing.

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

- **No protocol awareness in raw mode** -- when `protocol` is `"raw"` (or using the legacy string format), the handler treats all traffic as opaque bytes with no protocol-specific parsing. Use ForwardConfig with `protocol: "auto"` or an explicit protocol to enable L7 parsing.
- **No TLS termination in raw mode** -- without `tls: true` in ForwardConfig, encrypted TCP connections pass through as raw bytes without decryption
- **Explicit mapping required** -- connections to unmapped ports are closed immediately
- **No target scope enforcement** -- the TCP handler does not apply target scope rules (the mapping itself acts as the scope)
- **No safety filter** -- since there is no HTTP-layer understanding in raw mode, safety filter rules do not apply

## Related pages

- [SOCKS5](socks5.md) -- SOCKS5 tunnels that may fall through to raw TCP relay
- [Plugin hook reference](../plugins/hook-reference.md) -- detailed hook documentation
- [proxy_start](../tools/proxy-start.md) -- configuring TCP forwards
