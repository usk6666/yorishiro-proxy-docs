# HTTP/2

yorishiro-proxy supports HTTP/2 in both cleartext (h2c) and TLS (h2) modes. Each HTTP/2 stream is recorded as an individual flow, providing per-request visibility even when multiple requests share a single connection.

## h2c (cleartext HTTP/2)

Cleartext HTTP/2 connections are detected by the HTTP/2 connection preface -- the string `PRI * HTTP/2.0\r\n` that clients send at the start of an h2c connection.

When the proxy detects this preface in the peeked bytes, the connection is dispatched directly to the HTTP/2 handler without any TLS involvement.

```json
// proxy_start
{
  "listen_addr": "127.0.0.1:8080"
}
```

Clients that support h2c can connect directly:

```bash
curl --http2-prior-knowledge http://127.0.0.1:8080/api/endpoint
```

## h2 (HTTP/2 over TLS)

HTTP/2 over TLS is handled through the HTTPS MITM tunnel. During the TLS handshake with the client, the proxy advertises both `h2` and `http/1.1` via ALPN (Application-Layer Protocol Negotiation):

1. Client sends `CONNECT example.com:443` to the proxy
2. Proxy responds with `200 Connection Established`
3. Client initiates TLS handshake, proposing `h2` and `http/1.1` in ALPN
4. If `h2` is negotiated, the proxy delegates to the HTTP/2 handler via `HandleH2`
5. If `http/1.1` is negotiated, the connection stays with the HTTP/1.x handler

This means the protocol is automatically selected based on what the client and proxy negotiate -- no manual configuration is needed.

## Stream-level flow recording

Unlike HTTP/1.x where each connection processes requests sequentially, HTTP/2 multiplexes multiple streams over a single connection. The proxy records each stream as a separate flow:

- Each stream gets its own flow with `FlowType: "unary"`
- Stream-level request and response messages are recorded independently
- Concurrent streams on the same connection do not interfere with each other

The handler uses Go's `http2.Server.ServeConn` internally, which dispatches each stream to a separate goroutine. All in-flight stream goroutines are tracked and waited on before the connection is closed, ensuring complete flow recording.

## Request processing pipeline

Each HTTP/2 stream follows this processing pipeline:

1. **Read request body** -- the full body is read and buffered
2. **Resolve scheme and host** -- determines `https` (for h2) or `http` (for h2c) and sets the target host
3. **Target scope check** -- enforces which destinations are allowed
4. **Rate limit check** -- enforces per-host rate limits
5. **Safety filter** -- checks request body and URL against safety rules
6. **Plugin hooks** -- dispatches `on_receive_from_client`
7. **Build outbound request** -- creates the upstream request with cleaned headers
8. **Intercept check** -- pauses for AI agent review if matching rules exist
9. **Plugin hooks** -- dispatches `on_before_send_to_server`
10. **Forward upstream** -- sends the request and reads the response
11. **Response intercept** -- allows modification of the response
12. **Plugin hooks** -- dispatches response hooks
13. **Output filter** -- masks sensitive data before sending to client
14. **Write response** -- sends the response back to the client
15. **Record flow** -- stores the complete request/response

## Differences from HTTP/1.x

| Aspect | HTTP/1.x | HTTP/2 |
|--------|----------|--------|
| Multiplexing | Sequential (one request at a time per connection) | Concurrent streams on a single connection |
| Flow recording | One flow per request | One flow per stream |
| Protocol detection | HTTP method prefix (`GET `, `POST `, etc.) | Connection preface (`PRI * HTTP/2.0`) or ALPN `h2` |
| Header format | Text-based | Binary (HPACK compressed) |
| Hop-by-hop headers | `Connection`, `Keep-Alive`, `Proxy-Connection`, `Transfer-Encoding`, `Upgrade`, `Te`, `Trailer`, `Proxy-Authenticate`, `Proxy-Authorization` | `Connection`, `Keep-Alive`, `Proxy-Connection`, `Transfer-Encoding`, `Upgrade` |

## gRPC detection

When the HTTP/2 handler detects a `Content-Type: application/grpc` (or `application/grpc+proto`, `application/grpc+json`, etc.) on a stream, it delegates flow recording to the [gRPC handler](grpc.md). The stream is still proxied by the HTTP/2 handler, but the recorded flow includes gRPC-specific metadata (service, method, grpc-status).

## Upstream connection

The proxy uses Go's `http.Transport` with `ForceAttemptHTTP2: true` for upstream connections. This ensures that the proxy negotiates HTTP/2 with the upstream server when possible, maintaining protocol parity between client and server.

When a custom TLS transport is configured (e.g., uTLS fingerprint spoofing), the upstream TLS connection uses the configured transport instead of Go's default `crypto/tls`.

## Limitations

- **No server push** -- HTTP/2 server push is not supported by the proxy
- **No h2c upgrade from HTTP/1.1** -- the proxy only supports h2c via the connection preface (`PRI`), not the HTTP/1.1 Upgrade mechanism
- **Body size limits** -- request and response bodies are limited to the configured maximum body size
- **No CONNECT-UDP** -- only TCP-based HTTP/2 is supported

## Related pages

- [HTTP/1.x](http.md) -- plaintext HTTP handling
- [HTTPS MITM](https-mitm.md) -- TLS interception and ALPN negotiation
- [gRPC](grpc.md) -- gRPC protocol handling over HTTP/2
- [Intercept](../features/intercept.md) -- pausing and modifying HTTP/2 streams
