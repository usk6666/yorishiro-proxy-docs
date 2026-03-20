# gRPC

yorishiro-proxy records gRPC traffic flowing over HTTP/2, extracting service and method names, parsing gRPC frames, and classifying streaming patterns. gRPC support is integrated into the HTTP/2 handler -- it is not a standalone protocol handler.

## How gRPC is detected

gRPC uses HTTP/2 as its transport. The proxy identifies gRPC traffic by checking the `Content-Type` header on each HTTP/2 stream:

- `application/grpc`
- `application/grpc+proto`
- `application/grpc+json`
- Any value matching `application/grpc+*`

When a matching content type is found, the HTTP/2 handler delegates flow recording to the gRPC handler while continuing to proxy the stream normally.

## Service and method extraction

gRPC encodes the service and method in the URL path following the pattern:

```
/package.ServiceName/MethodName
```

The proxy parses this path and stores the extracted service and method names as metadata on each flow message. For example, a request to `/helloworld.Greeter/SayHello` produces:

- `service`: `helloworld.Greeter`
- `method`: `SayHello`

If the path cannot be parsed (e.g., malformed URLs), the service and method are recorded as `unknown`.

## Protobuf framing

gRPC messages use a Length-Prefixed Message format with a 5-byte header:

```
+------------------+------------------+-------------------+
| Compressed (1B)  | Length (4B, BE)  |  Payload (N bytes) |
+------------------+------------------+-------------------+
```

- **Compressed flag** (1 byte): `0` = uncompressed, `1` = compressed (per `grpc-encoding` header)
- **Length** (4 bytes): big-endian uint32 indicating the payload size
- **Payload** (N bytes): the serialized Protocol Buffers (or JSON) message

The proxy parses all frames from both the request and response bodies. Each frame's payload is stored as a separate flow message, preserving the compressed flag in metadata.

## Progressive recording

Unlike HTTP/1.x flows that are recorded only after the full request/response cycle, gRPC flows use progressive (frame-by-frame) recording. This means you can see active gRPC streams in the flow list before they complete.

The lifecycle works as follows:

1. **Flow creation** -- when the first gRPC request arrives, a flow is created with `State="active"` and `FlowType="unary"` (the initial default)
2. **Initial send message** -- the request headers (method, URL, service, method) are recorded as the first send message (sequence 0)
3. **Frame-by-frame recording** -- as each gRPC frame arrives (from either client or server), it is immediately appended as a new flow message via `AppendMessage`
4. **Stream completion** -- when the stream terminates, the flow is updated to `State="complete"` with the final flow type and trailer metadata

Each recorded frame message includes:

| Metadata key | Description |
|-------------|-------------|
| `direction` | `"client_to_server"` or `"server_to_client"` |
| `sequence` | Message sequence number within the flow |
| `compressed` | `"true"` if the frame's compressed flag is set |
| `encoding` | `"protobuf"` |
| `grpc_encoding` | Compression algorithm (e.g., `"gzip"`) if present |

A per-stream message count limit prevents excessive memory use for very long-running streams. When the limit is reached, frames continue to be forwarded but are no longer recorded.

## Streaming transport

The proxy handles all four gRPC streaming patterns using bidirectional io.Pipe-based streaming. Instead of buffering the entire request body before forwarding (which would deadlock bidirectional streams), data is streamed as it arrives:

```
Client --> Read+Parse --> Pipe Writer --> Upstream
                |
                v
         FrameBuffer (progressive recording)

Upstream --> Read+Parse+Flush --> Client
     |
     v
FrameBuffer (progressive recording)
```

### Streaming classification

The proxy classifies each gRPC session based on the number of request and response frames:

| Request frames | Response frames | Flow type |
|---------------|----------------|-----------|
| 0-1 | 0-1 | `unary` |
| >1 | >1 | `bidirectional` |
| Other combinations | | `stream` (client or server streaming) |

This covers all four gRPC patterns:

- **Unary RPC**: single request, single response
- **Server streaming**: single request, multiple responses
- **Client streaming**: multiple requests, single response
- **Bidirectional streaming**: multiple requests, multiple responses

The flow type starts as `"unary"` when the flow is created and is updated to the final classification when the stream completes.

## Trailer handling

gRPC relies on HTTP/2 trailers to carry status information. The proxy records trailers in a dedicated final receive message with `grpc_type: "trailers"`.

### Standard trailers

In a normal gRPC response, trailers arrive as a HEADERS frame with `END_STREAM` after all DATA frames. The proxy extracts:

- `grpc-status` -- the gRPC status code (e.g., `0` for OK)
- `grpc-message` -- the error message (if any)

These values are recorded in the final receive message metadata and in the flow's tags as `grpc_status`.

### Trailers-only responses

