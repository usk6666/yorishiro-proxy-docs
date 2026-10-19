# HTTP/1.x

yorishiro-proxy operates as a forward proxy for plaintext HTTP/1.x traffic. The HTTP/1.x Layer (`internal/layer/http1/`) parses HTTP/1.x wire bytes into `HTTPMessage` Envelopes for each request and response, preserving wire fidelity (header case, order, OWS, anomalies).

L7 structured view: yes. L4 raw bytes: yes. The Layer uses a custom parser; `net/http` is not used in the data path.

## How detection works

When a new TCP connection arrives, the connector peeks at the first bytes and checks for known HTTP method prefixes:

- `GET `, `POST `, `PUT `, `DELETE `, `HEAD `, `OPTIONS `, `PATCH `, `CONNECT `

If the peeked bytes match any of these prefixes, the ConnectionStack is built around the HTTP/1.x Layer. The `CONNECT` method triggers the [HTTPS MITM](https-mitm.md) path instead.

## Forward proxy mode

yorishiro-proxy functions as a standard HTTP forward proxy. Clients send requests with absolute-form URLs (e.g., `GET http://example.com/path HTTP/1.1`), and the proxy forwards them to the upstream server.

```
Client  ──HTTP Request──>  yorishiro-proxy  ──HTTP Request──>  Upstream Server
        <──HTTP Response──                  <──HTTP Response──
```

Configure your client to use the proxy:

```bash
export http_proxy=http://127.0.0.1:8080
curl http://httpbin.org/get
```

## Request/response recording

Every request and response flowing through the proxy is recorded as a **flow** in the flow store. Each flow message carries an `HTTPMessage` Envelope with:

- **Request** (direction: `send`): method, URL, headers, body, raw wire bytes
- **Response** (direction: `receive`): status code, headers, body, raw wire bytes

The Layer preserves the raw wire-format bytes verbatim on `Envelope.Raw` before any normalization, which is useful for detecting [HTTP request smuggling](https://portswigger.net/web-security/request-smuggling) patterns.

### Wire-form request path

The recorded request path is kept exactly as it appeared on the wire: percent-encoding is never decoded, and the query is split off at the first `?`. A request for `/a%2Fb?x=1` is recorded, queried, exported, and resent with the path `/a%2Fb`. JSON surfaces (`query`, JSONL export, HAR) `%XX`-escape only bytes that are not valid UTF-8.

Security checks match either form: a target scope `path_prefix` or SafetyFilter `url` rule matches the wire path or its percent-decoded form, so `/%61dmin` does not slip past a `/admin` deny.

### Body size limits

Request and response bodies are recorded up to a configurable maximum size. The Layer's parser surfaces oversize bodies as `*layer.StreamError`.

### Keep-alive support

The HTTP/1.x Layer processes multiple requests on the same connection in a loop, supporting HTTP/1.1 keep-alive. When the client sends `Connection: close`, the loop terminates after that request.

## Hop-by-hop headers

The proxy removes hop-by-hop headers before forwarding requests upstream, as specified by HTTP/1.1:

- `Connection`, `Keep-Alive`, `Proxy-Authenticate`, `Proxy-Authorization`
- `Proxy-Connection`, `Te`, `Trailer`, `Transfer-Encoding`, `Upgrade`

## WebSocket upgrade

When the proxy detects a WebSocket upgrade request (`Connection: Upgrade` + `Upgrade: websocket`), the HTTP/1.x Layer detaches the underlying stream and the [WebSocket Layer](websocket.md) (`internal/layer/ws/`) takes over for frame-level recording. See the WebSocket page for details.

## Request timeout

`request_timeout_ms` (default `60000`) bounds how long a client may take to send each request **head** — the request line and headers. The same bound applies to the CONNECT request and to the client-side TLS MITM handshake, on the forward proxy, CONNECT, and `tcp_forwards` paths. This protects against Slowloris-style attacks where a client trickles bytes to hold a connection slot. The request body is not covered.

On a keep-alive connection, the proxy waits up to `max(120s, request_timeout_ms)` for the first byte of the next request, then applies `request_timeout_ms` to the rest of the head. An idle keep-alive connection is closed after that wait.

## Request size limits

Limits are set at browser parity, so real-world requests with very long URLs (for example ad-tech requests with 20+ KiB paths) pass through:

| Limit | Value | When exceeded |
|-------|-------|---------------|
| Request line | 4 MiB | `414 URI Too Long` + `Connection: close` |
| Header section | 4 MiB | `431 Request Header Fields Too Large` + `Connection: close` |

### Recording of rejected request heads

When the proxy itself closes a connection while reading a request head — a `request_timeout_ms` expiry after some bytes arrived, a `414`, or a `431` — it records a stream with state `error` so you can see why the client failed. The stream carries a failure reason (`request_head_timeout`, `request_line_too_long`, or `request_header_too_large`), a `closed_by=proxy` tag, the received raw bytes as a send flow, and the generated `414` / `431` as a receive flow. `capture_scope` applies.

## Connection limit

Each listener accepts up to `max_connections` concurrent connections (default `512`, sized for a single proxied browser). Connections refused at the cap are surfaced to the agent: `query` with `resource: "status"` reports `connection_rejections`, and refused targets are recorded as error streams with `blocked_by: capacity` (rate-limited to about 10 per second).

## HTTP request smuggling detection

The HTTP/1.x Layer's parser inspects the raw header bytes for smuggling patterns such as:

- Conflicting `Content-Length` and `Transfer-Encoding` headers
- Duplicate `Content-Length` headers with different values
- Obfuscated `Transfer-Encoding` values

Detected patterns are surfaced as anomalies on the resulting `HTTPMessage` envelope and tagged on the recorded flow for review. The original wire bytes remain intact on `Envelope.Raw` so a downstream analyst can inspect the exact byte sequence.

## Plugin hooks

HTTP/1.x envelopes dispatch the `http` plugin hooks:

| Hook | Phases | Supported actions |
|------|--------|-------------------|
| `("http", "on_request")` | `pre_pipeline`, `post_pipeline` | CONTINUE, DROP, RESPOND |
| `("http", "on_response")` | `pre_pipeline`, `post_pipeline` | CONTINUE, RESPOND |

See [Hook reference](../plugins/hook-reference.md) for the full surface.

## Starting an HTTP listener

```json
// proxy_start
{
  "listen_addr": "127.0.0.1:8080"
}
```

The default listener accepts both HTTP and HTTPS (via CONNECT) traffic on the same port. Protocol detection is automatic.

## Limitations

- **Forward proxy only** -- yorishiro-proxy does not support reverse proxy mode for HTTP/1.x
- **No HTTP/0.9** -- only HTTP/1.0 and HTTP/1.1 are supported
- **Absolute-form URLs required** -- clients must use proxy-style requests (most HTTP clients do this automatically when configured with `http_proxy`)

## Related pages

- [HTTPS MITM](https-mitm.md) -- how CONNECT tunnels and TLS interception work
- [WebSocket](websocket.md) -- WebSocket upgrade handling within HTTP connections
- [HTTP/2](http2.md) -- differences between HTTP/1.x and HTTP/2 handling
- [Target scope](../features/target-scope.md) -- restricting which hosts the proxy can access
- [Intercept](../features/intercept.md) -- pausing and modifying requests in flight
