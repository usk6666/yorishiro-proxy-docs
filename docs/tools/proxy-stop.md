# proxy_stop

Stop proxy listener(s). Performs a graceful shutdown, waiting for existing connections to complete before stopping.

## Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `name` | string | No | | Listener name to stop. If omitted, all running listeners are stopped |

## Response

| Field | Type | Description |
|-------|------|-------------|
| `status` | string | Proxy state after the operation (`"stopped"`) |
| `stopped` | string[] | Names of listeners that were stopped |

## Examples

### Stop all listeners

```json
// proxy_stop
{}
```

### Stop a specific listener

```json
// proxy_stop
{
  "name": "http-listener"
}
```

## Notes

- The proxy must be running; otherwise an error is returned.
- Active connections are allowed to finish before the server shuts down.
- After stopping, you can call `proxy_start` again to restart.

## Related pages

- [proxy_start](proxy-start.md) -- Start the proxy
- [query](query.md) -- Check proxy status with `{"resource": "status"}`
