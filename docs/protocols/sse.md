# SSE (Server-Sent Events)

yorishiro-proxy detects and handles Server-Sent Events (SSE) responses at the event level. The SSE Layer (`internal/layer/sse/`) wraps the response side of an HTTP/1.x exchange whose body is a `text/event-stream` and re-shapes the byte stream into one `SSEMessage` envelope per parsed event.

L7 structured view: yes. L4 raw bytes: yes. Per-event envelopes flow through a streaming-aware Pipeline.

## How detection works

SSE detection happens at the HTTP response level. When the upstream server returns a response with `Content-Type: text/event-stream`, the connector swaps the response Channel into the SSE Layer. The MIME type check ignores parameters like `charset`, so `text/event-stream; charset=utf-8` is also detected. The SSE Layer itself contains no detection logic — it is a Channel adapter.

SSE is not a separate transport — it rides on top of HTTP/1.x (or HTTPS via MITM). The request phase flows through the [HTTP/1.x](http.md) Layer normally, and the response side switches to event-level streaming.

```
Client ──HTTP Request──> yorishiro-proxy ──HTTP Request──> Upstream Server
       <──SSE Events───                 <──SSE Events───
```

## Event parsing

The proxy parses SSE events according to the [HTML Living Standard](https://html.spec.whatwg.org/multipage/server-sent-events.html). Each event is delimited by a blank line and can contain the following fields:

| Field | Description |
|-------|-------------|
| `event:` | Event type name (default is "message" if omitted) |
| `data:` | Event payload (multiple `data:` lines are joined with newlines) |
| `id:` | Last event ID for reconnection |
| `retry:` | Reconnection interval hint (milliseconds) |

Lines starting with `:` are treated as comments and silently consumed. Unknown fields are ignored per the specification.

### Size limits

- **Single event**: capped at 1 MB of raw bytes to prevent memory exhaustion
- **Events per stream**: up to 10,000 events are recorded per stream; beyond this limit, events are still forwarded to the client but no longer saved to the flow store
- **Event body recording**: event data is truncated at 254 MB per event in the flow store
- **Stream duration**: a safety cap of 24 hours prevents indefinite resource consumption

## Event-level recording

Each parsed SSE event is emitted as a separate envelope with `Direction=Receive` and `Envelope.Protocol=sse`. The first call to `Next` returns a clone of the original response Envelope with Protocol overridden to `ProtocolSSE` (preserving Sequence). Subsequent `Next` calls drive the SSE parser over the body reader and emit one envelope per parsed event with `Message=*envelope.SSEMessage` and `Envelope.Raw` containing the wire bytes of the event (including the trailing blank line).

`Send` is a programmer error on the SSE Layer; it returns the sentinel `ErrSendUnsupported`. SSE is half-duplex server→client.

### Flow structure

An SSE flow is recorded with:

- **Protocol**: `sse`
- **Flow type**: `stream`
- **State**: `active` while the stream is open, `complete` when it ends

### Event message metadata

Each event message includes the following metadata fields:

| Key | Value |
|-----|-------|
| `sse_type` | `"event"` for SSE events, `"headers"` for the initial response headers |
| `sse_event` | Event type from the `event:` field (omitted if empty) |
| `sse_id` | Event ID from the `id:` field (omitted if empty) |
| `sse_retry` | Retry value from the `retry:` field (omitted if empty) |

The event `Data` is stored on the `SSEMessage` Message struct, and the original wire-format bytes (including the trailing blank line) are preserved on `Envelope.Raw`.

## Intercept support

SSE supports intercept at two levels:

### Response-level intercept

When the SSE response matches intercept rules, the entire response is held for review before any events are streamed. At this level:

- **Release**: the stream proceeds normally
- **Drop**: the stream is terminated and the client receives a 502 error
- **Modify**: not supported at the response level (the body is a stream); treated as release with a warning

### Event-level intercept

Individual SSE events can be intercepted and held for review. All three actions are supported:

- **Release**: the event is forwarded unchanged
- **Drop**: the event is silently skipped (not forwarded to the client)
- **Modify**: the event data and metadata can be modified before forwarding

Event metadata is exposed as pseudo-headers for intercept rule matching:

| Header | Maps to |
|--------|---------|
| `X-SSE-Event` | Event type (`event:` field) |
| `X-SSE-Id` | Event ID (`id:` field) |
| `X-SSE-Retry` | Retry value (`retry:` field) |

### Variant tracking

When an event is modified by intercept or a plugin, both the original and modified versions are recorded:

- **Sequence N**: original event (metadata `variant=original`)
- **Sequence N+1**: modified event (metadata `variant=modified`)

Unmodified events are recorded as a single message without variant metadata.

## Plugin hooks

The SSE Layer participates in the standard Pipeline. Plugin authors register hooks via the pluginv2 engine using the `(protocol, event, phase)` triple — for example `(sse, on_event, pre)` to observe each event envelope before intercept. See the [Plugin hook reference](../plugins/hook-reference.md) for the full surface.

## Safety filter

The output filter (PII masking) is applied per-event to the SSE `Data` field before forwarding to the client:

- **Mask rules**: matching data is replaced with masked values in the forwarded event; the flow store always records the original (unfiltered) data
- **Block rules**: if a block-action rule matches an event's data, the entire SSE stream is terminated immediately

When events are displayed in the intercept queue, the output filter is also applied so that the reviewing agent sees masked data.

## Auto-transform

Response auto-transform rules are **not applied** to SSE streams. Auto-transform requires buffering the full response body in memory, which is not possible for a streaming response.

## Query examples

### Find SSE flows

The canonical Envelope.Protocol value for SSE is `sse`:

```json
// query
{
  "resource": "flows",
  "filter": {"protocol": "sse"}
}
```

### Get SSE flow details

```json
// query
{
  "resource": "flow",
  "id": "flow-abc-123"
}
```

The response includes `flow_type: "stream"`, `message_count` with the total number of recorded events, and `message_preview` with the first 10 messages.

### Page through SSE events

```json
// query
{
  "resource": "messages",
  "id": "flow-abc-123",
  "limit": 50,
  "offset": 0
}
```

### Filter receive-only messages

```json
// query
{
  "resource": "messages",
  "id": "flow-abc-123",
  "filter": {"direction": "receive"},
  "limit": 50
}
```

## Processing pipeline

Each SSE event passes through the following steps in order:

1. **Parse** -- `SSEParser.Next()` reads and parses the next event from the stream
2. **Plugin hook** -- `on_receive_from_server` (event-level)
3. **Snapshot** -- original event data is captured for variant tracking
4. **Intercept** -- event is checked against intercept rules; if matched, held for review (release / drop / modify)
5. **Plugin hook** -- `on_before_send_to_client` (event-level)
6. **Record** -- event is saved to the flow store (with variant if modified)
7. **Output filter** -- PII masking / block rules applied to the data
8. **Forward** -- the (possibly filtered) event bytes are written to the client

## Limitations

- **Receive-only Layer** -- SSE is half-duplex server→client; `Send` on the SSE Channel returns `ErrSendUnsupported`
- **No auto-transform** -- response auto-transform rules require full body buffering and are skipped for SSE
- **Response-level modify** -- `modify_and_forward` is not supported at the response header level for SSE (treated as release)

## Related pages

- [HTTP/1.x](http.md) -- the HTTP transport that SSE rides on
- [HTTPS MITM](https-mitm.md) -- SSE over HTTPS connections
- [Intercept](../features/intercept.md) -- intercept rules and the review queue
- [SafetyFilter](../features/safety-filter.md) -- output filter rules for PII masking
- [Plugin hook reference](../plugins/hook-reference.md) -- detailed hook documentation
