# TLS

yorishiro-proxy provides fine-grained control over TLS behavior for upstream connections. You can spoof TLS fingerprints, bypass MITM for specific domains, configure mTLS client certificates, and manage TLS verification on a per-host basis.

## TLS fingerprinting

When connecting to upstream HTTPS servers, yorishiro-proxy uses [uTLS](https://github.com/refraction-networking/utls) to mimic a real browser's TLS ClientHello. This helps evade JA3/JA4-based bot detection systems.

### Available profiles

| Profile | Description |
|---------|-------------|
| `firefox` | Mimics Mozilla Firefox's TLS fingerprint (**default**) |
| `chrome` | Mimics Google Chrome's TLS fingerprint |
| `safari` | Mimics Apple Safari's TLS fingerprint |
| `edge` | Mimics Microsoft Edge's TLS fingerprint |
| `random` | Randomly selects a profile for each connection |
| `none` | Uses Go's standard `crypto/tls` stack (no fingerprint spoofing) |

### Setting the fingerprint profile

**CLI flag:**

```bash
yorishiro-proxy -tls-fingerprint firefox
```

**Environment variable:**

```bash
YP_TLS_FINGERPRINT=firefox yorishiro-proxy
```

**Config file:**

```json
{
  "tls_fingerprint": "firefox"
}
```

**Runtime change via MCP:**

```json
// configure
{
  "tls_fingerprint": "safari"
}
```

The fingerprint is **runtime-mutable**: subsequent outbound dials use the new profile, while connections already pinned to a pooled upstream Layer retain the fingerprint they were established with.

When nothing is configured, the profile resolves to `firefox` on every path — live MITM traffic, resend, and fuzz. Set `none` to opt out of spoofing entirely. Values are case-insensitive, and an unknown value (for example a typo like `firefx`) is rejected at config load, at the `-tls-fingerprint` flag, and by the `proxy_start` / `configure` tools, instead of silently falling back to a Go-native ClientHello.

### HTTP/2 fingerprint

Anti-bot frontends also fingerprint a client's HTTP/2 behavior. With `firefox` selected, the upstream HTTP/2 connection preface matches Firefox as well:

| Element | Firefox shape |
|---------|---------------|
| SETTINGS (wire order) | `HEADER_TABLE_SIZE=65536`, `ENABLE_PUSH=0`, `INITIAL_WINDOW_SIZE=131072`, `MAX_FRAME_SIZE=16384` |
| Connection WINDOW_UPDATE | `12517377` |
| Request pseudo-header order | `:method :path :authority :scheme` |

This corresponds to the Akamai H2 fingerprint `1:65536;2:0;4:131072;5:16384|12517377|0|m,p,a,s`. There is no separate setting — the H2 shape follows `tls_fingerprint`.

!!! note "Limitations"
    - Only `firefox` has a dedicated H2 profile. `chrome`, `edge`, `safari`, `random`, and `none` use the default (Chrome-shaped) H2 behavior.
    - Firefox's PRIORITY frame tree is not reproduced.
    - Both the TLS ClientHello and the H2 shape are pinned to **Firefox 120**, the newest Firefox profile uTLS ships. An anti-detect browser advertising a newer Firefox user agent shows a small version skew, but the fingerprint is still a genuine Firefox fingerprint.
    - HTTP/3 / QUIC is out of scope: the proxy advertises only `h2` and `http/1.1` via ALPN.

### Client fingerprint capture (JA3 / JA4)

On the MITM path, yorishiro-proxy also computes the JA3 and JA4 fingerprints of the **client's** ClientHello (your browser or tool). They are recorded with the stream and surface as:

- `tls_client_ja3` / `tls_client_ja4` in the `conn_info` of a [`query`](../tools/query.md) flow result
- `ja3` / `ja4` keys on `ctx.tls` and in the `tls.on_handshake` hook data for [plugins](../plugins/data-map-reference.md)

This is capture only: the client's fingerprint is not mirrored to the upstream dial (the upstream uses `tls_fingerprint`). ClientHellos larger than 4 KiB are not fingerprinted. Passthrough connections are fingerprinted too, and the values appear on the TLS handshake audit flow.

## TLS passthrough

TLS passthrough lets you specify domains that should bypass MITM interception entirely. Connections to matching hosts are forwarded as raw TCP tunnels without decryption. This is useful for certificate-pinned applications or traffic you don't need to inspect.

### Supported patterns

- **Exact match**: `"example.com"` -- matches only `example.com`
- **Wildcard**: `"*.example.com"` -- matches any subdomain of `example.com`

### Config file

```json
{
  "tls_passthrough": [
    "*.google.com",
    "*.googleapis.com",
    "certificate-pinned.example.com"
  ]
}
```

### Runtime configuration via MCP

Add patterns incrementally:

```json
// configure
{
  "tls_passthrough": {
    "add": ["*.apple.com", "*.icloud.com"]
  }
}
```

Remove patterns:

```json
// configure
{
  "tls_passthrough": {
    "remove": ["*.apple.com"]
  }
}
```

Replace the entire list:

```json
// configure
{
  "operation": "replace",
  "tls_passthrough": {
    "patterns": ["*.google.com", "*.googleapis.com"]
  }
}
```

## mTLS client certificates

When upstream servers require mutual TLS (mTLS) authentication, you can configure client certificates globally or per host.

### Global client certificate

Set a default client certificate used for all upstream connections. Both `client_cert` and `client_key` must be provided together.

**Config file:**

```json
{
  "client_cert": "/path/to/client.crt",
  "client_key": "/path/to/client.key"
}
```

**Runtime change via MCP:**

```json
// configure
{
  "client_cert": {
    "cert_path": "/path/to/client.crt",
    "key_path": "/path/to/client.key"
  }
}
```

To remove the global client certificate at runtime:

```json
// configure
{
  "client_cert": {
    "cert_path": "",
    "key_path": ""
  }
}
```

### Per-host client certificates

Use the `host_tls` section to assign different client certificates to specific hosts. Per-host settings override the global client certificate for matching hosts.

```json
{
  "host_tls": {
    "api.internal.corp": {
      "client_cert": "/path/to/api-client.crt",
      "client_key": "/path/to/api-client.key"
    },
    "payments.example.com": {
      "client_cert": "/path/to/payments-client.crt",
      "client_key": "/path/to/payments-client.key"
    }
  }
}
```

Wildcard patterns are supported:

```json
{
  "host_tls": {
    "*.internal.corp": {
      "client_cert": "/path/to/internal-client.crt",
      "client_key": "/path/to/internal-client.key"
    }
  }
}
```

## TLS verification control

By default, yorishiro-proxy verifies upstream TLS certificates. You can disable verification globally or per host.

### Global verification skip

Disable TLS verification for all upstream connections using the `-insecure` flag. This is useful when targets use self-signed or expired certificates.

```bash
yorishiro-proxy -insecure
```

Or via environment variable:

```bash
YP_INSECURE=true yorishiro-proxy
```

!!! warning
    Disabling TLS verification removes security checks on upstream connections. Only use this when necessary, such as during vulnerability assessments against development environments.

### Per-host verification

Use `host_tls` entries to control verification per host. The `tls_verify` field accepts three states:

| Value | Behavior |
|-------|----------|
| `true` | Enforce TLS verification for this host (overrides `-insecure`) |
| `false` | Skip TLS verification for this host |
| `null` (omitted) | Use the global setting |

```json
{
  "host_tls": {
    "*.staging.example.com": {
      "tls_verify": false
    },
    "production.example.com": {
      "tls_verify": true
    }
  }
}
```

### Custom CA bundles

When an upstream server uses a private CA, you can specify a custom CA bundle per host:

```json
{
  "host_tls": {
    "private-ca.example.com": {
      "ca_bundle": "/path/to/custom-ca.pem",
      "tls_verify": true
    }
  }
}
```

## host_tls reference

Each entry in the `host_tls` map is keyed by a hostname or wildcard pattern and supports the following fields:

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `client_cert` | `string` | -- | Path to PEM-encoded client certificate |
| `client_key` | `string` | -- | Path to PEM-encoded client private key |
| `tls_verify` | `bool\|null` | `null` (use global) | TLS verification: `true` = enforce, `false` = skip, `null` = global setting |
| `ca_bundle` | `string` | -- | Path to PEM-encoded CA bundle file |

### Combined example

A comprehensive `host_tls` configuration combining mTLS, verification control, and custom CA:

```json
{
  "tls_fingerprint": "firefox",
  "tls_passthrough": ["*.google.com"],
  "client_cert": "/path/to/default-client.crt",
  "client_key": "/path/to/default-client.key",
  "host_tls": {
    "api.internal.corp": {
      "client_cert": "/path/to/api-client.crt",
      "client_key": "/path/to/api-client.key"
    },
    "*.staging.example.com": {
      "tls_verify": false
    },
    "private-ca.example.com": {
      "ca_bundle": "/path/to/custom-ca.pem",
      "tls_verify": true
    },
    "*.dev.example.com": {
      "client_cert": "/path/to/dev-client.crt",
      "client_key": "/path/to/dev-client.key",
      "tls_verify": false
    }
  }
}
```

## Related pages

- [CLI flags](cli-flags.md) -- All available CLI flags including TLS options
- [Config file](config-file.md) -- Full config file reference
- [HTTPS MITM](../protocols/https-mitm.md) -- How HTTPS interception works
- [CA certificate](../getting-started/ca-certificate.md) -- CA certificate setup
