# HTTP/1.x

yorishiro-proxy operates as a forward proxy for plaintext HTTP/1.x traffic. It automatically detects HTTP requests, records complete request/response flows, and supports keep-alive connections.

## How detection works

When a new TCP connection arrives, the proxy peeks at the first bytes and checks for known HTTP method prefixes:

- `GET `, `POST `, `PUT `, `DELETE `, `HEAD `, `OPTIONS `, `PATCH `, `CONNECT `

If the peeked bytes match any of these prefixes, the connection is dispatched to the HTTP/1.x handler. The `CONNECT` method triggers the [HTTPS MITM](https-mitm.md) path instead.

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

Every request and response flowing through the proxy is recorded as a **flow** in the flow store. Each flow contains:

- **Request message** (direction: `send`): method, URL, headers, body, raw bytes
- **Response message** (direction: `receive`): status code, headers, body, raw bytes

The proxy captures the raw wire-format bytes before any normalization, which is useful for detecting [HTTP request smuggling](https://portswigger.net/web-security/request-smuggling) patterns.

### Body size limits

Request and response bodies are recorded up to a configurable maximum size. Raw byte capture is capped at 2 MB to prevent excessive memory usage.

### Keep-alive support

The handler processes multiple requests on the same connection in a loop, supporting HTTP/1.1 keep-alive. When the client sends `Connection: close`, the loop terminates after that request.

## Hop-by-hop headers

The proxy removes hop-by-hop headers before forwarding requests upstream, as specified by HTTP/1.1:

- `Connection`, `Keep-Alive`, `Proxy-Authenticate`, `Proxy-Authorization`
- `Proxy-Connection`, `Te`, `Trailer`, `Transfer-Encoding`, `Upgrade`

## WebSocket upgrade

When the proxy detects a WebSocket upgrade request (`Connection: Upgrade` + `Upgrade: websocket`), it hands off to the [WebSocket handler](websocket.md) for message-level recording. See the WebSocket page for details.

## Request timeout

The proxy enforces a configurable read timeout (default: 60 seconds) for HTTP request headers. This protects against Slowloris-style denial-of-service attacks. If no complete request header arrives within the timeout, the connection is closed.

## HTTP request smuggling detection

Before parsing requests with Go's `net/http.ReadRequest`, the proxy inspects the raw header bytes for smuggling patterns such as:

- Conflicting `Content-Length` and `Transfer-Encoding` headers
- Duplicate `Content-Length` headers with different values
- Obfuscated `Transfer-Encoding` values

Detected patterns are logged as warnings and tagged on the recorded flow for review.

## Plugin hooks

The HTTP/1.x handler dispatches the following plugin hooks during request processing:

| Hook | Trigger | Supported actions |
|------|---------|-------------------|
| `on_receive_from_client` | After receiving a request from the client | CONTINUE, DROP, RESPOND |
| `on_before_send_to_server` | Before forwarding the request upstream | CONTINUE (modify headers/body) |
| `on_receive_from_server` | After receiving the response from upstream | CONTINUE (modify headers/body) |
| `on_before_send_to_client` | Before sending the response to the client | CONTINUE (modify headers/body) |

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
