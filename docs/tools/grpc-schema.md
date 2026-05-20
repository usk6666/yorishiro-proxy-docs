# grpc_schema

Manage `.proto` schemas for schema-aware gRPC decode (via [`query`](query.md)) and encode (via [`resend_grpc`](resend-grpc.md)).

Once a schema is registered, `query` decodes matching gRPC Data envelopes as protojson with the real `.proto` field names (`body_decoded_encoding="proto-json"`), and `resend_grpc` accepts `body_encoding="proto-json"` for the messages array. Schemaless fallback always applies when no schema matches the flow's `(grpc_service, grpc_method)`.

## Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `action` | string | Yes | One of `register`, `list`, `unregister`, `clear`, `discover` |
| `params` | object | Conditional | Action-specific parameters (see below) |

## Actions

### register

Upsert a service from a precompiled protobuf `FileDescriptorSet` (recommended) or a list of `.proto` files compiled by a host `protoc` binary. Last-write-wins: re-registering a service replaces every method's input/output binding.

#### Descriptor-set source (recommended)

Generate the descriptor set on the operator machine with:

```bash
protoc --include_imports --descriptor_set_out=schema.desc <protos>
```

The `--include_imports` flag is required -- without it transitive imports stay unresolved and `register` rejects the payload with an explicit error.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `source` | string | No | `"descriptor_set"` (default when omitted) |
| `descriptor_set_b64` | string | Yes | Base64-encoded FileDescriptorSet payload. Max 16 MiB after base64 decode |
| `service_filter` | string[] | No | Fully-qualified service names to register. Empty registers every service in the descriptor. Names not found in the descriptor produce an error |
| `source_label` | string | No | Free-form label preserved in `list` output (filename hint, version tag, etc.) |

This is the recommended path because it does not require `protoc` on the proxy host.

```json
// grpc_schema
{
  "action": "register",
  "params": {
    "source": "descriptor_set",
    "descriptor_set_b64": "Cg...base64...",
    "service_filter": ["pkg.Greeter"],
    "source_label": "greeter@v1.2.0"
  }
}
```

#### File source (host protoc)

The proxy invokes a host `protoc` binary to compile a list of absolute `.proto` paths into a FileDescriptorSet.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `source` | string | Yes | `"file"` |
| `proto_paths` | string[] | Yes | Absolute paths to `.proto` files. Each path must be canonical (no `..`, no double-slashes) and resolve under the proxy's CWD or one of `import_paths`. Symlinks are followed and the resolved target must also fall under the allowed roots |
| `import_paths` | string[] | No | Absolute `-I` roots for `protoc`. Defaults to the parent directory of each proto path |
| `service_filter` | string[] | No | Same semantics as the descriptor-set source |
| `source_label` | string | No | Defaults to a comma-joined list of `.proto` basenames |

`protoc` is invoked with `--include_imports` and a 30-second timeout. The environment is reduced to `PATH` only. By default the binary is resolved against `PATH` as `protoc`; configure an explicit path via `ProxyConfig.GRPCSchema.ProtocBinary` (config file) or the `YP_PROTOC_BINARY` environment variable. If `protoc` is missing, the call returns an install hint.

```json
// grpc_schema
{
  "action": "register",
  "params": {
    "source": "file",
    "proto_paths": ["/abs/path/to/greeter.proto"],
    "import_paths": ["/abs/path/to/protos"],
    "service_filter": ["pkg.Greeter"]
  }
}
```

Returns: `registered[]` -- list of `{ service, methods: [{name, input, output}], source_label, registered_at }`.

### discover

Probe a target gRPC server's reflection endpoint and register every service it exposes. Mirrors `grpcurl -plaintext <addr> list` semantics but runs from inside the proxy so the same TLS / mTLS / upstream-proxy / Target Scope / rate-limit / budget gates that protect resend traffic apply here too.

The proxy opens an outbound bidi gRPC stream to `grpc.reflection.v1.ServerReflection/ServerReflectionInfo`. If the server returns gRPC status `UNIMPLEMENTED` (12), the proxy retries once against the legacy v1alpha service path. Any other failure surfaces verbatim.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `target_addr` | string | Yes | Upstream `host` or `host:port` exposing the reflection endpoint |
| `scheme` | string | No | `"http"` (h2c) or `"https"` (TLS + ALPN h2). Defaults to `"https"` |
| `service_filter` | string[] | No | Restrict the fetch to these fully-qualified service names. Each entry must appear in the target's `ListServices` reply |
| `metadata` | array | No | Ordered gRPC metadata list (`[{"name": "...", "value": "..."}]`) forwarded on the reflection stream. Useful for reflection endpoints gated by a bearer token |
| `timeout_ms` | integer | No | Per-call timeout covering dial+handshake+RPC. Defaults to `30000`; capped at `300000` |

The proxy reuses the same `TLSTransport` as `resend_grpc`, so any configured mTLS or upstream-proxy applies automatically. `source_label` on the registered entry is `reflection://<target_addr>` so `list` distinguishes reflection-discovered schemas.

