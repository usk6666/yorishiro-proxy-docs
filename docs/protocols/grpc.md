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

## gRPC frame parsing

gRPC messages are framed using a 5-byte Length-Prefixed Message format:

```
+------------------+------------------+
| Compressed (1B)  | Length (4B, BE)  |  Payload (N bytes)
+------------------+------------------+
```

- **Compressed flag**: `0` = uncompressed, `1` = compressed (per `grpc-encoding` header)
- **Length**: 4-byte big-endian uint32 indicating the payload size
- **Payload**: the serialized Protocol Buffers (or JSON) message

The proxy parses all frames from both the request and response bodies. Each frame's payload is stored as a separate flow message, preserving the compressed flag in metadata.

## Streaming classification

The proxy classifies each gRPC session based on the number of request and response frames:

| Request frames | Response frames | Flow type |
|---------------|----------------|-----------|
| 0-1 | 0-1 | `unary` |
| >1 | >1 | `bidirectional` |
| Other combinations | | `stream` (client or server streaming) |

This classification covers all four gRPC streaming patterns:

- **Unary RPC**: single request, single response
- **Server streaming**: single request, multiple responses
- **Client streaming**: multiple requests, single response
- **Bidirectional streaming**: multiple requests, multiple responses

## Flow recording structure

A gRPC flow is recorded with:

- **Protocol**: `gRPC`
- **Flow type**: `unary`, `stream`, or `bidirectional`
- **State**: `complete`

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

The first send message carries the HTTP method, URL, and request headers. The first receive message carries the HTTP status code and response headers. The last receive message includes trailers (which carry `grpc-status` and `grpc-message`).

### gRPC status codes

The proxy extracts the gRPC status from trailers (or response headers for Trailers-Only responses):

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

## Related pages

- [HTTP/2](http2.md) -- the transport layer for gRPC
- [HTTPS MITM](https-mitm.md) -- TLS interception for gRPC over TLS
- [Plugin hook reference](../plugins/hook-reference.md) -- full hook documentation
