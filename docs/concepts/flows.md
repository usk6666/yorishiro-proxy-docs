# Flows

A **flow** is the fundamental data unit in yorishiro-proxy. It represents a single recorded exchange between a client and a server -- typically one request and one response, but streaming protocols may contain many messages. This page explains the flow data model, lifecycle, and how flows behave across different protocols.

## What is a flow?

A flow captures the complete lifecycle of a proxied exchange:

- **Connection metadata** -- client address, server address, connection ID, TLS details
- **Protocol information** -- detected protocol type (HTTP/1.x, HTTPS, gRPC, etc.)
- **Messages** -- the actual request and response data, including headers, bodies, and raw bytes
- **Timing** -- per-phase timing (send, wait/TTFB, receive) and total duration
- **Tags** -- key-value metadata such as error details, smuggling detection results, or SOCKS5 tunnel info

Every flow has a unique ID and belongs to a connection identified by `conn_id`. You can query flows by protocol, method, URL pattern, status code, host, state, and more using the `query` MCP tool.

## Flow types

Flows are classified by their communication pattern:

| Flow type | Description | Protocols |
|-----------|-------------|-----------|
| `unary` | Single request followed by a single response | HTTP/1.x, HTTPS, HTTP/2 (simple requests) |
| `stream` | One direction sends multiple messages | gRPC server streaming |
| `bidirectional` | Both directions send multiple messages | WebSocket, Raw TCP, gRPC bidirectional streaming |

For unary flows, there are exactly two messages: one `send` (request, sequence 0) and one `receive` (response, sequence 1). For streaming flows, messages are numbered sequentially and each has a direction (`send` or `receive`).

## Flow lifecycle

Every flow transitions through a simple state machine:

```mermaid
statediagram-v2
    [*] --> active: Request recorded
    active --> complete: Response recorded
    active --> error: Upstream failure
```

| State | Meaning |
|-------|---------|
| `active` | The request has been recorded, but the response has not yet arrived. The flow is in progress. |
| `complete` | The full exchange has been recorded. Both request and response (or all stream messages) are available. |
| `error` | The upstream connection failed (e.g., connection refused, TLS error, timeout). The request is preserved; the error details are stored in the flow's tags. |

### Progressive recording

yorishiro-proxy uses a **progressive recording** pattern rather than waiting for the complete exchange before writing to the store:

1. **Send phase** -- The request message is recorded and the flow is created with `state: "active"` *before* the request is forwarded upstream. This ensures the request is preserved even if the upstream server never responds.

2. **Receive phase** -- When the response arrives and has been relayed to the client, the response message is appended and the flow is updated to `state: "complete"` with timing data.

3. **Error phase** -- If the upstream connection fails, the flow is updated to `state: "error"` with the error message stored in the flow's tags. The original request remains intact.

This design means you can see in-progress requests in real time by filtering for `state: "active"` flows. It also guarantees that no request data is lost due to upstream failures.

## Messages

Each flow contains one or more **messages**. A message represents a single directional data unit:

| Field | Description |
|-------|-------------|
| `direction` | `"send"` (client to server) or `"receive"` (server to client) |
| `sequence` | Order within the flow (0-based) |
| `headers` | HTTP-style headers (may be nil for non-HTTP protocols) |
| `body` | Message body content |
| `raw_bytes` | Original bytes as captured on the wire, preserving header ordering and whitespace |
| `method` | HTTP method (send messages only) |
| `url` | Request URL (send messages only) |
| `status_code` | HTTP status code (receive messages only) |
| `metadata` | Protocol-specific key-value pairs |

### Raw bytes

The `raw_bytes` field preserves the exact bytes as they appeared on the wire. This is critical for security testing scenarios like HTTP request smuggling analysis, where header ordering, whitespace, and line endings matter. The `resend` tool's `resend_raw` action uses these bytes to replay requests byte-for-byte.

### Variant messages

When a request or response is modified by intercept rules or auto-transform, the flow records both the original and modified versions as **variant messages**:

- Sequence 0: `variant: "original"` -- the request as received from the client
- Sequence 1: `variant: "modified"` -- the request as actually sent upstream

This lets you compare what the client sent versus what the proxy forwarded, which is useful for understanding how intercept and transform rules affected the traffic.

## Streaming protocols

### WebSocket

WebSocket flows have `flow_type: "bidirectional"`. After the HTTP upgrade handshake, each WebSocket frame is recorded as a separate message with metadata including the frame opcode (`Text`, `Binary`, `Close`, `Ping`, `Pong`). The `protocol_summary` includes `message_count` and `last_frame_type`.

### gRPC

gRPC flows are built on top of HTTP/2. Each gRPC call is a separate flow with metadata extracted from the gRPC headers: `service`, `method`, `grpc_status`, and `grpc_status_name`. The `protocol_summary` includes these fields for quick filtering without inspecting individual messages.

### Raw TCP

Raw TCP flows capture any traffic that does not match a recognized protocol. Each direction's data chunks are recorded as separate messages with `send_bytes` and `receive_bytes` in the `protocol_summary`. TCP forwarding mappings (`tcp_forwards`) let you route specific local ports to upstream addresses for targeted capture.

## Flow metadata

### Tags

Tags are key-value string pairs attached to a flow. They carry contextual information that does not fit into the fixed schema:

| Tag | Set by | Purpose |
|-----|--------|---------|
| `error` | Error recording | Upstream failure details |
| `socks5_target` | SOCKS5 handler | Original target address from SOCKS5 request |
| `socks5_auth_method` | SOCKS5 handler | Authentication method used |
| `socks5_auth_user` | SOCKS5 handler | Authenticated username |
| `technologies` | Technology detection | Detected server technologies (JSON) |
| `smuggling_*` | Smuggling detector | HTTP request smuggling indicators |

### Timing

Each flow records per-phase timing when available:

| Field | Description |
|-------|-------------|
| `send_ms` | Time to send the request (headers + body) to the upstream server |
| `wait_ms` | Server processing time (time to first byte of response) |
| `receive_ms` | Time to receive the response body from the upstream server |
| `duration` | Total flow duration from start to completion |

These timing fields are useful for identifying slow endpoints, comparing response times across fuzz results, and detecting anomalies.

### Connection info

The `conn_info` object on each flow records network-level details:

| Field | Description |
|-------|-------------|
| `client_addr` | Client's remote address (e.g., `192.168.1.100:54321`) |
| `server_addr` | Upstream server's resolved address (e.g., `93.184.216.34:443`) |
| `tls_version` | Negotiated TLS version (e.g., `TLS 1.3`) |
| `tls_cipher` | Negotiated cipher suite |
| `tls_alpn` | Negotiated ALPN protocol (e.g., `h2`, `http/1.1`) |
| `tls_server_cert_subject` | Subject DN of the upstream server's TLS certificate |

## Querying flows

You interact with flows through the `query` MCP tool:

```json
// query
{
  "resource": "flows",
  "filter": {"protocol": "HTTPS", "method": "POST"},
  "limit": 20
}
```

For streaming flows, the `flow` resource returns a `message_preview` with the first 10 messages. Use the `messages` resource with `limit` and `offset` to page through all messages:

```json
// query
{
  "resource": "messages",
  "id": "flow-abc-123",
  "filter": {"direction": "send"},
  "limit": 50,
  "offset": 0
}
```

## Related pages

- [Architecture](architecture.md) -- How flows fit into the processing pipeline
- [MCP-first design](mcp-first-design.md) -- Querying and manipulating flows through MCP tools
- [Flow export & import](../features/flow-export-import.md) -- Exporting flows as JSONL, HAR, or cURL
