# SafetyFilter

SafetyFilter is a two-part content filtering system that prevents destructive payloads from being sent to target systems (Input Filter) and protects sensitive information from being exposed to AI agents (Output Filter). SafetyFilter rules are part of the Policy layer and **cannot be modified at runtime**.

SafetyFilter is **envelope-native**: rules operate on the typed `Envelope.Message` and the ordered `[]envelope.KeyValue` headers. The Pipeline `SafetyStep` dispatches via a type-switch on `env.Message` to per-protocol safety engines (`internal/rules/http`, `internal/rules/ws`, `internal/rules/grpc`); each engine reuses the regex / preset machinery in `internal/safety` for body, URL, query, and header matching.

## Input filter

The Input Filter inspects outgoing Send-direction messages (body, URL, query string, headers) against regex rules and blocks or logs matches before the message reaches the target. Receive-direction messages always pass through — the Input Filter is Send-only.

### How it works

When a message is about to be sent (via the proxy data path, resend, fuzz, macro, or intercept modify_and_forward), the Input Filter:

1. Evaluates the body, URL, query string, and headers against all input rules
2. If a rule matches with `block` action, the message is rejected with an error
3. If a rule matches with `log_only` action, the match is logged but the message proceeds

### Built-in presets

| Preset | Rules | Description |
|--------|-------|-------------|
| `destructive-sql` | 6 rules | DROP TABLE/DATABASE/INDEX/VIEW/SCHEMA, TRUNCATE TABLE, DELETE without WHERE, UPDATE WHERE 1=1, ALTER TABLE DROP, xp_ stored procedures |
| `destructive-os-command` | 5 rules | rm -rf, shutdown/reboot/halt/poweroff, mkfs, dd if=, Windows format |

### Input targets

| Target | Description |
|--------|-------------|
| `body` | Request body content |
| `url` | Full URL string |
| `query` | Query string portion of the URL |
| `header` | Individual header values (use `header:Name` for specific headers) |
| `headers` | All header values concatenated |

### Blocked response

When the Input Filter blocks a request at the proxy layer:

- **Status**: `403 Forbidden`
- **Header**: `X-Block-Reason: safety_filter`
- **Body**: JSON with violation details (rule ID, rule name, match location)

When it blocks an MCP tool operation:

- **MCP error response** with violation details

## Output filter

The Output Filter prevents sensitive information (PII) from being exposed to AI agents. It inspects response bodies and headers against regex rules and masks matching content before returning data to the agent.

### How it works

1. **Proxy layer**: Response body and headers are masked before returning to the client
2. **MCP tool layer**: Query results, resend responses, fuzz results, intercept queue entries, and export data are masked before returning to the AI agent
3. **Raw data preserved**: The Flow Store always contains the original unmasked data for human review via the Web UI

### Built-in PII presets

| Preset | Rules | Description |
|--------|-------|-------------|
| `credit-card` | 2 rules | Credit card numbers in separated (1234-5678-9012-3456) and continuous (1234567890123456) formats |
| `japan-my-number` | 1 rule | Japanese My Number (12-digit individual number) with check digit validation |
| `email` | 1 rule | Email addresses (user@example.com) |
| `japan-phone` | 2 rules | Japanese phone numbers in mobile (090-1234-5678) and landline (03-1234-5678) formats |

### Validators

Some presets use validator functions for additional verification beyond regex matching, reducing false positives:

- **credit-card (continuous)**: Luhn algorithm check -- only masks digit sequences that pass the Luhn checksum
- **japan-my-number**: Check digit validation -- only masks 12-digit sequences with a valid My Number check digit

### Output actions

| Action | Description |
|--------|-------------|
| `mask` | Replace matched content with the replacement string |
| `log_only` | Log the match but return data unmodified |

### Replacement strings

Replacement strings support regex capture group references:

| Syntax | Description |
|--------|-------------|
| `[MASKED:credit_card]` | Static replacement |
| `$1` | First capture group |
| `${name}` | Named capture group |

## Two-layer architecture

Like [Target scope](target-scope.md) and [Rate limits](rate-limits.md), SafetyFilter uses the Policy + Agent two-layer architecture:

- **Policy layer**: SafetyFilter rules are defined in the config file and are **immutable at runtime**
- **Agent layer**: AI agents can view rules via `get_safety_filter` but cannot modify them

This ensures that safety boundaries remain enforced regardless of agent behavior.

## Enforcement points

The Input Filter is enforced at every point where the proxy sends data to external targets:

| Tool | Enforcement |
|------|-------------|
| **Proxy data path** | `SafetyStep` runs on every Send-direction envelope, before transform and downstream encode |
| **resend_http / resend_ws / resend_grpc** | Typed message checked before sending |
| **resend_raw** | Raw bytes checked before sending |
| **fuzz_http / fuzz_ws / fuzz_grpc / fuzz_raw** | Template envelope checked at start; each expanded payload checked before sending |
| **macro** | Each step's outbound message checked before sending |
| **intercept** | `modify_and_forward` envelope re-checked before release (USK-702) |

### Re-check at HoldQueue release

When an operator holds a message via Intercept and chooses **modify-and-forward**, the user-modified envelope is re-evaluated by `SafetyStep` before being released downstream. The original held envelope already passed `SafetyStep` at hold time (the canonical chain runs `Safety → Intercept`), but `modify_and_forward` lets the operator inject content that would bypass that gate — for example, replacing a benign body with a `DROP TABLE` payload.

The re-check uses the same per-protocol engine as the inline check, so coverage is identical for HTTP, WebSocket, and gRPC. If the modified envelope trips the Input Filter, the release is rejected and the envelope is dropped.

The Output Filter is enforced at every point where data is returned to the AI agent:

| Source | Enforcement |
|--------|-------------|
| **Proxy responses** | Body and headers masked before returning to client |
| **query results** | Flow data masked in query output |
| **resend results** | Response body and headers masked |
| **fuzz results** | Response data masked |
| **export data** | Inline export data masked |

## Viewing current rules

Use the `security` tool to view the current SafetyFilter configuration:

```json
// security
{
  "action": "get_safety_filter"
}
```

Response:

```json
{
  "enabled": true,
  "input_rules": [
    {
      "id": "destructive-sql:drop",
      "name": "DROP statement",
      "pattern": "(compiled regex)",
      "targets": ["body", "url", "query"],
      "action": "block",
      "category": "destructive-sql"
    }
  ],
  "output_rules": [
    {
      "id": "credit-card:separated",
      "name": "Credit card (separated)",
      "pattern": "(compiled regex)",
      "targets": ["body"],
      "action": "mask",
      "replacement": "[MASKED:credit_card]",
      "category": "credit-card"
    }
  ],
  "immutable": true
}
```

## Custom rules

Custom rules are defined in the config file.

### Custom input rule

```json
{
  "id": "custom-api",
  "name": "Dangerous API endpoint",
  "pattern": "(?i)/api/v[0-9]+/(delete-all|reset)",
  "targets": ["url"]
}
```

### Custom output rule

```json
{
  "id": "custom-api-key",
  "name": "API key pattern",
  "pattern": "(sk-[a-zA-Z0-9]{32,})",
  "targets": ["body"],
  "action": "mask",
  "replacement": "[MASKED:api_key]"
}
```

## Related pages

- [Security tool reference](../tools/security.md) -- MCP tool parameter reference
- [Target scope](target-scope.md) -- Host-level access control
- [Rate limits & budgets](rate-limits.md) -- Rate limiting and budget controls
- [Security model](../concepts/security-model.md) -- Architecture-level security design
- [Config file](../configuration/config-file.md) -- SafetyFilter configuration
