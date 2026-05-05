# RFC-001 Documentation Migration Plan

**Source-repo tracking issue:** [USK-699](https://linear.app/usk6666/issue/USK-699)
**Status:** Tracking — actual rewrite happens in follow-up PRs (this repo)

> Issues are disabled on `yorishiro-proxy-docs`; coordination is via this PR.

## Background

The source repository (`usk6666/yorishiro-proxy`) completed the **RFC-001 Envelope + Layered Connection Model** rewrite over the N1-N9 milestones (2026-04-12 → 2026-05). The new architecture replaces the M36-M44-era `Codec` + `Exchange` data path with `Envelope` + `Message` + `Layer` + `Channel` + per-protocol rule engines + Plugin v2 (Starlark, `(protocol, event, phase)` 3-axis hook identity).

Authoritative source documents (track them via `src-ref/` after the upcoming submodule bump):

- RFC: `src-ref/docs/rfc/envelope.md` (Accepted 2026-04-12)
- Implementation guide: `src-ref/docs/rfc/envelope-implementation.md`
- Plugin migration table: `src-ref/docs/rfc/plugin-migration.md`
- CHANGELOG (Keep-a-Changelog 1.1.0): `src-ref/CHANGELOG.md`
- Repo-internal architecture summary: `src-ref/CLAUDE.md`

This documentation site currently mirrors the **pre-RFC-001** architecture and tool surface. It must be rewritten to match the v0.x release shipping after N9 closes (USK-700 cuts the tag).

## Scope of follow-up PRs

This PR opens the migration as a coordination signal. The actual rewrite is intentionally split across the following follow-up PRs (one PR per top-level rewrite area). Each follow-up uses the docs-repo branch convention `feat/<short-desc>`.

### 1. Architecture page rewrite

- **Action**: rewrite
- **Page**: `docs/concepts/architecture.md`
- **Source references**: `src-ref/docs/rfc/envelope.md` §3 (Core Concepts), §4 (canonical scenarios), `src-ref/CLAUDE.md` Architecture section
- **Key changes**:
  - Replace "Layer 4 TCP proxy ... protocol detection ... protocol handlers" framing with the **Layer + Channel + Envelope** model
  - Pipeline diagram: 8-step canonical chain (HostScope → HTTPScope → Safety → PluginPre → Intercept → Transform → Macro → PluginPost → Record)
  - Drop references to `internal/protocol/*` (deleted at N9); update to `internal/layer/*`
  - Drop the "Modular protocol handlers" framing; replace with "ConnectionStack — explicit stack of Layers per connection"
  - Surface the L7-first / L4-capable principle (raw bytes always preserved; L7 is an overlay)

### 2. Protocol ↔ Layer mapping section

- **Action**: new section (inside `docs/concepts/architecture.md` or a sibling page)
- **Source reference**: `src-ref/CLAUDE.md` "L7/L4 Support Status by Protocol" table
- **Content** (final shape):

  | Protocol | L7 view | L4 raw bytes | Layer package |
  |----|----|----|----|
  | HTTP/1.x | YES | YES | `internal/layer/http1/` |
  | HTTP/2 | YES | YES | `internal/layer/http2/` + `httpaggregator/` |
  | gRPC | YES | YES (via H2) | `internal/layer/grpc/` |
  | gRPC-Web | YES | YES | `internal/layer/grpcweb/` |
  | WebSocket | YES | YES (per frame) | `internal/layer/ws/` |
  | SSE | YES | YES | `internal/layer/sse/` |
  | Raw TCP | N/A | YES (byte stream) | `internal/layer/bytechunk/` |
  | TLS handshake | YES (observation) | N/A | `internal/layer/tls/` |
  | SOCKS5 | N/A (transport) | N/A | `internal/connector/socks5_handler.go` |

### 3. Plugin development guide rewrite

- **Action**: rewrite three pages, delete one
- **Pages**:
  - Rewrite: `docs/plugins/overview.md`, `docs/plugins/writing-plugins.md`, `docs/plugins/hook-reference.md`
  - Delete: `docs/plugins/codec-plugins.md` (codec plugin feature removed at N9)
  - Update: `docs/plugins/data-map-reference.md`, `docs/plugins/examples.md`
- **Source references**: `src-ref/docs/rfc/envelope.md` §9.3, `src-ref/docs/rfc/plugin-migration.md`, `src-ref/internal/pluginv2/`
- **Key concepts to surface**:
  - `(protocol, event, phase)` 3-axis hook identity (replaces 8 fixed hooks like `on_receive_from_client`)
  - `register_hook(protocol, event, fn, phase=)` Starlark builtin (replaces fixed-name hook functions)
  - 17-entry `(protocol, event)` surface table (RFC §9.3)
  - Two phase positions: `pre_pipeline` / `post_pipeline` / `none` (lifecycle)
  - `msg["raw"]` raw injection (mutate wire bytes alongside structured fields)
  - Ordered `Headers` (no canonicalization, lossless wire fidelity)
  - `ctx.transaction_state` (per request/response) / `ctx.stream_state` (per long-lived stream) — replaces global-state-only model
  - `action.RESPOND(...)` / `action.RESPOND_GRPC(...)` callable builtins (short-circuit response from plugin)
  - 6 lifecycle dict events: ConnectionConnect / Disconnect / SOCKS5Connect / TLSHandshake / WSClose / GRPCEnd
  - `plugin_introspect` MCP tool (introspect loaded plugins, hooks, last actions; redacted)

### 4. MCP tools nav refresh

- **Action**: split + delete + add
- **Pages**:
  - Delete: `docs/tools/resend.md`, `docs/tools/fuzz.md`, `docs/tools/plugin.md` (singular legacy tools removed at N9)
  - Add: `docs/tools/resend-http.md`, `docs/tools/resend-ws.md`, `docs/tools/resend-grpc.md`, `docs/tools/resend-raw.md`
  - Add: `docs/tools/fuzz-http.md`, `docs/tools/fuzz-ws.md`, `docs/tools/fuzz-grpc.md`, `docs/tools/fuzz-raw.md`
  - Add: `docs/tools/plugin-introspect.md`
  - Update: `docs/tools/overview.md`, `docs/tools/query.md` (canonical Protocol family filter: `http`/`ws`/`grpc`/`grpc-web`/`sse`/`raw`/`tls-handshake`), `docs/tools/intercept.md`
- **Source references**: `src-ref/internal/mcp/tool_*.go`, `src-ref/internal/mcp/resources/`, `src-ref/CHANGELOG.md`

### 5. Features page cleanup

- **Action**: delete + update
- **Pages**:
  - Delete: `docs/features/comparer.md` (Comparer feature removed at N9)
  - Update: `docs/features/intercept.md`, `docs/features/auto-transform.md`, `docs/features/safety-filter.md` to reflect per-protocol rule engine split (`internal/rules/{http,ws,grpc,sse,raw}/`)
  - Verify: `docs/features/target-scope.md` — note that CaptureScope was deleted in USK-705; HostScope is the live mechanism
- **Source references**: `src-ref/internal/rules/`, `src-ref/internal/safety/`

### 6. Protocols pages refresh

- **Action**: update (already partial coverage; align language with new Layer surface)
- **Pages**: `docs/protocols/{http,https-mitm,http2,grpc,websocket,sse,raw-tcp,socks5}.md`
- **Add**: `docs/protocols/grpc-web.md` (currently missing; was supported via N7)
- **Source references**: `src-ref/docs/rfc/envelope.md` §3.2 (per-protocol Message types), `src-ref/internal/layer/*`

### 7. mkdocs.yml nav update

- **Action**: update
- Remove: `tools/resend`, `tools/fuzz`, `tools/plugin` (singular), `features/comparer`, `plugins/codec-plugins`
- Add: 4 typed `resend_*`, 4 typed `fuzz_*`, `plugin_introspect`, `protocols/grpc-web`

### 8. Submodule bump (final step)

- **Action**: bump `src-ref` from `90b6382` (v0.14.0) to the v0.x release tag cut by USK-700
- **Timing**: do this AFTER the v0.x tag is published in source repo

## Acceptance criteria (this tracking PR)

- [x] Tracking PR open in `yorishiro-proxy-docs`
- [x] URL noted in USK-699 comment (source repo)
- [x] Zero code changes in `usk6666/yorishiro-proxy`

## Acceptance criteria (full rewrite — out of scope for this PR)

The rewrite work tracked here is intentionally split. Full rewrite is complete when:

- [ ] Architecture page reflects RFC-001 Layer/Channel/Envelope model + L7-first/L4-capable principle
- [ ] Protocol ↔ Layer mapping table present
- [ ] Plugin development guide reflects `(protocol, event, phase)` 3-axis identity + `msg["raw"]` injection + ordered Headers + `ctx.transaction_state`/`ctx.stream_state` + lifecycle events + `action.RESPOND*` builtins
- [ ] MCP tools section split into typed siblings (no `resend.md`/`fuzz.md`/`plugin.md` singular pages)
- [ ] `comparer.md` + `codec-plugins.md` deleted
- [ ] `mkdocs.yml` nav matches the above
- [ ] `protocols/grpc-web.md` added
- [ ] `src-ref` submodule bumped to v0.x.0 (post USK-700 tag cut)
- [ ] `mkdocs build` clean

## References

- RFC: https://github.com/usk6666/yorishiro-proxy/blob/rewrite/rfc-001/docs/rfc/envelope.md
- Plugin migration: https://github.com/usk6666/yorishiro-proxy/blob/rewrite/rfc-001/docs/rfc/plugin-migration.md
- CHANGELOG: https://github.com/usk6666/yorishiro-proxy/blob/rewrite/rfc-001/CHANGELOG.md
- N9 milestone (Linear): https://linear.app/usk6666/project/yorishiro-proxy?selectedMilestone=e7bd30d3-f190-4300-aac2-e60a442f776a
