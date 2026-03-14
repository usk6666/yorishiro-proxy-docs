# Retention

yorishiro-proxy can automatically clean up old flows to manage database size. The retention system runs as a background task that periodically removes flows exceeding the configured limits.

## Settings overview

| Setting | JSON field | Type | Default | Description |
|---------|-----------|------|---------|-------------|
| Max flows | `retention_max_flows` | `int` | `0` (unlimited) | Maximum number of flows to retain |
| Max age | `retention_max_age` | `duration` | `0` (unlimited) | Maximum age of flows to retain |
| Cleanup interval | `cleanup_interval` | `duration` | `1h` | Interval between automatic cleanup runs |

All three settings are configured at the application level (not in the proxy config file). They are set via the `Config` struct fields and take effect at startup.

## How it works

The flow cleaner is a background goroutine that runs at the configured `cleanup_interval`. On each run, it:

1. **Removes excess flows** -- If `retention_max_flows` is set and the total flow count exceeds the limit, the oldest flows are deleted first.
2. **Removes expired flows** -- If `retention_max_age` is set, flows older than the specified duration are deleted.

The cleaner only starts if at least one retention policy is configured (`retention_max_flows > 0` or `retention_max_age > 0`). When neither is set, no cleanup runs and all flows are retained indefinitely.

## Duration format

Duration values use Go's duration string format:

| Suffix | Meaning | Example |
|--------|---------|---------|
| `s` | Seconds | `30s` |
| `m` | Minutes | `15m` |
| `h` | Hours | `24h` |

You can combine units: `1h30m`, `2h45m`, `72h`.

## Configuration examples

### Limit by flow count

Keep only the most recent 10,000 flows:

```json
{
  "retention_max_flows": 10000
}
```

### Limit by age

Remove flows older than 7 days:

```json
{
  "retention_max_age": "168h"
}
```

### Combined limits

Keep at most 5,000 flows and remove anything older than 24 hours, with cleanup running every 30 minutes:

```json
{
  "retention_max_flows": 5000,
  "retention_max_age": "24h",
  "cleanup_interval": "30m"
}
```

### Frequent cleanup

For high-traffic environments, reduce the cleanup interval:

```json
{
  "retention_max_flows": 50000,
  "cleanup_interval": "5m"
}
```

### Disable automatic cleanup

Set `cleanup_interval` to `0` to disable the background cleaner. Note that this means flows will accumulate indefinitely even if `retention_max_flows` or `retention_max_age` is set:

```json
{
  "cleanup_interval": "0s"
}
```

## Validation rules

The following validation rules apply:

- `retention_max_flows` must be `>= 0`. A value of `0` means unlimited (no flow count limit).
- `retention_max_age` must be `>= 0`. A value of `0` means unlimited (no age limit).
- `cleanup_interval` must be `>= 0`. A value of `0` disables automatic cleanup. The default is `1h`.

## Related pages

- [CLI flags](cli-flags.md) -- All available CLI flags
- [Config file](config-file.md) -- Full config file reference
- [Flows](../concepts/flows.md) -- Understanding flows
