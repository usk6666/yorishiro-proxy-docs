# HTTPS MITM

yorishiro-proxy intercepts HTTPS traffic through man-in-the-middle (MITM) TLS termination. When a client sends an HTTP `CONNECT` request, the proxy establishes a TLS tunnel, dynamically issues a certificate for the target hostname, and decrypts the traffic for inspection and recording.

## CONNECT tunnel flow

The HTTPS interception follows the standard HTTP CONNECT proxy protocol:

```
1. Client ──CONNECT example.com:443──> Proxy
2. Proxy validates target scope and rate limits
3. Proxy ──200 Connection Established──> Client
4. Client ──TLS ClientHello──> Proxy (TLS handshake with dynamic cert)
5. Proxy <──decrypted HTTP──> Client (MITM tunnel established)
6. Proxy ──TLS──> example.com:443 (separate upstream TLS connection)
```

After the tunnel is established, the proxy reads decrypted HTTP requests from the client-side TLS connection, forwards them to the upstream server over a separate TLS connection, and records the full request/response flow.

## Dynamic certificate issuance

During the TLS handshake with the client, the proxy dynamically generates a certificate for the target hostname:

1. The client sends a TLS `ClientHello` with the SNI (Server Name Indication) extension
2. The proxy's `GetCertificate` callback extracts the server name from SNI (falls back to the CONNECT authority if SNI is empty, e.g., when connecting to an IP address)
3. A new certificate is issued on the fly, signed by the proxy's CA certificate
4. The client completes the TLS handshake using this dynamically issued certificate

The proxy enforces a minimum TLS version of TLS 1.2 for client-side connections.

## CA certificate management

For MITM to work, the client must trust the proxy's CA certificate. yorishiro-proxy supports three CA modes:

### File-based CA

Provide your own CA certificate and private key:

```bash
yorishiro-proxy server -ca-cert /path/to/ca.crt -ca-key /path/to/ca.key
```

### Ephemeral CA

Generate a temporary CA that exists only for the session:

```bash
yorishiro-proxy server -ca-ephemeral
```

This is useful for quick testing but requires re-installing the CA certificate each time the proxy restarts.

### Default CA

If no CA flags are specified, the proxy generates a persistent CA and stores it in `~/.yorishiro-proxy/`. The CA certificate can be exported and installed on the client system.

For detailed CA installation instructions, see [CA certificate](../getting-started/ca-certificate.md).

## ALPN negotiation

When the HTTP/2 handler is available, the proxy advertises both `h2` and `http/1.1` in the ALPN extension during the TLS handshake with the client:

- If the client negotiates **h2**, the connection is handed off to the [HTTP/2 handler](http2.md) via the `HandleH2` interface
- If the client negotiates **http/1.1** (or no ALPN), the connection stays with the HTTP/1.x handler

This means a single CONNECT tunnel can serve either HTTP/1.1 or HTTP/2 traffic depending on client preference.

## TLS passthrough

For hosts that should not be intercepted (e.g., certificate-pinned services, out-of-scope domains), you can configure TLS passthrough. When a CONNECT target matches a passthrough pattern, the proxy relays encrypted bytes directly without performing a TLS handshake:

```
Client ──encrypted bytes──> Proxy ──encrypted bytes──> Upstream
```

No certificate is issued, no traffic is decrypted, and no flow is recorded for passthrough connections.

Configure passthrough via the `configure` tool:

```json
// configure
{
  "tls_passthrough": ["pinned.example.com", "*.internal.corp"]
}
```

## TLS fingerprint spoofing

By default, the proxy uses a configurable TLS fingerprint profile (default: `chrome`) for upstream connections. This prevents upstream servers from detecting the proxy based on the TLS ClientHello fingerprint.

Available profiles: `chrome`, `firefox`, `safari`, `edge`, `random`, `none`.

The fingerprint applies to all outgoing TLS connections, including those established through CONNECT tunnels and WebSocket upgrades.

## HTTPS request processing

Once the MITM tunnel is established, the proxy processes HTTPS requests in a loop (supporting keep-alive), applying the same pipeline as HTTP/1.x:

1. **Target scope check** -- re-validates the Host header inside the tunnel (may differ from the CONNECT authority)
2. **Safety filter** -- checks request body and URL against safety rules
3. **Plugin hooks** -- dispatches `on_receive_from_client` and other hooks
4. **Intercept** -- pauses the request for AI agent review if matching rules exist
5. **Auto-transform** -- applies automatic request/response modifications
6. **Forward upstream** -- sends the request to the upstream server over TLS
7. **Record flow** -- stores the request/response as a flow with protocol `HTTPS`

### Flow recording

HTTPS flows are recorded with:

- Protocol: `HTTPS` (or `SOCKS5+HTTPS` when arriving via a SOCKS5 tunnel)
- TLS metadata: version, cipher suite, ALPN protocol, server certificate subject
- Full request/response with headers and body

## Plugin hooks

The `on_tls_handshake` lifecycle hook fires after each successful TLS handshake, providing:

- Server name (SNI)
- TLS version, cipher suite, ALPN protocol
- Client address

This hook is observe-only (fail-open) and cannot block the connection.

## Limitations

- **No TLS 1.0/1.1** -- the proxy requires TLS 1.2 or higher for client connections
- **No client certificate authentication** -- mutual TLS (mTLS) with the upstream server is not supported
- **SNI required for correct certificates** -- when connecting to IP addresses without SNI, the proxy falls back to the CONNECT authority for certificate generation
- **Passthrough is all-or-nothing** -- a passthrough host bypasses all recording and interception

## Related pages

- [HTTP/1.x](http.md) -- plaintext HTTP handling
- [HTTP/2](http2.md) -- HTTP/2 over TLS via ALPN
- [SOCKS5](socks5.md) -- SOCKS5 tunnel with HTTPS MITM
- [CA certificate](../getting-started/ca-certificate.md) -- installing the CA certificate
- [TLS configuration](../configuration/tls.md) -- TLS-related settings
