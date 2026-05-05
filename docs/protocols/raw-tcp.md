# Raw TCP

yorishiro-proxy includes a `bytechunk` Layer (`internal/layer/bytechunk/`) that wraps a `net.Conn` and yields one `RawMessage` envelope per `Read()` call. This is the identity Layer for raw TCP passthrough and TLS-terminate-only diagnostic mode (HTTP request smuggling).

L7 structured view: not applicable. L4 raw bytes: yes. The Layer is smuggling-safe: no accumulation or framing is applied, so chunk boundaries are determined by the OS TCP stack — the proxy observes wire reality without imposing its own segmentation.

## Fallback protocol

The connector's protocol detector applies the following priority order on the peeked bytes:

```
1. SOCKS5  (first byte = 0x05)
2. HTTP/2  (preface: "PRI * HTTP/2.0\r\n")
3. HTTP    (method prefix: GET, POST, PUT, etc.)
4. bytechunk Layer  (fallback for everything else)
```

If none of the specific protocols match, the ConnectionStack is built around the bytechunk Layer.

## TCP forwarding mappings

Raw TCP forwarding requires explicit configuration. Each mapping associates a local listen port with an upstream target address. Connections arriving on a port without a configured mapping are closed immediately.

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

When you configure a TCP forward with the ForwardConfig object format, the `protocol` field controls which Layer the connector wraps the connection in at L7.

### Auto detection (default)

When `protocol` is `"auto"` (or omitted), the connector peeks at the initial bytes of the connection to choose a Layer. The detection logic follows the same priority as the main listener:

1. If the bytes match the HTTP/2 connection preface, the connection is wrapped in the HTTP/2 Layer
2. If the bytes match an HTTP method prefix, the connection is wrapped in the HTTP/1.x Layer
3. Otherwise, the connection falls through to the `bytechunk` Layer

This allows a single forwarded port to handle multiple protocols automatically.

### Explicit protocol

When `protocol` is set to a specific value (`"http"`, `"http2"`, `"grpc"`, `"websocket"`), the connector wires the corresponding Layer directly without peeking. This is useful when you know the upstream protocol in advance and want to skip detection overhead.

### Raw mode

When `protocol` is `"raw"`, the connector wires the `bytechunk` Layer with no further L7 parsing. This is the same behavior as the legacy string format (`"3306": "db.example.com:3306"`).

## TLS MITM on TCP forwards

When `tls` is `true` in a ForwardConfig, the connector wires the `tlslayer/` Layer between the listener and the chosen inner Layer:

1. The TLS Layer accepts the inbound TLS connection, issuing a certificate using the target hostname
2. The decrypted byte stream feeds the inner Layer (chosen by `protocol`)
3. A separate upstream TLS connection is dialed and its cleartext stream feeds back through the inner Layer's `Send` direction

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

Each `Read()` from the underlying connection produces exactly one `RawMessage` envelope:

- **Protocol**: `raw`
- **Flow type**: `bidirectional`
- **Direction**: `send` (client to upstream) or `receive` (upstream to client)

`Envelope.Raw` carries the raw chunk bytes verbatim. Sequence is a monotonically increasing counter shared across both directions, preserving the interleaved ordering of the conversation.

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

The bytechunk Layer participates in the standard Pipeline. Plugin authors register hooks via the pluginv2 engine using the `(protocol, event, phase)` triple — for example `(raw, on_chunk, pre)` to observe each `RawMessage` envelope before intercept. See the [Plugin hook reference](../plugins/hook-reference.md) for the full surface.

Plugin errors are logged but do not interrupt the relay (fail-open behavior).

## Limitations

- **No protocol awareness in raw mode** -- when `protocol` is `"raw"` (or using the legacy string format), the bytechunk Layer treats all traffic as opaque bytes with no protocol-specific parsing. Use ForwardConfig with `protocol: "auto"` or an explicit protocol to enable L7 parsing.
- **No TLS termination in raw mode** -- without `tls: true` in ForwardConfig, encrypted TCP connections pass through as raw bytes without decryption
- **Explicit mapping required** -- connections to unmapped ports are closed immediately
- **No target scope enforcement** -- raw forwarding does not apply target scope rules (the mapping itself acts as the scope)
- **No safety filter** -- since there is no L7 understanding in raw mode, safety filter rules do not apply

## Related pages

- [SOCKS5](socks5.md) -- SOCKS5 tunnels that may fall through to raw TCP relay
- [Plugin hook reference](../plugins/hook-reference.md) -- detailed hook documentation
- [proxy_start](../tools/proxy-start.md) -- configuring TCP forwards
