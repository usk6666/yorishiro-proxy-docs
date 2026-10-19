# Flow export and import

You can export recorded flows to share with other tools or team members, and import flows from previous sessions. yorishiro-proxy supports three export formats: JSONL, HAR 1.2, and cURL commands.

## Export directory

File paths in `export_flows` (`output_path`) and `import_flows` (`input_path`) are confined to the server's **export directory** — by default `~/.yorishiro-proxy/exports`, configurable with the [`-export-dir`](../configuration/cli-flags.md#export-directory-export-dir) server flag or `YP_EXPORT_DIR`.

- Relative paths resolve against the export directory, not against your working directory or the server's.
- Absolute paths are accepted only if they lie inside the export directory. Anything else — `../` escapes, symlinks pointing outside, absolute paths elsewhere — is rejected.
- The response's `output_path` (and the import result's `source`) is the absolute path of the file.
- `query` with `resource: "status"` reports the directory as `export_dir`.

This prevents an MCP caller from overwriting operator-owned files such as the policy file, config file, CA key pair, or flow database through an export.

## Export formats

### JSONL format

JSONL (JSON Lines) is the native export format. Each line is a complete JSON object containing a flow and its messages. This format supports all protocols and flow types, including raw TCP and gRPC.

```json
// manage
{
  "action": "export_flows",
  "params": {
    "format": "jsonl",
    "output_path": "flows.jsonl"
  }
}
```

When `output_path` is omitted, small exports (up to 100 flows) are returned inline in the response:

```json
// manage
{
  "action": "export_flows",
  "params": {
    "format": "jsonl"
  }
}
```

### HAR 1.2 format

HAR (HTTP Archive) 1.2 is a standard format compatible with browser DevTools, Burp Suite, and OWASP ZAP. HAR export always requires a file path:

```json
// manage
{
  "action": "export_flows",
  "params": {
    "format": "har",
    "output_path": "flows.har"
  }
}
```

!!! note "HAR limitations"
    HAR is an HTTP-specific format. Raw TCP and gRPC flows are excluded from HAR exports. WebSocket flows include a `_webSocketMessages` custom field. Binary bodies are Base64-encoded with `content.encoding: "base64"`.

### cURL command format

Export individual flows as cURL commands using the `query` tool:

```json
// query
{
  "action": "curl",
  "params": {
    "flow_id": "abc-123"
  }
}
```

## Filtering exports

Apply filters to export a subset of flows:

```json
// manage
{
  "action": "export_flows",
  "params": {
    "format": "jsonl",
    "filter": {
      "protocol": "HTTPS",
      "url_pattern": "/api/",
      "time_after": "2026-02-01T00:00:00Z",
      "time_before": "2026-02-28T23:59:59Z"
    },
    "output_path": "api-flows.jsonl"
  }
}
```

### Filter fields

| Field | Type | Description |
|-------|------|-------------|
| `protocol` | string | Filter by protocol (e.g. `"HTTPS"`, `"HTTP/1.x"`, `"WebSocket"`) |
| `url_pattern` | string | Filter by URL substring |
| `time_after` | string | Include flows after this time (RFC3339 format) |
| `time_before` | string | Include flows before this time (RFC3339 format) |

## Metadata-only export

Export flow metadata without message bodies for a lightweight overview:

```json
// manage
{
  "action": "export_flows",
  "params": {
    "format": "jsonl",
    "include_bodies": false,
    "output_path": "flows-meta.jsonl"
  }
}
```

## Importing flows

Import flows from a JSONL file. Each line must be a valid export record with version `"1"`:

```json
// manage
{
  "action": "import_flows",
  "params": {
    "input_path": "flows.jsonl"
  }
}
```

### Conflict handling

When importing flows with IDs that already exist in the store, use `on_conflict` to control behavior:

| Policy | Description |
|--------|-------------|
| `skip` | Skip flows with existing IDs (default) |
| `replace` | Delete existing flows and re-import |

```json
// manage
{
  "action": "import_flows",
  "params": {
    "input_path": "flows.jsonl",
    "on_conflict": "replace"
  }
}
```

### Import result

The import action returns:

| Field | Description |
|-------|-------------|
| `imported` | Number of flows successfully imported |
| `skipped` | Number of flows skipped due to conflicts |
| `errors` | Number of flows that failed to import |
| `source` | Absolute path of the import file |
| `error_details` | Details of any import errors |

## Practical use cases

### Export for external analysis

Export HTTPS flows to HAR for analysis in Burp Suite:

```json
// manage
{
  "action": "export_flows",
  "params": {
    "format": "har",
    "filter": {"protocol": "HTTPS"},
    "output_path": "analysis.har"
  }
}
```

### Session backup

Export all flows from a testing session:

```json
// manage
{
  "action": "export_flows",
  "params": {
    "format": "jsonl",
    "output_path": "session-backup.jsonl"
  }
}
```

### Restore a session

Import a previous session's flows:

```json
// manage
{
  "action": "import_flows",
  "params": {
    "input_path": "session-backup.jsonl",
    "on_conflict": "skip"
  }
}
```

### Time-scoped export

Export flows from a specific time window:

```json
// manage
{
  "action": "export_flows",
  "params": {
    "format": "jsonl",
    "filter": {
      "time_after": "2026-03-14T09:00:00Z",
      "time_before": "2026-03-14T17:00:00Z"
    },
    "output_path": "today.jsonl"
  }
}
```

## Related pages

- [Manage tool reference](../tools/manage.md) -- MCP tool parameter reference
- [Flows](../concepts/flows.md) -- Flow data model
- [Query tool reference](../tools/query.md) -- cURL export via query
