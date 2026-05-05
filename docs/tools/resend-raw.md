# resend_raw

Resend a recorded raw byte payload via a freshly dialled TCP (or TLS) upstream connection. This is the smuggling-and-anomaly-test surface: payload bytes (`override_bytes`, `patches[].data`, recovered `Flow.RawBytes`) are NEVER normalized. They reach the wire verbatim.

`flow_id` is **required** -- `resend_raw` has no from-scratch path. Use [`fuzz_raw`](fuzz-raw.md) for ad-hoc byte injection.

## Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `flow_id` | string | Yes | | Recorded raw stream id |
| `target_addr` | string | Yes | | Upstream `host:port`. Explicit port required (no defaulting -- raw is protocol-agnostic) |
| `use_tls` | boolean | No | `false` | `true` to upgrade the dialled connection to TLS |
| `sni` | string | No | target host | SNI server name. Defaults to `target_addr` host portion when `use_tls=true` |
| `override_bytes` | string | No | | Replacement payload interpreted per `override_bytes_encoding`. Mutually exclusive with `patches` |
| `override_bytes_encoding` | string | No | `"text"` | `"text"` or `"base64"`. Use `"base64"` for binary smuggling payloads |
| `override_bytes_set` | boolean | No | `false` | Set `true` to replace with empty bytes; otherwise an empty `override_bytes` string is treated as no override |
| `patches` | array | No | | Offset-based byte patches applied to the recovered bytes (see below). Mutually exclusive with `override_bytes` |
| `insecure_skip_verify` | boolean | No | `false` | Skip TLS server certificate verification when `use_tls=true` |
| `tls_fingerprint` | string | No | | Informational v1; per-call selection is deferred |
| `timeout_ms` | integer | No | `30000` | Per-call timeout in milliseconds |
| `tag` | string | No | | Tag stored on the new flow's `Tags` map |

### patches

Each entry overwrites bytes at a given offset in the recovered payload:

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `offset` | integer | Yes | | Zero-based byte offset in the recovered payload |
| `data` | string | Yes | | Replacement bytes interpreted per `data_encoding` (must not be empty) |
| `data_encoding` | string | No | `"text"` | `"text"` or `"base64"` |

CR/LF in `target_addr` and `sni` is rejected, but never in `override_bytes`, `data`, or recovered `Flow.RawBytes` -- those are the wire and must reach it verbatim.

## Response

| Field | Type | Description |
|-------|------|-------------|
| `stream_id` | string | New stream record id holding the resend's send Flow plus every received chunk Flow |
| `response_bytes` | string | Concatenated payload across all received bytechunk envelopes (always base64) |
| `response_size` | integer | Total response size in bytes |
| `response_chunks` | integer | Envelope count for callers that want the segmentation shape |
| `truncated` | boolean | `true` when the receive loop hit the per-call response cap before the upstream closed |
| `duration_ms` | integer | Total duration in milliseconds |
| `tag` | string | Echo of the supplied tag (when set) |

## Pipeline placement

The resend traverses `PluginStepPost -> RecordStep`. `PluginStepPre` and `InterceptStep` are bypassed (RFC-001 §9.3).

## Examples

### Replay a recorded TCP flow as-is

```json
// resend_raw
{
  "flow_id": "raw-abc-123",
  "target_addr": "10.0.0.5:3306"
}
```

### Replay over TLS

```json
// resend_raw
{
  "flow_id": "raw-abc-123",
  "target_addr": "secure.example.com:8443",
  "use_tls": true
}
```

### Override the entire payload

```json
// resend_raw
{
  "flow_id": "raw-abc-123",
  "target_addr": "target.example.com:80",
  "override_bytes": "R0VUIC8gSFRUUC8xLjENCkhvc3Q6IGV4YW1wbGUuY29tDQoNCg==",
  "override_bytes_encoding": "base64"
}
```

### Apply byte-level patches

```json
// resend_raw
{
  "flow_id": "raw-abc-123",
  "target_addr": "target.example.com:80",
  "patches": [
    {"offset": 0, "data": "POST", "data_encoding": "text"}
  ]
}
```

## Related pages

- [resend_http](resend-http.md) -- HTTP request resend
- [resend_ws](resend-ws.md) -- WebSocket frame resend
- [resend_grpc](resend-grpc.md) -- gRPC RPC resend
- [fuzz_raw](fuzz-raw.md) -- Raw bytes fuzz with the same schema (and from-scratch injection)
- [Resender](../features/resender.md) -- Resender feature guide
