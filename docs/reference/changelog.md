# Changelog

The authoritative changelog lives in the source repository at
[`CHANGELOG.md`](https://github.com/usk6666/yorishiro-proxy/blob/main/CHANGELOG.md).
The summary below tracks the user-facing changes that shape this documentation;
see the upstream file for the complete per-PR entries.

## Unreleased

### Added

- **`grpc_schema` MCP tool** -- register `.proto` schemas via descriptor-set upload, host `protoc`, or reflection-based discovery (USK-923 / USK-926 / USK-927 / USK-928 / USK-933). Unlocks `body_decoded_encoding="proto-json"` on [`query`](../tools/query.md) and `body_encoding="proto-json"` on [`resend_grpc`](../tools/resend-grpc.md). See [grpc_schema](../tools/grpc-schema.md).
- **TCP forward L7 dispatch** -- `tcp_forwards[]` entries gain `upstream_tls`, `upstream_insecure_skip_verify`, and an `sse` protocol value. The full TLS × UpstreamTLS matrix is documented on [proxy_start](../tools/proxy-start.md#tcp_forwards) and [config_file](../configuration/config-file.md#tcp_forwards) (USK-911 .. USK-918).
- **Wire-level overlay envelopes** -- `h1-chunk`, `h2-frame`, `grpc-lpm-frame`, `grpcweb-base64` rows surface via `filter.wire_level` on [`query`](../tools/query.md#wire-level-overlays) and [`manage.export_flows`](../tools/manage.md) for forensic export (USK-895 / USK-896 / USK-897 / USK-899 / USK-932).
- **`max_concurrent_streams` end-to-end** -- HTTP/2 `SETTINGS_MAX_CONCURRENT_STREAMS` is exposed via [`proxy_start`](../tools/proxy-start.md) / [`configure`](../tools/configure.md), the `-max-concurrent-streams` CLI flag, and the `YP_MAX_CONCURRENT_STREAMS` env var (USK-862). Default raised 100 -> 500.
- **`query` body inclusion controls** -- `include_bodies` and `body_max_bytes` parameters opt in to body payloads with a per-call size cap.
- **Per-protocol intercept hold/timeout overrides** -- `intercept_queue.protocol_overrides` lets operators set distinct `hold_timeout` and `timeout_behavior` per canonical `envelope.Protocol`. WS / SSE / gRPC / gRPC-Web default raised 8 s -> 60 s (USK-855).
- **WS hold-window keepalive injection** -- the WS Layer injects synthetic ping frames toward the upstream while a frame is held in intercept review (USK-854).
- **Linux NSSDB CA registration in `setup`** -- `install ca --trust` now registers the proxy CA into the per-user NSSDB so Chromium/Firefox trust intercepted traffic without a manual `certutil` step (USK-857).
- **`proxy_start` listen-address collision detection** -- `listen_addr` and `tcp_forwards` ports are checked before bind, with rollback on any partial-bind failure.
- **HTTP/2 extended CONNECT** -- ServerRole mirrors the upstream's `SETTINGS_ENABLE_CONNECT_PROTOCOL`, enabling RFC 8441 extended `CONNECT` end-to-end.
- **HTTPMessage anomalies on Flow rows** -- parser-detected anomalies persist on the flow row and surface through `query`.

### Changed

- **HTTP/2 `SETTINGS_MAX_CONCURRENT_STREAMS` default raised 100 -> 500** (USK-862).
- **WS / SSE intercept hold-timeout default raised 8 s -> 60 s** (USK-855 / #860).
- **HTTP/1 and WS layers wire `StateReleaser.ReleaseStream` on terminal events** (#849) -- removes a slow plugin-state leak.

### Removed

- **Breaking**: technology detection (the `query` tool's `technologies` resource and `technology` filter, plus the `internal/fingerprint/` package) is deleted (USK-843). Migrate to response-header matching via existing `query` filters or a Starlark plugin.
- **Breaking**: `proxy_start.protocols` MCP input and `query("config").enabled_protocols` output are deleted (USK-870). The WebUI Settings -> Proxy panel no longer shows the protocol-selection toggle. Use `target_scope` (host), `intercept_rules` (request pattern), `tls_passthrough` (MITM target), and `capture_scope` (recording target) for the same effect.

### Fixed

- **Security (USK-879)** -- `security set_target_scope` is enforced on CONNECT tunnels, plain-HTTP forward proxy, and SOCKS5 tunnels. Before this fix, agent-layer scope mutations reached only MCP-initiated requests (`resend_*` / `fuzz_*` / `test_target`); client traffic through CONNECT / plain HTTP / SOCKS5 bypassed the evaluator.
- **`failure_reason` classification (USK-858)** -- client-side MITM handshake rejections (e.g. Chromium refusing the proxy CA with an `unknown_certificate` / `bad_certificate` TLS alert) are now tagged `client_tls_error` instead of being misclassified as `upstream_tls_error`. See [HTTPS MITM](../protocols/https-mitm.md#diagnosing-handshake-failures-client_tls_error-vs-upstream_tls_error).
- **Macro unresolved-template warnings (#838)** -- variables that remain unsubstituted after rendering are surfaced as a warning rather than sent upstream as `{{var}}`.
- **`request_timeout_ms` on plain HTTP / CONNECT** -- the timeout is now honored on the inner read deadline of CONNECT tunnels and on plain-HTTP forward (#840).
- **CONNECT / SOCKS5 passthrough audit** -- the original target hostname is preserved on the passthrough audit envelope (#839).
- **HAR export protocol predicates** -- aligned with canonical `Envelope.Protocol` values (#848).
- **`resend_http` query-string handling** -- `?` inside `path` is auto-split into `path` + `raw_query`; combining a `?`-bearing `path` with an explicit `raw_query` is rejected (#855).
- **Intercept-release EOF tag** -- only attached on successful downstream relay (#857).
- **HTTP/2 connection-specific headers stripped on send** -- RFC 7540 §8.1.2.2 hop-by-hop headers (`Connection`, `Keep-Alive`, `Proxy-Connection`, `Transfer-Encoding`, `Upgrade`) are stripped on the H2 send path (#836).
- **Post-hold upstream EOF (USK-851)** -- surfaced via a Stream tag instead of opaque connection loss.
- **WebSocket permessage-deflate with context-takeover (USK-867)** -- multi-frame messages now correctly serialize `deflateState`.
- **Sec-WebSocket-Extensions propagation (#845)** -- the negotiated value reaches the post-Upgrade `ws.Layer`.
- **WebSocket StreamID reuse across handshake -> frames (#843)** -- keeps Upgrade + data-frame phase under the same Stream identity.
- **`query` filter `scheme=ws|wss` is rejected** (USK-848).
- **`manage` export/import path resolution clarified** -- relative paths resolve against the proxy server's CWD (USK-868).

### Docs

- `target_scope` / `rate_limit` / `budget` applicability scopes (Agent vs Policy Layer) are spelled out on [security model](../concepts/security-model.md#applicability-scopes-agent-vs-policy-layer) and in the `security` MCP help text (USK-842).

## 0.15.0 (2026-05-05)

Shipped the [RFC-001 Envelope + Layered Connection Model](https://github.com/usk6666/yorishiro-proxy/blob/main/docs/rfc/envelope.md) rewrite (milestones N1-N9). The data path was rebuilt around an `Envelope` + typed `Message` + `Layer` + `Channel` model that fixes the HTTP bias of the previous `Exchange`/`Codec` abstractions and unlocks structurally-honest support for HTTP/2 multiplexing, gRPC, gRPC-Web, WebSocket, SSE, and raw-byte smuggling diagnostics.

This release contained breaking changes for plugin authors and MCP clients. There is **no compatibility shim**; see the upstream CHANGELOG for the full Added / Changed / Removed / Migration sections.

## 0.14.x and earlier

Pre-RFC-001 history is preserved as git tags `v0.3.0` through `v0.14.1` and in the corresponding GitHub Releases. Detailed per-tag changelogs were not maintained at the time; consult `git log` in the upstream repo for fine-grained history.
