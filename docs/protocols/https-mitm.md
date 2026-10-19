# HTTPS MITM

yorishiro-proxy intercepts HTTPS traffic through man-in-the-middle (MITM) TLS termination. When a client sends an HTTP `CONNECT` request, the connector establishes a TLS tunnel, dynamically issues a certificate for the target hostname, and decrypts the traffic for inspection and recording.

The flow is owned by three components:

- The **connector** (`internal/connector/`) accepts the TCP connection.
- `connect_handler.go` recognizes the `CONNECT` request and completes the tunnel.
- The **TLS Layer** (`internal/layer/tlslayer/`) terminates TLS on both the client and upstream sides.
- ALPN routing (`internal/connector/alpn_routing.go`) chooses the inner Layer (`http1`, `http2`, or `bytechunk` for unknown ALPN).

## CONNECT tunnel flow

The HTTPS interception follows the standard HTTP CONNECT proxy protocol:

```
1. Client ──CONNECT example.com:443──> connector
2. connector validates target scope and rate limits
3. connector ──200 Connection Established──> Client
4. connector peeks the client's ClientHello (SNI, ALPN offer, JA3/JA4) without consuming it
5. connector dials upstream and runs tlslayer.Client toward example.com:443,
   offering the client's ALPN list verbatim
6. tlslayer.Server completes the client handshake (MITM cert presentation),
   advertising only the protocol the upstream selected
7. ALPN routing chooses the inner Layer (http1 / http2 / bytechunk)
8. Inner traffic flows through the chosen Layer + standard Pipeline
```

The CONNECT request and the client-side TLS handshake are both bounded by `request_timeout_ms`, so a client that stalls mid-handshake cannot hold a connection slot indefinitely.

`tlslayer.Server` and `tlslayer.Client` each return a `*envelope.TLSSnapshot` capturing the negotiated TLS parameters (SNI, ALPN, peer certificate, cipher suite, version). Both snapshots are attached to envelopes produced by the inner Layer.

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

The proxy uses a **sniff-first** flow so that ALPN on both legs reflects what the upstream actually chose, rather than a guess:

1. The client's ClientHello is peeked before the MITM handshake.
2. The client's ALPN offer list is forwarded to the upstream byte-for-byte — same order, no deduplication, no case folding.
3. The upstream's selected protocol is advertised back to the client as a single-element ALPN list (or omitted if the upstream selected none), so the client necessarily agrees.

This keeps the end-to-end protocol faithful even when an upstream violates RFC 7301 (for example by answering `http/1.1` to an `h2`-only offer). If the peek fails — timeout, non-TLS bytes, ECH, or a ClientHello larger than 4 KiB — the proxy falls back to offering `h2` and `http/1.1` and redialing as needed.

ALPN routing then picks the inner Layer:

- `http/1.1` (or empty ALPN) — the inner Layer is `internal/layer/http1/`
- `h2` — the inner Layer is `internal/layer/http2/` (see [HTTP/2](http2.md))
- Anything else — the inner Layer falls back to `internal/layer/bytechunk/` for raw passthrough with TLS visibility

A single CONNECT tunnel can therefore serve HTTP/1.1, HTTP/2, or any opaque protocol depending on the negotiated ALPN.

## TLS passthrough

For hosts that should not be intercepted (e.g., certificate-pinned services, out-of-scope domains), you can configure TLS passthrough. When a CONNECT target matches a passthrough pattern, the proxy relays encrypted bytes directly without performing a TLS handshake:

```
Client ──encrypted bytes──> Proxy ──encrypted bytes──> Upstream
```

No certificate is issued and no traffic is decrypted. The SNI and the client's JA3/JA4 fingerprints are still observed from the ClientHello (without delaying the handshake) for the TLS handshake audit record.

Configure passthrough via the `configure` tool:

```json
// configure
{
  "tls_passthrough": ["pinned.example.com", "*.internal.corp"]
}
```

## TLS fingerprint spoofing

By default, the proxy uses a configurable TLS fingerprint profile (default: `firefox`) for upstream connections. This prevents upstream servers from detecting the proxy based on the TLS ClientHello fingerprint. With `firefox`, the upstream HTTP/2 SETTINGS, WINDOW_UPDATE, and pseudo-header order match Firefox too.

Available profiles: `firefox`, `chrome`, `safari`, `edge`, `random`, `none`.

The fingerprint applies to all outgoing TLS connections, including those established through CONNECT tunnels and WebSocket upgrades. See [TLS fingerprinting](../configuration/tls.md#tls-fingerprinting) for details.

Independently, the proxy records the JA3 / JA4 fingerprint of the **client's** ClientHello as `tls_client_ja3` / `tls_client_ja4` on the flow's connection info.

## HTTPS request processing

Once the MITM tunnel is established, traffic flows through the chosen inner Layer's Channel and the standard Pipeline. For `http1`-routed CONNECT tunnels the request loop matches the plaintext [HTTP/1.x](http.md) flow exactly; for `h2`-routed tunnels see the [HTTP/2 page](http2.md) for stream-level details.

The canonical Pipeline chain runs on every Envelope:

`HostScope → HTTPScope → Budget → Safety → PluginPre → Intercept → Transform → PluginPost → Record`

### Flow recording

Recorded envelopes carry both TLS snapshots produced by the `tlslayer/` Layer:

- `clientSnap` — the synthetic MITM TLS handshake presented to the client
- `upstreamSnap` — the real upstream TLS handshake observed at dial time

Each snapshot exposes version, cipher suite, ALPN protocol, peer certificate chain, and SNI. Envelopes also carry the `Envelope.Protocol` of the inner Layer (e.g. `http`, `ws`, `grpc`).

### Diagnosing handshake failures (`client_tls_error` vs `upstream_tls_error`)

Failed MITM handshakes are tagged with a `failure_reason` so the agent can distinguish *which side* refused the handshake (USK-858):

| `failure_reason` | Means | Typical causes |
|------------------|-------|----------------|
| `client_tls_error` | The client rejected the proxy's dynamically-issued certificate (TLS alert observed at the client-facing handshake) | CA not installed / not trusted, certificate pinning, or a proxy-CA mismatch between processes |
| `upstream_tls_error` | The proxy's dial to the upstream failed at the TLS layer | Upstream cert expired, hostname mismatch, untrusted chain, broken cipher negotiation |

Before USK-858 the proxy misclassified `unknown_certificate` / `bad_certificate` alerts from the client side as `upstream_tls_error`, hiding the real cause. Query for the recorded reason via `flow.tags["failure_reason"]` rather than parsing `tags["error"]`.

## Plugin hooks

The `("tls", "on_handshake")` lifecycle hook fires after each successful TLS handshake, providing:

- Server name (SNI)
- TLS version, cipher suite, ALPN protocol, peer certificate subject
- The client's JA3 / JA4 fingerprints (`ja3`, `ja4`)

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