```json
// grpc_schema
{
  "action": "discover",
  "params": {
    "target_addr": "127.0.0.1:9000",
    "scheme": "http",
    "service_filter": ["pkg.Greeter"]
  }
}
```

When the target lacks reflection support (both v1 and v1alpha return UNIMPLEMENTED), the call fails with an error listing remediation hints (for grpc-go: `reflection.Register(s)`).

Returns: `{ discovered[], target, reflection_version }`. `reflection_version` is `"v1"` or `"v1alpha"` (informational).

!!! note
    The reflection chatter itself is **not** persisted as a Flow -- schema management is control-plane and the registered service's `source_label` already records the source.

### list

Return every currently registered schema, ordered alphabetically by service name.

```json
// grpc_schema
{ "action": "list" }
```

Returns: `schemas[]` -- same entry shape as `register`'s `registered[]`.

### unregister

Remove a single service from the registry. Methods owned by that service stop matching the schema-aware path; the schemaless fallback resumes.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `service` | string | Yes | Fully-qualified service name to remove |

```json
// grpc_schema
{
  "action": "unregister",
  "params": {"service": "pkg.Greeter"}
}
```

Returns: `{ service, unregistered }`.

### clear

Remove every registered schema.

```json
// grpc_schema
{ "action": "clear" }
```

Returns: `{ cleared }` -- number of rows deleted from persistent storage.

## Decode path (query)

When [`query`](query.md) is invoked with `decode_bodies=true` (the default) and the flow's metadata carries `grpc_service` + `grpc_method`, the proxy:

1. Looks up the `(service, method)` pair in the registered schemas.
2. **Hit**: decodes the body via `protoreflect.DynamicMessage` + `protojson`, returns `body_decoded_encoding="proto-json"` with real field names. The marshaller uses `UseProtoNames=true` and `EmitUnpopulated=false`, so the output matches the `.proto` definition.
3. **Hit, parse failure**: falls back to the schemaless path *and* surfaces `body_decode_anomaly { type: "proto_schema_mismatch" }` so the caller can correlate the failure.
4. **Miss**: falls back to the schemaless decode (`body_decoded_encoding="proto-schemaless-json"`).

Output Filter (PII masking) applies identically to the protojson output.

## Encode path (resend_grpc)

[`resend_grpc`](resend-grpc.md) gains a new `body_encoding="proto-json"` value:

```json
// resend_grpc
{
  "flow_id": "<recorded gRPC stream>",
  "messages": [
    {
      "body_encoding": "proto-json",
      "payload": "{\"f_string\":\"hello\",\"f_int32\":42}"
    }
  ]
}
```

The payload JSON is parsed against the registered schema's input descriptor for `(service, method)`, encoded to proto wire bytes via `proto.Marshal`, then LPM-framed by the gRPC Layer.

`protojson.UnmarshalOptions{DiscardUnknown:true}` is applied -- JSON keys not in the schema are silently dropped, so AI-typoed field names produce a message with the field absent rather than a hard error. Type mismatches (string for int32, etc.) still fail.

If no schema is registered for the target service, `body_encoding="proto-json"` returns a hard error pointing at the `grpc_schema register` command.

### Lossy round-trip warning

`protojson.Marshal` drops fields not in the schema. A decode -> user edit -> encode round-trip via `proto-json` therefore **loses any wire field not present in the registered `.proto`** -- both unknown fields and protocol-level extensions.

When `flow_id` is supplied, `resend_grpc` inspects the source flow's bytes against the registered schema and emits a non-fatal `warnings[]` entry if the source carried fields outside the schema:

```text
source flow message 0 carries fields not in the registered schema;
proto-json round-trip will drop those bytes -- consider body_encoding=base64
or proto-schemaless-json for lossless preservation
```

For lossless round-trips, use `body_encoding="proto-schemaless-json"` (synthetic-key JSON via `internal/encoding/protobuf`) or `body_encoding="base64"`.

## Caveats and limits

- **Last-write-wins.** Re-registering a service replaces its previous binding. The `list` output shows the most recent registration for each service.
- **Persistence.** Registrations survive process restart via the `grpc_schemas` SQLite table.
- **`source="file"` requires a host `protoc`.** The recommended path is `source="descriptor_set"`. Use the file path only when invoking `protoc` server-side is acceptable. Resolution order: `YP_PROTOC_BINARY` -> `ProxyConfig.GRPCSchema.ProtocBinary` -> `PATH:protoc`.
- **`discover` reflection chatter is not recorded as a Flow.** Use `resend_grpc` if you need to round-trip a recorded reflection RPC for analysis.
- **Case-sensitive lookup.** Protobuf service/method identifiers are case-sensitive. The lookup against `Flow.Metadata["grpc_service"]` / `["grpc_method"]` is exact-match.
- **Output Filter still applies.** PII patterns configured in the safety engine mask the protojson output before it leaves the MCP boundary.

## Related pages

- [query](query.md) -- consumes the registered schema for `proto-json` decoding
- [resend_grpc](resend-grpc.md) -- accepts `body_encoding="proto-json"`
- [gRPC](../protocols/grpc.md) -- gRPC Layer overview