A trailers-only response occurs when the server sends `grpc-status` in the initial response headers (no DATA frames). This happens for immediate errors like `UNIMPLEMENTED` or `PERMISSION_DENIED`.

The proxy detects trailers-only responses by checking whether `Grpc-Status` is present in the response headers (rather than in trailers). When detected, the final receive message includes:

```
"grpc_trailers_only": "true"
```

This `grpc_trailers_only` metadata field lets you distinguish trailers-only responses from normal responses where no data frames happened to be sent.

## Flow recording structure

A gRPC flow is recorded with:

- **Protocol**: `gRPC`
- **Flow type**: `unary`, `stream`, or `bidirectional`
- **State**: `active` during streaming, `complete` when finished

### Messages

Request frames are recorded as `send` messages, response frames as `receive` messages. Each message includes metadata:

**Send message metadata:**

| Key | Description |
|-----|-------------|
| `service` | gRPC service name (e.g., `helloworld.Greeter`) |
| `method` | gRPC method name (e.g., `SayHello`) |
| `grpc_encoding` | Compression encoding (e.g., `gzip`) if present |
| `compressed` | `"true"` if the frame's compressed flag is set |

**Receive message metadata:**

| Key | Description |
|-----|-------------|
| `service` | gRPC service name |
| `method` | gRPC method name |
| `grpc_status` | gRPC status code (e.g., `0` for OK) |
| `grpc_message` | gRPC error message if present |
| `grpc_encoding` | Compression encoding if present |
| `compressed` | `"true"` if the frame's compressed flag is set |

The first send message carries the HTTP method, URL, and request headers. The first receive message carries the HTTP status code and response headers. The last receive message (with `grpc_type: "trailers"`) includes trailers and the final `grpc-status`.

### Flow tags

Completed gRPC flows include the following tags:

| Tag | Description |
|-----|-------------|
| `streaming_type` | `"grpc"` |
| `grpc_service` | Service name |
| `grpc_method` | Method name |
| `grpc_status` | gRPC status code |
| `grpc_messages_recorded` | Total number of messages recorded |

### gRPC status codes

The proxy extracts the gRPC status from trailers (or response headers for trailers-only responses):

| Code | Name |
|------|------|
| 0 | OK |
| 1 | CANCELLED |
| 2 | UNKNOWN |
| 3 | INVALID_ARGUMENT |
| 4 | DEADLINE_EXCEEDED |
| 5 | NOT_FOUND |
| 6 | ALREADY_EXISTS |
| 7 | PERMISSION_DENIED |
| 8 | RESOURCE_EXHAUSTED |
| 9 | FAILED_PRECONDITION |
| 10 | ABORTED |
| 11 | OUT_OF_RANGE |
| 12 | UNIMPLEMENTED |
| 13 | INTERNAL |
| 14 | UNAVAILABLE |
| 15 | DATA_LOSS |
| 16 | UNAUTHENTICATED |

## Plugin hooks (observe-only)

The gRPC handler dispatches plugin hooks for each frame, but plugins **cannot modify or drop** gRPC traffic. All hooks are observe-only:

| Hook | Trigger | Notes |
|------|---------|-------|
| `on_receive_from_client` | For each request frame | Data includes service, method, body, compression flag |
| `on_receive_from_server` | For each response frame | Data includes grpc_status, grpc_message, trailers |

If a plugin returns a non-`CONTINUE` action (e.g., `DROP`), it is logged but ignored. This differs from HTTP/1.x and WebSocket where plugins can drop or modify traffic.

A per-session transaction context (`ctx`) is shared across all hook dispatches within the same gRPC session, allowing plugins to correlate request and response frames.

## Querying gRPC flows

```json
// query
{
  "resource": "flows",
  "filter": {"protocol": "gRPC"}
}
```

You can also filter by flow type to find streaming RPCs:

```json
// query
{
  "resource": "flows",
  "filter": {"protocol": "gRPC", "flow_type": "bidirectional"}
}
```

## Limitations

- **Observe-only** -- plugins cannot modify or drop gRPC frames (they can only observe)
- **No protobuf decoding** -- payloads are stored as raw bytes; the proxy does not decode Protocol Buffers messages
- **No decompression** -- compressed frames are stored in their compressed form; the `compressed` metadata flag indicates when decompression is needed
- **HTTP/2 only** -- gRPC-Web over HTTP/1.1 is not supported
- **Frame size limit** -- individual gRPC messages exceeding the configured maximum size are rejected
- **Message recording limit** -- very long-running streams stop recording after a per-stream message limit; forwarding continues unaffected

## Related pages

- [HTTP/2](http2.md) -- the transport layer for gRPC
- [HTTPS MITM](https-mitm.md) -- TLS interception for gRPC over TLS
- [Plugin hook reference](../plugins/hook-reference.md) -- full hook documentation
