# Changelog

The authoritative changelog lives in the source repository at
[`CHANGELOG.md`](https://github.com/usk6666/yorishiro-proxy/blob/main/CHANGELOG.md).
The summary below tracks the user-facing changes that shape this documentation;
see the upstream file for the complete per-PR entries.

## Unreleased

_No changes yet._

## 0.20.1 (2026-10-19)

A bug-fix release on top of 0.20.0.

### Fixed

- **HTTP/2 resend of h2-recorded flows** -- [`resend_http`](../tools/resend-http.md) and [`fuzz_http`](../tools/fuzz-http.md) now send flows recorded over HTTP/2 over HTTP/2 instead of silently downgrading to HTTP/1.1 (#171).
- **Native gRPC blocks answer `PERMISSION_DENIED`** -- a native gRPC call blocked by scope, the SafetyFilter, or an intercept drop now fails with gRPC status `PERMISSION_DENIED` instead of a transport error (#176).
- **Server-first protocols** -- WebSocket frames and raw TCP bytes the server sends first are relayed without waiting for the client (#175, #180).
- **Intercept holds ended by context cancellation** are recorded as `blocked_by: intercept_cancelled` (#179).
- **Long-lived HTTP/2 connections** -- closed streams are retired so the connection is no longer refused after many streams (#174).
- **Recorded TLS passthrough** no longer stalls 5 s on the ClientHello peek (#172).
- **`capture_scope` on overlay rows** is decided from the stream's request head (#173), and `Stream.Scheme` is backfilled when an `h2-frame` overlay creates the stream (#170).
- **gRPC-Web** forwards malformed bodies and request trailers and reconciles framing headers (#177).

## 0.20.0 (2026-10-19)

This release tightens target-scope enforcement end to end and adds operator-side controls for the Policy Layer and export paths.

### Added

- **`SIGHUP` reloads the Policy Layer** -- operators can edit the target-scope policy file and send `SIGHUP` to apply it without restarting (#148). See [security](../tools/security.md#reloading-the-policy-layer-operator-only).
- **Target scope re-checked at more points** -- every fuzz variant is checked against target scope (#153), an intercept hold is re-checked when it is released (#154), and admitted live connections are closed when a policy change denies them (#155, #168).
- **`max_connections` rejections surfaced to the agent** -- rejected connections are recorded with `blocked_by: capacity` and counted under `connection_rejections` in `query("status")` (#149).

### Changed

- **Breaking**: `manage` `export_flows` / `import_flows` file paths are confined to an operator export directory (`-export-dir` / `YP_EXPORT_DIR`, default `~/.yorishiro-proxy/exports/`). Paths outside it are rejected (#142). See [manage](../tools/manage.md).
- **`max_connections` default raised 128 -> 512** (#144).
- **HPACK and request-line limits raised to browser parity**; an oversized request line is answered with `414` (#151).
- **`request_timeout_ms` bounds the client request head** (slowloris protection) (#158).
- **Recorded request paths stay in wire form** -- the path is kept exactly as sent (including `%XX` escapes) in flows, `path_pattern` / `url_prefix` matching, and resend (#150).
- **Config validation at load** -- `tls_fingerprint` (#116) and Policy Layer target-scope rules (#141) are validated when the config is loaded.

### Fixed

- **Target scope rules** -- bracketed IPv6 hosts, `path_prefix`, and empty-host rules now take effect (#143); every scope check receives a resolved scheme derived from the observed TLS state (#129, #132).
- **gRPC SafetyFilter** -- the Input SafetyFilter now scans `:path` and the query string (#131).
- **`resend_grpc`** takes its TLS decision from the observed transport instead of the declared scheme (#125).
- **CLI client** coerces nullable and dot-notation parameters to their schema type (#147).
- Numerous HTTP/2 robustness fixes around GOAWAY, CONTINUATION, flow control, and `REFUSED_STREAM` propagation (#145, #146, #156, #159, #161–#163, #166).

## 0.19.1 (2026-09-14)

### Fixed

- **npm distribution** -- fixed the release publish paths so the npm packages install correctly (#114).

## 0.19.0 (2026-09-14)

### Added

- **`docs` MCP tool and `docs` CLI subcommand** -- the full reference ships in-band: agents call [`docs`](../tools/docs.md) with a `topic` (and optional `section`), and humans run `yorishiro-proxy docs <topic>` (#106, #110).
- **npm distribution** -- prebuilt binaries are published to npm via `optionalDependencies` (#84). See [installation](../getting-started/installation.md).
- **Client ClientHello fingerprint capture** -- the JA3 / JA4 of the client's ClientHello is recorded on the flow (`conn_info.tls_client_ja3` / `tls_client_ja4`) (#88).
- **Firefox-shaped upstream HTTP/2 fingerprint** -- the upstream HTTP/2 SETTINGS / priority shape follows `tls_fingerprint` (#87).
- **Loud failure for unresolved macro template variables** -- an unresolved `§var§` is reported as a warning instead of passing silently (#101). The `§var§` syntax is now documented in the macro tool schema (#98).

### Changed

- **`tls_fingerprint` default is now `firefox`** (was `chrome`), and the `none` escape hatch is fixed (#89, #93).

### Fixed

- **SafetyFilter violation logs redact the matched fragment** (CWE-312) (#85).
- **MCP prompt playbooks** disambiguate `{{}}` prompt placeholders from `§§` macro templates (#99).

## 0.18.2 (2026-05-26)

### Added

- **Sniff-first MITM** -- the client's ClientHello ALPN is peeked before the upstream is dialed, so the proxy negotiates the same protocol on both legs (#69, #70).
- **Upstream redial metrics** -- `h1_upstream` / `h2_upstream` redial counters are exposed in `query("status")` (#73, #74).

### Fixed

- **Stale HTTP/1.x upstream connections** are detected and redialed before send, with a replay-safe retry on `EPIPE` (#71, #72).

## 0.18.1 (2026-05-24)

### Fixed

- **HTTP/2 GOAWAY handling** -- upstream GOAWAY now triggers a transparent redial / pre-warm across N-chain upstream proxies, matching browser behaviour (#65–#68).

## 0.18.0 (2026-05-23)

### Added

- **Upstream proxy rotation** -- `upstream_proxy` accepts a URL template with a rotation policy, both per listener for live MITM traffic ([`proxy_start`](../tools/proxy-start.md) / [`configure`](../tools/configure.md)) and per iteration for [`fuzz_http`](../tools/fuzz-http.md) / [`resend_http`](../tools/resend-http.md) (#33, #34, #50). The WebUI Settings page gains an upstream proxy editor (#42).
- **Fuzzer macro hooks** -- `fuzz_http`, `fuzz_ws`, `fuzz_raw`, and `fuzz_grpc` support per-iteration and per-job (`scope: "job"`) pre/post macro hooks, per-hook `vars` overrides, and `run_interval` (#38, #47, #55, #57–#64). See [fuzzer](../features/fuzzer.md).
- **`query("config")` exposes `intercept_queue`** settings, which the WebUI uses to prefill its forms (#52).

### Fixed

- WebUI fixes across Flow detail, Intercept, Resend, Security, and Settings, including pseudo-header filtering in cURL / HAR exports (#35–#51).

## 0.17.0 (2026-05-21)

This release migrates the bundled vulnerability-verification playbooks from a filesystem-installed `.claude/skills/` skill tree to MCP Prompts delivered over the standard `prompts/list` / `prompts/get` surface. The `yorishiro-proxy install skills` subcommand and the `--skills-dir` flag are removed -- after upgrading, the same nine playbooks become visible to your MCP host automatically on next connect, with no install step.

### Added

- **MCP Prompts: nine vulnerability-verification playbooks** -- `verify-idor`, `verify-sqli`, `verify-xss`, `verify-csrf`, `audit-auth`, `fuzz-endpoint`, `replay-with-mods`, `capture-traffic`, `stateful-fuzz-loop` are now served via the MCP `prompts/list` / `prompts/get` surface (USK-949 / #28). Hosts that surface prompts (Claude Code, Claude Desktop, ...) see them automatically on connect. See [MCP prompts](../guides/mcp-prompts.md).
- **`proxy_start` reset semantics + partial-update guidance** -- the [`proxy_start`](../tools/proxy-start.md) tool description, JSONSchema, and help resource now spell out that each call fully resets prior session settings, and document the recommended "initialize with `proxy_start`, update with `configure`" pattern (#29).
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

- **Breaking**: `yorishiro-proxy install skills` subcommand, the `--skills-dir` flag, and the bundled `.claude/skills/yorishiro/` skill tree are deleted (USK-951 / #32). Replacement: the same vulnerability-verification playbooks are served as [MCP Prompts](../guides/mcp-prompts.md) over `prompts/list` / `prompts/get`. Existing `.claude/skills/yorishiro/` directories on disk are inert -- delete them at your convenience. `yorishiro-proxy install skills` now errors with `unknown install target`.
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
- **TLS passthrough audit: `ECONNRESET` classified as tunneled** (USK-952 / #30) -- peer-side ungraceful close after bytes have flowed (TLS `close_notify` in flight, abrupt browser / mobile-handover close) is now an `outcome="tunneled"` audit record instead of being misclassified as `outcome="failed"`. Genuine mid-stream failures (i/o timeout, no route to host) still land in the `failed` bucket.

### Docs

- `target_scope` / `rate_limit` / `budget` applicability scopes (Agent vs Policy Layer) are spelled out on [security model](../concepts/security-model.md#applicability-scopes-agent-vs-policy-layer) and in the `security` MCP help text (USK-842).

## 0.15.0 (2026-05-05)

Shipped the [RFC-001 Envelope + Layered Connection Model](https://github.com/usk6666/yorishiro-proxy/blob/main/docs/rfc/envelope.md) rewrite (milestones N1-N9). The data path was rebuilt around an `Envelope` + typed `Message` + `Layer` + `Channel` model that fixes the HTTP bias of the previous `Exchange`/`Codec` abstractions and unlocks structurally-honest support for HTTP/2 multiplexing, gRPC, gRPC-Web, WebSocket, SSE, and raw-byte smuggling diagnostics.

This release contained breaking changes for plugin authors and MCP clients. There is **no compatibility shim**; see the upstream CHANGELOG for the full Added / Changed / Removed / Migration sections.

## 0.14.x and earlier

Pre-RFC-001 history is preserved as git tags `v0.3.0` through `v0.14.1` and in the corresponding GitHub Releases. Detailed per-tag changelogs were not maintained at the time; consult `git log` in the upstream repo for fine-grained history.
