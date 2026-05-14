# manage

Manage flow data and CA certificates. Delete flows, export data in JSONL or HAR format, import previously exported flows, and regenerate the CA certificate.

## Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `action` | string | Yes | Action to perform: `delete_flows`, `export_flows`, `import_flows`, `regenerate_ca_cert` |
| `params` | object | Yes | Action-specific parameters (see below) |

## Actions

### delete_flows

Delete flows by ID, by age, by protocol, or all at once.

#### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `flow_id` | string | No | Delete a specific flow by ID |
| `older_than_days` | integer | No | Delete flows older than this many days (min 1). Requires `confirm` |
| `protocol` | string | No | Delete flows by canonical protocol family (`"http"`, `"ws"`, `"grpc"`, `"grpc-web"`, `"sse"`, `"raw"`, `"tls-handshake"`). Requires `confirm` |
| `scheme` | string | No | Delete flows by wire-observed handshake transport: `"http"`, `"https"`, `"tcp"`. Requires `confirm` |
| `http_version` | string | No | Delete flows by HTTP version (`"http/1.0"`, `"http/1.1"`, `"h2"`, `"h2c"`). Empty-string matches pre-USK-788 rows. Requires `confirm` |
| `filter` | object | No | Bulk filter (mutually exclusive with the top-level `protocol`/`scheme`/`http_version` axes). Supports `url_pattern`, `time_after`, `time_before` |
| `confirm` | boolean | No | Required for bulk deletion (age-based, filter-based, or all) |

One of `flow_id`, `older_than_days`, the top-level filter axes (with confirm), `filter` (with confirm), or `confirm` alone (for delete-all) must be specified. Top-level filter axes and `params.filter` cannot be combined in a single call.

#### Response

| Field | Type | Description |
|-------|------|-------------|
| `deleted_count` | integer | Number of flows deleted |
| `cutoff_time` | string | Cutoff time for age-based deletion (RFC3339) |

### export_flows

Export flows to JSONL or HAR (HTTP Archive 1.2) format with optional filtering.

#### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `format` | string | No | `"jsonl"` | Export format: `"jsonl"` or `"har"` |
| `filter` | object | No | | Flow filter criteria |
| `include_bodies` | boolean | No | `true` | Include message bodies in export |
| `output_path` | string | Conditional | | File path for output. Required for HAR; optional for JSONL (inline if omitted) |

Filter fields:

| Field | Type | Description |
|-------|------|-------------|
| `protocol` | string | Canonical protocol family (`"http"`, `"ws"`, `"grpc"`, `"grpc-web"`, `"sse"`, `"raw"`, `"tls-handshake"`) |
| `scheme` | string | Wire-observed handshake transport: `"http"`, `"https"`, `"tcp"`. Use `scheme=https` to export all TLS flows regardless of HTTP version |
| `http_version` | string | HTTP version (`"http/1.0"`, `"http/1.1"`, `"h2"`, `"h2c"`). Empty-string explicit value matches pre-USK-788 rows |
| `url_pattern` | string | Filter by URL substring |
| `time_after` | string | Include flows after this time (RFC3339) |
| `time_before` | string | Include flows before this time (RFC3339) |

**Format details:**

- **JSONL** -- each line is a complete JSON object. Supports inline and file output.
- **HAR** -- HTTP Archive 1.2 format compatible with browser DevTools, Burp Suite, and OWASP ZAP. File output only. Raw TCP and gRPC flows are excluded. WebSocket flows include `_webSocketMessages`.

#### Response

| Field | Type | Description |
|-------|------|-------------|
| `exported_count` | integer | Number of flows exported |
| `format` | string | Export format used |
| `output_path` | string | File path (if file output) |
| `data` | string | Exported data (if inline output) |

### import_flows

Import flows from a JSONL file.

!!! note "Path resolution"
    `input_path` (and `export_flows.output_path`) is resolved relative to the proxy server's working directory at the time the tool runs, not the MCP client's working directory. Use absolute paths when in doubt.

#### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `input_path` | string | Yes | | File path to read import data |
| `on_conflict` | string | No | `"skip"` | Conflict policy: `"skip"` or `"replace"` |

#### Response

| Field | Type | Description |
|-------|------|-------------|
| `imported` | integer | Number of flows imported |
| `skipped` | integer | Number of flows skipped |
| `errors` | integer | Number of errors |
| `source` | string | Source file path |
| `error_details` | array | Details of import errors (if any) |

### regenerate_ca_cert

Regenerate the CA certificate. Behavior depends on the CA initialization mode:

- **Auto-persist mode** (default) -- generates a new CA and saves to disk. You must re-install the certificate.
- **Ephemeral mode** (`--ca-ephemeral`) -- generates in memory only. Lost on restart.
- **Explicit mode** (`-ca-cert`/`-ca-key`) -- returns an error. User-provided CA files are not overwritten.

No parameters required.

#### Response

| Field | Type | Description |
|-------|------|-------------|
| `fingerprint` | string | SHA-256 fingerprint of the new CA |
| `subject` | string | CA subject |
| `not_after` | string | Expiration date |
| `persisted` | boolean | Whether the CA is saved to disk |
| `cert_path` | string | File path of the persisted CA |
| `install_hint` | string | Guidance for installing the CA certificate |

## Examples

### Delete a single flow

```json
// manage
{
  "action": "delete_flows",
  "params": {"flow_id": "abc-123"}
}
```

### Delete old flows

```json
// manage
{
  "action": "delete_flows",
  "params": {"older_than_days": 30, "confirm": true}
}
```

### Delete all flows

```json
// manage
{
  "action": "delete_flows",
  "params": {"confirm": true}
}
```

### Export all flows to file

```json
// manage
{
  "action": "export_flows",
  "params": {
    "format": "jsonl",
    "include_bodies": true,
    "output_path": "/tmp/export.jsonl"
  }
}
```

### Export flows to HAR file

```json
// manage
{
  "action": "export_flows",
  "params": {
    "format": "har",
    "filter": {
      "scheme": "https",
      "url_pattern": "/api/"
    },
    "output_path": "/tmp/api-flows.har"
  }
}
```

### Import flows

```json
// manage
{
  "action": "import_flows",
  "params": {
    "input_path": "/tmp/export.jsonl",
    "on_conflict": "skip"
  }
}
```

### Regenerate CA certificate

```json
// manage
{
  "action": "regenerate_ca_cert",
  "params": {}
}
```

## Related pages

- [Flow export & import](../features/flow-export-import.md) -- Export and import feature guide
- [CA certificate](../getting-started/ca-certificate.md) -- CA certificate setup
- [query](query.md) -- Query flow data
